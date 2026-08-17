# LainOS DNS Mediation Architecture

## Overview

LainOS Layer 03 provides a localized, stateless DNS forwarding architecture built around `dnsmasq`. Rather than exposing upstream resolver information directly to applications, the operating system presents a single, stable resolver endpoint (`127.0.0.1`) and centralizes DNS policy within a dedicated forwarding layer.

This architecture minimizes locally retained DNS state, hides upstream resolver topology from ordinary applications, and allows the operating system to transition cleanly between normal networking, encrypted DNS, and Tor-routed privacy mode without requiring application awareness or configuration changes.

---

## Design Philosophy

The DNS infrastructure follows the same architectural principles used throughout LainOS Layer 03:

- Present a stable interface while hiding implementation details.
- Separate policy from mechanism.
- Minimize retained state wherever practical.
- Keep components simple, composable, and independently replaceable.
- Centralize network policy within the operating system rather than individual applications.

Applications should interact only with the local resolver. Decisions regarding upstream DNS infrastructure remain the responsibility of the operating system.

---

## Architecture

All DNS resolution follows a single deterministic path:

```
                +----------------------+
                |   Local Application  |
                +----------+-----------+
                           |
                           v
                    /etc/resolv.conf
                           |
                           v
                      127.0.0.1:53
                           |
                           v
                        dnsmasq
                           |
               +-----------+-----------+-----------+
               |                       |           |
               |                       |           |
         Plaintext Mode          Encrypted Mode  Private Mode
               |                       |           |
               v                       v           v
     DHCP / Manual Fallback     Local Proxy    Tor DNSPort
     (1.1.1.1, 9.9.9.9)        (127.0.0.1:5053)  (9059)
                                  |
                                  v
                               unbound
                           (127.0.0.1:5053)
                                  |
                                  v
                            dnscrypt-proxy
                           (127.0.0.1:5300)
                                  |
                    +-------------+-------------+
                    |                           |
                    v                           v
              Anonymized Relay              Resolver
              (IP hiding)              (DNSCrypt, no-log)
```

Applications never communicate directly with upstream DNS servers.

The resolver architecture remains identical regardless of operational mode; only the forwarding destination changes.

---

## Key Properties

### Stateless Forwarding

`dnsmasq` operates as a forwarding resolver rather than a caching resolver.

```
cache-size=0
no-negcache
```

This intentionally avoids retaining successful or negative DNS query history in memory. All caching, prefetching, and TTL management is delegated to `unbound`. This design ensures that `dnsmasq` remains a blind, stateless mediator: if compromised, it leaks no historical query data, and no stale records can be served from a local cache that bypasses upstream validation.

### Loopback Isolation

```
bind-interfaces
listen-address=127.0.0.1
```

The resolver is bound exclusively to the loopback interface and is never exposed externally.

### Resolver Mediation

Applications only observe:

```
127.0.0.1
```

They never directly observe:

- DHCP-provided resolvers
- fallback DNS servers
- Tor DNSPort
- encrypted DNS proxy endpoints
- resolver transitions
- upstream resolver topology
- anonymized relay infrastructure

---

## Operational Modes

### Plaintext Mode (Default)

During normal operation, `dnsmasq` forwards requests to the resolver information provided by DHCP.

Manual fallback servers are available for situations where DHCP has not yet completed or no resolver information is available.

Typical upstream hierarchy:

- DHCP-provided DNS
- Cloudflare (`1.1.1.1`)
- Quad9 (`9.9.9.9`)

Activated by: `lainos-dns plaintext`

---

### Encrypted Mode

Encrypted mode routes all DNS queries through a local chain that terminates in **Anonymized DNSCrypt**. Queries are encrypted on the wire, and the user's IP is hidden from the final resolver by an intermediate relay.

```
no-resolv
server=127.0.0.1#5053
```

The chain is:

1. `dnsmasq` forwards to `unbound` on `127.0.0.1:5053`
2. `unbound` validates DNSSEC, serves from cache, and forwards cache misses to `dnscrypt-proxy` on `127.0.0.1:5300`
3. `dnscrypt-proxy` encrypts the query via DNSCrypt and routes it through an **anonymized relay**
4. The relay forwards to the resolver. The relay knows the user's IP but not the query; the resolver knows the query but not the user's IP.

**Components:**

- **`unbound`** — Full recursive resolver with local DNSSEC validation, QNAME minimization, aggressive caching, and TTL management. It forwards to `dnscrypt-proxy` rather than performing its own encrypted transport.
- **`dnscrypt-proxy`** — DNSCrypt client with anonymized relay support. Handles all encrypted transport, relay selection, and load balancing across resolvers. Listens on `127.0.0.1:5300`.

The `lainos-dns` utility manages the full chain: it starts `dnscrypt-proxy`, waits for `:5300`, starts `unbound`, waits for `:5053`, then starts `dnsmasq` and verifies end-to-end resolution before declaring success.

Activated by: `lainos-dns encrypted`

Prerequisites:
```
pacman -S dnscrypt-proxy unbound
```

---

### Private Mode

Private Mode completely ignores DHCP resolver information and forces all DNS through Tor.

```
no-resolv
server=127.0.0.1#9059
```

All DNS resolution is forced through Tor's DNSPort.

Applications continue communicating only with the local resolver and require no configuration changes.

Activated by: `private-mode on`

---

## Mode Transitions & State Persistence

The `lainos-dns` utility manages transitions between plaintext and encrypted modes, persisting the active mode to `/var/lib/lainos/dns-mode`.

The `private-mode` utility manages transitions into and out of Tor-routed privacy mode. When entering private mode, the previous DNS mode (plaintext or encrypted) is saved to `/var/lib/lainos/dns-mode-previous`. When exiting private mode, the saved mode is restored automatically.

This ensures:

- A user who prefers encrypted DNS does not silently revert to plaintext after using private mode.
- Mode transitions are explicit, auditable, and reversible.
- No hidden magic — the user always knows which mode is active.

---

## Private-Mode Automation

The `private-mode` utility automates safe transitions between operating modes.

### private-mode on

The transition sequence is intentionally ordered to prevent accidental DNS leakage during mode changes.

1. Bring network interfaces down.
2. Verify network connectivity has stopped.
3. Save current DNS mode to `dns-mode-previous`.
4. Enable Snowflake.
5. Enable Tor-based `sdwdate`.
6. Replace the active `dnsmasq` configuration with private mode (Tor DNSPort).
7. Restart `dnsmasq`.
8. Restore network connectivity.

---

### private-mode off

The reverse transition follows the same philosophy.

1. Bring network interfaces down.
2. Disable Tor-specific privacy services.
3. Stop Tor.
4. Restore the saved DNS mode (plaintext or encrypted).
5. Restart `dnsmasq`.
6. Restore network connectivity.

---

### private-mode status

Reports:

- current DNS operating mode
- dnsmasq status
- active encrypted proxy chain and component status
- Snowflake status
- Tor status
- sdwdate status
- network interface state

---

## Why dnscrypt-proxy + Unbound

The encrypted DNS layer is split into two components with clearly separated responsibilities.

### Unbound: Validation and Cache

`unbound` operates as a local recursive resolver with the following responsibilities:

- **DNSSEC validation** — Responses are cryptographically verified before reaching `dnsmasq`, independent of upstream resolver trustworthiness.
- **QNAME minimization** — Upstream resolvers (via dnscrypt-proxy) receive only the labels necessary at each delegation step, reducing data exposure.
- **Aggressive caching with prefetching** — Unbound maintains a complete, optimized cache; popular domains are refreshed before TTL expiry, reducing upstream query volume and traffic analysis surface.
- **EDNS Client Subnet stripping** — Geographic metadata is removed before upstream transmission.
- **Query padding** — Frustrates size-based traffic analysis over the encrypted tunnel.

Unbound does **not** perform its own encrypted transport. Instead, it forwards cache misses to `dnscrypt-proxy` over plaintext localhost, leveraging `dnscrypt-proxy` for all wire encryption and relay management.

### dnscrypt-proxy: Encryption and Anonymization

`dnscrypt-proxy` operates as a dedicated encryptor with the following responsibilities:

- **DNSCrypt wire encryption** — All queries are encrypted before leaving the host. Local network observers cannot read DNS traffic.
- **Anonymized relay routing** — Queries are routed through an intermediate relay. The relay knows the user's IP but not the query; the resolver knows the query but not the user's IP. No single party holds both pieces of information.
- **Resolver load balancing** — Multiple no-log, DNSSEC-capable resolvers are maintained with automatic health checks and failover.
- **Ephemeral keys** — Per-session keying prevents long-term correlation.
- **No local cache** — Caching is disabled in `dnscrypt-proxy`; all caching is delegated to Unbound to avoid double-caching and to ensure the relay network sees only sparse, irregular traffic.

### Why This Split?

Separating validation/cache from encryption/transport provides architectural clarity and defense in depth:

- **Unbound** can be audited, updated, or replaced independently of the encryption layer.
- **dnscrypt-proxy** can be reconfigured with different relays or resolvers without touching validation policy.
- If `dnscrypt-proxy` is compromised, Unbound's cache and validation remain intact.
- If Unbound is compromised, dnscrypt-proxy's relay anonymization still hides the user's IP from resolvers.
- The relay network cannot build a query profile because Unbound's cache absorbs repeated lookups.

This is a stronger privacy model than direct DoT to a single provider (e.g., Cloudflare or Quad9), where one entity sees both the user's IP and the full query stream.

---

## Security & Privacy Properties

This architecture provides several security and privacy benefits.

- Applications interact only with a local resolver endpoint.
- Upstream DNS infrastructure remains abstracted behind the operating system.
- DNS resolver state is not retained in the forwarding layer; `dnsmasq` is zero-cache and blind.
- Resolver transitions occur under explicit operating system control.
- Switching between normal networking, encrypted DNS, and Tor routing requires no application reconfiguration.
- The DNS forwarding layer can be audited independently of higher-level networking components.
- Encrypted mode prevents local network observers from reading DNS queries.
- Anonymized relay mode prevents upstream resolvers from linking queries to the user's identity.
- Local DNSSEC validation prevents upstream resolvers from serving tampered or bogus records undetected.
- Query minimization and local cache reduce the relay network's visibility into user behavior.

---

## AppArmor Confinement

The entire DNS forwarding layer is confined under AppArmor:

- `/usr/bin/dnsmasq` — enforced; limited to loopback networking and necessary configuration paths.
- `/usr/bin/unbound` — enforced; restricted to resolver operations, cache directories, and localhost forwarding to dnscrypt-proxy.
- `/usr/bin/dnscrypt-proxy` — enforced; restricted to encrypted socket operations, relay/resolver lists, and cache directories. No access to Unbound's cache or validation keys.

These profiles complement the existing `protocol7-core` confinement layer and reduce the attack surface by enforcing strict sandboxing, limiting filesystem access, network capabilities, and system calls to only what is required for DNS forwarding operations.

---

## Threat Model

This architecture is intended to:

- Prevent ordinary applications from learning which upstream DNS infrastructure is currently in use.
- Minimize locally retained DNS resolver state in the forwarding layer.
- Provide a consistent resolver interface regardless of networking mode.
- Ensure DNS policy transitions occur in a controlled and deterministic order.
- Protect DNS queries from local network eavesdropping (encrypted mode).
- Hide the user's IP address from upstream DNS resolvers (anonymized relay mode).
- Anonymize DNS queries through Tor (private mode).
- Detect upstream DNS tampering via local DNSSEC validation.

It is **not** intended to:

- Prevent applications from implementing their own DNS-over-HTTPS (DoH) or DNS-over-TLS (DoT) clients.
- Prevent applications from embedding independent DNS resolvers.
- Anonymize applications that intentionally bypass the system resolver.
- Replace firewall or network-level enforcement mechanisms.
- Provide multi-hop anonymity comparable to Tor (the relay is a single hop; for multi-hop anonymity, use private mode).

Applications are expected to use the operating system's resolver interface (`127.0.0.1`).

---

## Summary

The LainOS DNS infrastructure is designed as a mediation layer rather than a traditional caching resolver.

Instead of exposing upstream resolver information directly to applications, the operating system presents a stable loopback interface while retaining full control over resolver policy internally. The forwarding layer (`dnsmasq`) is deliberately stateless, delegating all caching and validation to `unbound`, while all encrypted transport and relay anonymization is delegated to `dnscrypt-proxy`. This approach aligns with the broader LainOS architecture: small, focused components with clearly defined responsibilities, explicit policy transitions, minimal retained state, and defense in depth through AppArmor confinement. It also enables seamless integration with `lainos-dns` and `private-mode`, allowing the operating system to transition between conventional networking, encrypted anonymized DNS, and Tor-routed DNS without requiring application awareness or configuration changes.

---
