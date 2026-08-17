# lainOS layer 03 (Circuit VIII)

> A systemd-free Gentoo-based Linux distribution built with OpenRC as PID 1, delivering compile-time hardening and native OpenRC service isolation (modeled on Kicksecure/Whonix systemd directives). Built upon the lainOS layer 02 foundation.

## [lainOS layer 03 is developed at Forgejo](https://forgejo.lain.rocks/lainOS/lainOS-layer-03)

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Architecture](https://img.shields.io/badge/Arch-x86__64-blue)](https://gentoo.org)
[![Init](https://img.shields.io/badge/Init-OpenRC-green)](https://github.com/OpenRC/openrc)

---

## Table of Contents

- [Overview](#overview)
- [What's New in Layer 03](#whats-new-in-layer-03)
- [Architecture](#architecture)
- [Key Features](#key-features)
- [System Requirements](#system-requirements)
- [Building](#building)
- [Installation](#installation)
- [Session Types](#session-types)
- [System Hardening](#system-hardening)
- [Network Configuration](#network-configuration)
- [lainos-utils](#lainos-utils)
- [Troubleshooting](#troubleshooting)
- [Project Structure](#project-structure)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)

---

## Overview

lainOS Layer 03 is the next generation of the lainOS operating system. It preserves the hardened, privacy-first but daily drivable runtime of layer 02 while migrating the foundation from Arch Linux to Gentoo, with a new modular 4 layer OpenRC process isolation/containment stack.

The result is a system where security policy is defined **before compilation**, not applied after installation. Compiler hardening, dependency selection, USE flag policy, and service architecture are all properties of the build system. Every official release is a pre-built SquashFS image produced by a deterministic pipeline ~ your installed system is identical to the one tested by the maintainer.

Like layer 02, layer 03 is systemd-free, OpenRC-native, AppArmor-enforced, and built for bare-metal daily use. Unlike layer 02, it requires no compatibility layer to achieve this ~ OpenRC is the native init system on Gentoo.

---

## What's New in layer 03

### Compile-Time Security

On layer 02, hardening was applied to pre-built binaries. On Layer 03, hardening is **baked into the build itself**:

- **Position Independent Executables (PIE)** and **Full RELRO** on every binary
- **FORTIFY_SOURCE=3** and **stack protection** for buffer overflow detection
- **Control Flow Integrity (CFI)** to restrict indirect function calls
- **Link-Time Optimization (LTO)** to strip dead code paths

These are not post-install toggles. They are compiler defaults set in the Portage profile.

### Native OpenRC Architecture

Layer 02 required **Protocol 7** ~ a compatibility layer translating systemd assumptions into OpenRC. layer 03 eliminates this entirely:

- No `protocol7-core`
- No `lainos-init`, `lainos-dbus-bridge`, `lainos-notifyd`
- No `lainos-audio-init`
- No systemd ABI stubs, no elogind, no D-Bus facades

OpenRC is the native init system. Every service ships its own init script. The boot chain is simpler, faster, and fully legible.

### OpenRC Process Isolation (Rust)

Implementation Repo: [openrc-isolation implementation](https://forgejo.lain.rocks/lainOS/lainOS-layer-03/src/branch/main/OpenRC-Service-Isolation.md)

Modern service containment (`ProtectSystem=`, `PrivateTmp=`, `CapabilityBoundingSet=`, resource limits, and syscall filtering) is restored natively in OpenRC through four composed kernel primitives, driven by declarative `rc_*` variables in `/etc/conf.d/<service>` and invoked per-service by `rc-sandbox` and `lainos-sandbox-wrap`:

- **bwrap** ~ mount/PID/network namespace isolation and filesystem containment
- **cgroup-v2** ~ resource accounting and limits (`rc_memory_max`, `rc_cpu_quota`, `rc_pids_max`)
- **seccomp-bpf** ~ per-service syscall allowlisting (`rc_seccomp_profile`)
- **Landlock LSM** ~ path-scoped access enforcement that survives a namespace escape (`rc_landlock_ro`, `rc_landlock_rw`)

The chain is explicit: `rc-sandbox` (a compiled Rust binary) reads the service's `rc_*` variables, writes cgroup limits, and invokes `bwrap`; `bwrap` constructs the namespaces; then a statically-linked Rust binary (`lainos-sandbox-wrap`) running inside those namespaces applies Landlock rules, drops capabilities, and loads the seccomp-bpf filter before `execve()` into the target service.

Both `rc-sandbox` and `lainos-sandbox-wrap` are apparmor confined, written in Rust, and compiled to native code. No shell scripts or interpreted code exist in the isolation chain between OpenRC and the target service.

Declarative variables in `/etc/conf.d/<service>`:
- `rc_sandbox="YES"` ~ enable the full isolation stack (default). Set to `"NO"` to pass through to the target binary directly
- `rc_private_tmp="YES"` ~ private `/tmp` per service
- `rc_protect_home="YES"` ~ hidden `/home` and `/root`
- `rc_protect_system="STRICT"` ~ read-only `/usr` and `/boot`
- `rc_capability_bounding_set="..."` ~ minimal kernel capabilities, dropped by `lainos-sandbox-wrap` before its syscall filter loads
- `rc_seccomp_profile="..."` ~ named syscall profile (`lainos-base`, `lainos-network`, `lainos-privileged`)
- `rc_landlock_ro` / `rc_landlock_rw` ~ explicit path allowlists enforced at the LSM layer
- `rc_unshare_pid="YES"` ~ new PID namespace (default, requires foreground mode). Set to `"NO"` for services requiring host PID visibility

All lainOS-shipped runscripts invoke `rc-sandbox` by default. A service runs unsandboxed only if `rc_sandbox='NO'` is set in its `/etc/conf.d/<service>` file, or if its runscript does not call `rc-sandbox` at all (e.g. user-installed third-party services). `openrc-security-status` distinguishes between sandboxed, opted-out, and unsandboxed services.

Sandboxed services must be configured to run in the foreground (e.g. `dnsmasq --keep-in-foreground`, `tor --RunAsDaemon 0`). Daemonizing breaks host-visible PID tracking because `--unshare-pid` places the service in a new PID namespace. Services whose protocol or runtime model requires host PID visibility (e.g. `dhcpcd`, D-Bus policy) use `rc_unshare_pid="NO"` ~ this is a workload compatibility declaration, not a containment downgrade. They still receive mount namespace, seccomp, Landlock, cgroup limits, and capability drops.

No fork of OpenRC. No systemd code.

### Image-Based Deployment

 Layer 03 deploys a **pre-built SquashFS image**:

- No package resolution during install
- No configuration drift between systems

The SquashFS is the installation payload only. Once deployed to disk, the system is a fully writable Gentoo installation; users update normally via emerge, just like a traditional Gentoo system.

---

## Architecture

| Layer | Component | Role |
|-------|-----------|------|
| **Boot** | GRUB ~ Dracut | UEFI/BIOS boot, initramfs with `dmsquash-live` and LUKS support |
| **Init** | `/sbin/openrc-init` | PID 1, explicit kernel cmdline |
| **Live Session** | `greetd` + `tuigreet` | TUI login manager on TTY1 |
| **Wayland** | Sway + i3status-rs | Tiling compositor, autotiling, themed status bar |
| **Installer** | Calamares | Graphical system installer (autolaunches on liveuser login) |
| **Isolation** | `rc-sandbox` + `lainos-sandbox-wrap` + bwrap + cgroup-v2 + seccomp + Landlock | Composed namespace, resource, syscall, and path containment |

### Boot Chain

```
BIOS/UEFI ~ GRUB ~ kernel + initramfs
  ~ Dracut: dmsquash-live mounts squashfs, execs /sbin/openrc-init
     ~ OpenRC sysinit: dbus, machine-id
        ~ OpenRC boot: cgroup-delegate, syslog-ng
           ~ OpenRC default: seatd, greetd, chrony, nftables, acpid
              ~ greetd ~ tuigreet ~ Sway session
                 ~ Sway ~ lainos-utils ~ user session
```

`iwd` is intentionally **not** in this default startup chain ~ WiFi stays off until you deliberately turn it on.

---

## Key Features

### System
- **systemd-free** ~ OpenRC as PID 1 with no systemd binary present
- **Native OpenRC** ~ no compatibility layer, no Protocol 7, no elogind
- **Live ISO** ~ Fully bootable live environment with Calamares installer (autolaunches on login)
- **Dracut initramfs** ~ Modern initramfs with live boot and LUKS/crypt support
- **BTRFS by default** ~ with separate ext4 /boot for GRUB compatibility
- **Image-based install** ~ pre-built SquashFS deployed to disk, not assembled during install

### Compile-Time Hardening
- **PIE + Full RELRO** ~ all binaries position-independent with protected GOT
- **FORTIFY_SOURCE=3** ~ automated buffer overflow detection
- **Stack protection** ~ `-fstack-protector-strong` on all compiled code
- **CFI + LTO** ~ control-flow integrity and dead-code elimination
- **USE flag minimization** ~ unused features stripped at compile time per package

### Desktop (Sway/Wayland)
- **Sway** 1.12+ tiling compositor with custom keybindings
- **i3status-rs** themed status bar
- **wofi** application launcher
- **alacritty** terminal emulator
- **mako** notification daemon
- **swaybg** static wallpaper
- **Autotiling** automatic window tiling
- **Powerlevel10k** zsh prompt
- **CoplandOS-GTK** dark theme with StarLabs cursor
- **wlogout** session/power menu
- **swaylock** screen locker with wallpaper background
- **gnome-keyring** secrets service backend for Electron apps
- **PipeWire** audio, bundled by default

### Service Isolation (Rust)
- **Modular sandboxing** ~ bwrap (namespaces) + cgroup-v2 (resource limits) + seccomp-bpf (syscall filtering) + Landlock (path enforcement), stacked per service
- **Private tmpfs** per service (`rc_private_tmp`)
- **Read-only system binds** (`rc_protect_system`)
- **Capability bounding** ~ stripped to minimum required, dropped before the syscall filter loads (`rc_capability_bounding_set`)
- **Resource limits** ~ memory, CPU, and process-count ceilings via cgroup-v2 (`rc_memory_max`, `rc_cpu_quota`, `rc_pids_max`)
- **Syscall filtering** ~ per-service seccomp-bpf profiles (`rc_seccomp_profile`), broad-then-narrow starting point
- **Landlock backstop** ~ LSM-level path restriction that holds even if a process escapes its mount namespace
- **Complements AppArmor** ~ namespaces and Landlock restrict *what*/*where* at runtime; AppArmor restricts *where* via boot-loaded MAC policy
- **PID namespace control** ~ `rc_unshare_pid="YES"` (default, foreground required) or `"NO"` (host PID namespace for services requiring host PID visibility)

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                         OpenRC Service Layer                            │
│                                                                         │
│   /etc/init.d/<service>          /etc/conf.d/<service>                  │
│   ─────────────────────          ──────────────────────                        │
│   command=/usr/libexec/rc-sandbox   rc_private_tmp="YES"                │
│   command_args="<real binary>"      rc_protect_home="YES"               │
│   supervisor=supervise-daemon       rc_protect_system="STRICT"          │
│   name="<service> (sandboxed)"      rc_network_access="YES|NO"          │
│                                      rc_unshare_pid="YES|NO"            │
│                                      rc_capability_bounding_set="..."   │
│                                      rc_seccomp_profile="lainos-*"      │
│                                      rc_landlock_ro="path:path:..."     │
│                                      rc_landlock_rw="path:path:..."     │
│                                      rc_pids_max="N"                    │
│                                      rc_memory_max="N"                  │
│                                      rc_private_dev="YES|NO"            │
└──────────────────────────────┬─────────────────────────────────────────────────────────┘
                                │ exec
                                ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                    rc-sandbox  (Rust binary #1)                         │
│                    /usr/libexec/rc-sandbox                              │
│                                                                         │
│  Reads conf.d settings → builds a bwrap invocation:                     │
│    • --unshare-pid (conditional on rc_unshare_pid)                      │
│    • --unshare-net (conditional on rc_network_access != YES)            │
│    • --bind / /                                                         │
│    • --tmpfs /tmp --tmpfs /home --tmpfs /root  (rc_private_tmp/home)    │
│    • --ro-bind /usr /usr  --ro-bind /boot /boot                         │
│    • --dev /dev  (conditional on rc_private_dev != NO)                  │
│    • --cap-add ALL  (raw caps stay available for step 3 to narrow)      │
│    • --setenv RC_SECCOMP_PROFILE / RC_LANDLOCK_RO / RC_LANDLOCK_RW /    │
│                RC_CAPABILITY_BOUNDING_SET                               │
│    • --die-with-parent                                                  │
│    • cgroup-v2 setup: /sys/fs/cgroup/openrc.<service>                   │
│         memory.max, pids.max                                            │
└──────────────────────────────┬─────────────────────────────────────────────────────────┘
                                │ exec into bwrap
                                ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                          bwrap (bubblewrap)                             │
│                                                                         │
│  Creates the actual namespaces + mount layout described above,          │
│  then execs into the next binary inside the new sandbox.                │
└──────────────────────────────┬─────────────────────────────────────────────────────────┘
                                │ exec
                                ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│              lainos-sandbox-wrap  (Rust binary #2)                      │
│              /usr/libexec/lainos-sandbox-wrap                           │
│                                                                         │
│  Runs INSIDE the new namespace, before the real service starts:         │
│                                                                         │
│   1. LANDLOCK                                                           │
│      • fstat() each RO/RW path → file vs directory                      │
│      • RO: READ_FILE (+READ_DIR if dir)                                 │
│      • RW: READ_FILE+WRITE_FILE (+READ_DIR, MAKE_*, REMOVE_* if dir)    │
│      • RO paths: ENOENT = hard fail (real misconfiguration)             │
│      • RW paths: ENOENT = skip + warn (service creates its own file)    │
│                                                                         │
│   2. no_new_privs                                                       │
│      • prctl(PR_SET_NO_NEW_PRIVS, 1)                                    │
│                                                                         │
│   3. CAPABILITIES                                                       │
│      • read current Bounding set                                        │
│      • drop everything NOT in rc_capability_bounding_set                │
│      • narrow Effective + Permitted to the keep list                    │
│      • (Inheritable left untouched — narrowing it broke some            │
│         services' own internal capset() self-management)                │
│                                                                         │
│   4. SECCOMP-BPF                                                        │
│      • resolve profile file, expand @include: directives                │
│      • flat, sorted, deduplicated syscall allow-list                    │
│      • content-addressed cache: /var/cache/lainos/seccomp/*.bpf         │
│      • default action: KillProcess (Errno(EPERM) = debug-only toggle)   │
│                                                                         │
│   5. exec() the real service binary                                     │
└──────────────────────────────┬─────────────────────────────────────────────────────────┘
                                │ exec
                                ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                    Real service binary                                  │
│         (unbound / dnsmasq / tor / dhcpcd / chrony / etc.)              │
│                                                                         │
│  Runs fully confined by everything above, PLUS an independent,          │
│  pre-existing system-wide layer:                                        │
│                                                                         │
│   6. APPARMOR  (separate, orthogonal layer — not part of the            │
│      rc-sandbox/lainos-sandbox-wrap pipeline at all)                    │
│      /etc/apparmor.d/usr.bin.<service>                                  │
│      • enforced independently of Landlock/seccomp/capabilities          │
│      • acts as a real backstop when a Landlock/capability grant is      │
│        deliberately widened (e.g. /run, /dev granted broadly)           │
└────────────────────────────────────────────────────────────────────────────────────────┘

seccomp profile layering (lainos-base.list, lainos-network.list, etc.):

  lainos-network.list
  ├── @include:lainos-base   (must be explicit — no automatic inheritance)
  └── + recvmmsg, sendmmsg, getsockname, socket, connect, bind, ...

  lainos-base.list
  └── execve, mmap, mprotect, futex, openat, read, write, close,
      rt_sigaction, prctl, capget, capset, clone3, ... (grows per-service
      as real gaps are found via unsandboxed census + sandboxed testing)
```

### System Hardening
- **AppArmor mandatory access control** ~ per-daemon profiles for the full stack: DNS chain (dnsmasq, unbound, dnscrypt-proxy), networking (tor, iwd, dhcpcd), media (pipewire, wireplumber, mpv, vlc), crypto (gpg, keepassxc), browsers (librewolf, Tor Browser), and system utilities. Loaded at boot before services start.
- **hardened_malloc** (GrapheneOS, light variant) ~ systemwide toggle; preloaded for eligible binaries
- **ram-wipe** ~ RAM-extraction attack defense via dracut shutdown hook, plus continuous `init_on_alloc`/`init_on_free` poisoning
- **Kernel memory hardening** ~ `init_on_alloc=1`, `init_on_free=1`, `page_alloc.shuffle=1`
- **Full ASLR** ~ `randomize_va_space=2`
- **ptrace restricted** ~ `yama.ptrace_scope=1`
- **kexec disabled** ~ `kexec_load_disabled=1`
- **Unprivileged user namespaces disabled** ~ `unprivileged_userns_clone=0`
- **Core dumps disabled**
- **IPv6 disabled** by default (prevents VPN leaks)
- **SYN flood protection**, ICMP redirect rejection, reverse path filtering
- **Kernel pointer restriction** (`kptr_restrict=2`)
- **dmesg restricted** to root only
- **Magic SysRq disabled**
- **CPU RNG not unconditionally trusted** (`random.trust_cpu=off`)
- **Ephemeral machine-id** ~ regenerated on every boot
- **Boot clock randomization** ~ ±180 seconds before networking
- **WiFi off by default** ~ `iwd` does not start automatically; toggle with `wifi`/`wifi-autostart`
- **iwd MAC randomization** ~ new MAC every time iwd starts
- **Ethernet MAC randomization** ~ available via `eth0` toggle
- **Optional Tor-based time sync** ~ `sdwdate` (opt-in)
- **Optional Tor pluggable transports** ~ `snowflake`/`obfs4` toggles
- **Tor stream isolation** ~ default SocksPort isolates by destination; four dedicated circuits (`tor1`-`tor4`) for per-application isolation with automatic Electron app detection
- **private-mode** ~ one command for sensitive-work sequence: forces network down, enables snowflake/sdwdate, switches dnsmasq to Tor DNSPort; reverses safely restoring previous DNS mode
- **nftables firewall** ~ default-deny with established/related allowed
- **yescrypt** password hashing
- **doas** instead of sudo (minimal attack surface)
- **LUKS full disk encryption** ~ opt-in at install, confirmed working
- **Signed release artifacts** ~ detached signatures + SHA-256 checksums

### Networking
- **iwd** for WiFi ~ off by default
- **dhcpcd** + **openresolv** for wired DHCP
- **dnsmasq** as centralized DNS mediator ~ blind forwarding resolver, no caching, three modes:
  - **Plaintext** (default) ~ DHCP-provided resolver with fallbacks
  - **Encrypted** ~ `dnsmasq` → `unbound` (DNSSEC validation, QNAME minimization, caching) → `dnscrypt-proxy` (wire encryption, anonymized relay routing). No single component sees both your IP and your query.
  - **Private** ~ Tor DNSPort on `127.0.0.1:9059`
- **chrony** for NTP (default) or **sdwdate** (opt-in Tor-based alternative)
- **nftables** for firewall management
- No NetworkManager, no systemd-networkd, no systemd-resolved

---

## Network Configuration

### DNS Mediation Architecture

All DNS resolution in lainOS follows a single deterministic path through a centralized, stateless forwarding layer. Applications never communicate directly with upstream DNS servers. Instead, the operating system presents a single stable resolver endpoint (`127.0.0.1:53`) and centralizes all DNS policy within a dedicated forwarding layer.

This architecture minimizes locally retained DNS state, hides upstream resolver topology from ordinary applications, and allows the operating system to transition cleanly between normal networking, encrypted DNS, and Tor-routed privacy mode without requiring application awareness or configuration changes.

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

Applications interact only with the local resolver. Decisions regarding upstream DNS infrastructure remain the responsibility of the operating system.

---

#### Stateless Forwarding

`dnsmasq` operates as a forwarding resolver rather than a caching resolver. This intentionally avoids retaining successful or negative DNS query history in memory. All caching, prefetching, and TTL management is delegated to `unbound`. This ensures that `dnsmasq` remains a blind, stateless mediator: if compromised, it leaks no historical query data, and no stale records can be served from a local cache that bypasses upstream validation.

```
cache-size=0
no-negcache
```

The resolver is bound exclusively to the loopback interface and is never exposed externally.

---

#### Encrypted Mode: Split-Controller Architecture

Encrypted mode routes all DNS queries through a local chain that terminates in **Anonymized DNSCrypt**. Queries are encrypted on the wire, and the user's IP is hidden from the final resolver by an intermediate relay.

The chain is:

1. **`dnsmasq`** forwards to `unbound` on `127.0.0.1:5053`
2. **`unbound`** validates DNSSEC, serves from cache, and forwards cache misses to `dnscrypt-proxy` on `127.0.0.1:5300`
3. **`dnscrypt-proxy`** encrypts the query via DNSCrypt and routes it through an anonymized relay
4. The relay forwards to the resolver. The relay knows the user's IP but not the query; the resolver knows the query but not the user's IP.

**Unbound: Validation and Cache**
- DNSSEC validation ~ responses are cryptographically verified before reaching `dnsmasq`
- QNAME minimization ~ upstream resolvers receive only the labels necessary at each delegation step
- Aggressive caching with prefetching ~ popular domains are refreshed before TTL expiry
- EDNS Client Subnet stripping ~ geographic metadata is removed before upstream transmission
- Query padding ~ frustrates size-based traffic analysis over the encrypted tunnel

**dnscrypt-proxy: Encryption and Anonymization**
- DNSCrypt wire encryption ~ all queries are encrypted before leaving the host
- Anonymized relay routing ~ queries are routed through an intermediate relay
- Resolver load balancing ~ multiple no-log, DNSSEC-capable resolvers with automatic failover
- Ephemeral keys ~ per-session keying prevents long-term correlation
- No local cache ~ caching is delegated to Unbound to avoid double-caching

**Why This Split?**
Separating validation/cache from encryption/transport provides architectural clarity and defense in depth:
- Unbound can be audited, updated, or replaced independently of the encryption layer.
- dnscrypt-proxy can be reconfigured with different relays or resolvers without touching validation policy.
- If dnscrypt-proxy is compromised, Unbound's cache and validation remain intact.
- If Unbound is compromised, dnscrypt-proxy's relay anonymization still hides the user's IP from resolvers.
- The relay network cannot build a query profile because Unbound's cache absorbs repeated lookups.

**This is a stronger privacy model than direct DoT or DoH to a single provider**, where one entity sees both the user's IP and the full query stream.

---

#### Operational Modes

**Plaintext Mode (Default)**

During normal operation, `dnsmasq` forwards requests to the resolver information provided by DHCP. Manual fallback servers are available for situations where DHCP has not yet completed.

Typical upstream hierarchy:
- DHCP-provided DNS
- Cloudflare (`1.1.1.1`)
- Quad9 (`9.9.9.9`)

Activated by: `lainos-dns plaintext`

---

**Encrypted Mode**

All DNS queries are routed through the `unbound` → `dnscrypt-proxy` chain with anonymized relay routing. Queries are encrypted, the user's IP is hidden from the resolver, and DNSSEC is validated locally.

Activated by: `lainos-dns encrypted`

---

**Private Mode**

Private Mode ignores DHCP resolver information and forces all DNS through Tor's DNSPort (`127.0.0.1:9059`). All DNS resolution is forced through Tor, providing multi-hop anonymity for DNS queries.

Activated by: `private-mode on`

---

#### Mode Transitions & State Persistence

The `lainos-dns` utility manages transitions between plaintext and encrypted modes, persisting the active mode to `/var/lib/lainos/dns-mode`.

The `private-mode` utility manages transitions into and out of Tor-routed privacy mode. When entering private mode, the previous DNS mode (plaintext or encrypted) is saved to `/var/lib/lainos/dns-mode-previous`. When exiting private mode, the saved mode is restored automatically.

This ensures:
- A user who prefers encrypted DNS does not silently revert to plaintext after using private mode.
- Mode transitions are explicit, auditable, and reversible.
- No hidden magic ~ the user always knows which mode is active.

---

#### Security & Privacy Properties

- Applications interact only with a local resolver endpoint.
- Upstream DNS infrastructure remains abstracted behind the operating system.
- DNS resolver state is not retained in the forwarding layer; `dnsmasq` is zero-cache and blind.
- Resolver transitions occur under explicit operating system control.
- Switching between normal networking, encrypted DNS, and Tor routing requires no application reconfiguration.
- Encrypted mode prevents local network observers from reading DNS queries.
- Anonymized relay mode prevents upstream resolvers from linking queries to the user's identity.
- Local DNSSEC validation prevents upstream resolvers from serving tampered or bogus records.
- Query minimization and local cache reduce the relay network's visibility into user behavior.

---

#### AppArmor Confinement

The entire DNS forwarding layer is confined under AppArmor:
- `/usr/bin/dnsmasq` ~ enforced; limited to loopback networking and necessary configuration paths.
- `/usr/bin/unbound` ~ enforced; restricted to resolver operations, cache directories, and localhost forwarding to dnscrypt-proxy.
- `/usr/bin/dnscrypt-proxy` ~ enforced; restricted to encrypted socket operations, relay/resolver lists, and cache directories.

These profiles reduce the attack surface by enforcing strict sandboxing, limiting filesystem access, network capabilities, and system calls to only what is required for DNS forwarding operations.

---

#### Threat Model

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

### Commands

```bash
lainos-dns plaintext    # Plaintext fallbacks
lainos-dns encrypted    # Encrypted: unbound + dnscrypt-proxy
lainos-dns private      # Tor DNSPort, via private-mode
lainos-dns status       # Show current mode and full chain status
```

Mode transitions are explicit and stateful; `private-mode` remembers and restores your previous mode on exit.

---

## System Requirements

### Minimum
- 64-bit x86_64 processor
- 2 GB RAM
- 4 GB USB drive or free disk space
- UEFI or BIOS boot support

### Recommended
- 4+ GB RAM
- GPU with Mesa drivers (Intel/AMD recommended; software rendering fallback available)
- USB 3.0 for live boot

### Tested Hardware
- QEMU/KVM with Virtio GPU 
- ThinkPad T480 (Libreboot) ~ designed for compatibility with UEFI and BIOS boot systems, includes LUKS FDE with BTRFS as the default filesystem type

---

## Building

### Prerequisites

**Build Host:** Gentoo Linux with OpenRC, or any Linux distribution capable of chroot operations.

Required packages (on Gentoo):
```bash
emerge -uv app-portage/ebuild app-portage/gentoolkit \
  sys-fs/squashfs-tools sys-boot/grub sys-kernel/dracut
```

### Build Steps

1. Clone the repository:
```bash
git clone https://forgejo.lain.rocks/lainOS/lainos-iso-layer-03.git
cd lainos-iso-layer-03
```

2. Run the build script:
```bash
doas ./build-lainos.sh
```

The script will:
- Download and extract a Gentoo Stage 3 (OpenRC variant)
- Enter a chroot and sync the Portage tree
- Inject the `::lainos` overlay and profile
- Emerge `@world` and `app-lainos/lainos-desktop`
- Configure OpenRC services, AppArmor, and live environment settings
- Generate a dracut initramfs with `dmsquash-live`
- Compress the rootfs into a SquashFS image
- Assemble the bootable ISO with GRUB

3. Hash the ISO:
```bash
lainos-hash-iso
```

The resulting ISO will be at `/var/tmp/lainos-out/lainOS-layer-03-YYYY.MM.DD-x86_64.iso`.

### Overlay Repository

All custom packages are maintained in the `::lainos` overlay. The overlay contains:

| Package | Purpose |
|---------|---------|
| `lainos-utils` | Utility scripts (includes `lainos-dns`, `wifi`, `wscan`, etc.) |
| `lainos-ram-wipe` | RAM wipe on shutdown |
| `lainos-hardened-malloc` | GrapheneOS hardened_malloc wrapper |
| `lainos-keyring` | Project signing keys |
| `lainos-apparmor` | AppArmor profiles + OpenRC loader |
| `lainos-calamares-config` | Calamares installer configuration |
| `lainos-calamares-dracut` | Calamares + dracut integration |
| `lainos-kernel-backup` | Kernel backup utility |
| `lainos-desktop` | Desktop metapackage (all core packages) |
| `lainos-rescue` | Rescue/diagnostic tools metapackage |
| `openrc-isolation` | OpenRC service isolation stack (`rc-sandbox`, `lainos-sandbox-wrap`, seccomp profiles, `openrc-security-status`) |
| `sdwdate` | Tor-based secure time sync (Whonix port) |
| `bootclockrandomization` | Boot-time clock jitter (Whonix port) |
| `maybenot-tunnel` | Traffic analysis resistance |
| `kloak` | Keystroke anonymization |

### Deploying packages to the overlay

```bash
cd ~/lainos-overlay
# Edit ebuild, bump version, generate Manifest
ebuild app-lainos/<package>/<package>-<version>.ebuild manifest
git add .
git commit -m "bump <package> to <version>"
git push
```

---

## Installation

### Live Boot

1. Write the ISO to a USB drive:
```bash
doas dd if=~/lainos-out/lainOS-layer-03-*.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

2. Boot from USB and select **lainOS Layer 03** from the bootloader.

3. At the `tuigreet` login screen, login as `liveuser` (no password required). Calamares launches automatically.

### Installing to Disk

Calamares launches automatically on liveuser login.

Follow the installer wizard. To enable full disk encryption, select **Encrypt system** on the partition screen. LUKS unlock at boot is handled by dracut's crypt module.

**Note on filesystem:** BTRFS is the default. A separate 1GB ext4 `/boot` partition is created automatically for GRUB compatibility.

### Post-Install Notes

- WiFi: **off by default** ~ run `wifi on` (or `wifi-autostart enable` to restore auto-start at boot), then `wscan`
- Privilege escalation: `doas` (not sudo)
- Power menu: `wlogout`
- Lid close: automatically locks screen with swaylock and suspends
- Audio: PipeWire is bundled by default
- Time sync: `chrony` (plaintext NTP) is the default; `sdwdate` (Tor-based) is installed but not enabled ~ toggle with `lainos-sdwdate enable`
- Traffic shaping: `maybenot-tunnel` is available but not enabled by default ~ toggle with `maybenot-tunnel enable`
- A short quick-start guide (`lainos-quickstart-help`) opens automatically the first time a new user opens a terminal; the full guide (`lainos-help`) is always available

---

## Session Types

### Sway (Wayland)

lainOS Layer 03 is currently a Wayland-only distribution (X11 can be added if desired). The desktop is Sway with i3status-rs.

**Keybindings:**
| Key | Action |
|-----|--------|
| `Mod4+Return` | Open terminal (alacritty + tmux) |
| `Mod4+Space` | Open application launcher (wofi) |
| `Mod4+Shift+q` | Close focused window |
| `Mod4+1-9` | Switch to workspace |
| `Mod4+Shift+1-9` | Move window to workspace |
| `Mod4+h/j/k/l` | Focus left/down/up/right |
| `Mod4+Shift+h/j/k/l` | Move window left/down/up/right |
| `Mod4+w` | Open LibreWolf |

Full keybinding list in `lainos-help`.

---

## System Hardening

### Kernel Parameters

`GRUB_CMDLINE_LINUX`:
```
random.trust_cpu=off init_on_alloc=1 init_on_free=1 page_alloc.shuffle=1
```

### sysctl Configuration

`/etc/sysctl.d/99-lainos-hardening.conf`:
```ini
# Network
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.rp_filter = 1

# Kernel
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.sysrq = 0
kernel.yama.ptrace_scope = 1
kernel.kexec_load_disabled = 1
kernel.unprivileged_userns_clone = 0
kernel.randomize_va_space = 2

# Filesystem
fs.suid_dumpable = 0
kernel.core_pattern = |/bin/false
```

### Building software with sandboxed build requirements

Some software needs unprivileged user namespaces for its own build-time sandboxing. Temporarily enable it:
```bash
doas sysctl -w kernel.unprivileged_userns_clone=1
# build your software
doas sysctl -w kernel.unprivileged_userns_clone=0
```
Resets to the hardened default automatically on reboot.

### AppArmor Mandatory Access Control

The `lainos-apparmor` package provides an OpenRC init script that loads profiles at boot before the daemons start. Profiles cover the full stack:

- **DNS chain:** dnsmasq, unbound, dnscrypt-proxy
- **Networking:** tor, iwd, dhcpcd, chronyd
- **Media:** pipewire, wireplumber, mpv, vlc
- **Crypto/Secrets:** gpg, gpg-agent, keepassxc
- **Browsers:** librewolf, Tor Browser
- **System:** ssh, sshd, nft, syslog-ng

Separate AppArmor profiles are also provided for the sandboxing components themselves (`rc-sandbox`, `lainos-sandbox-wrap`, and bwrap) to protect the enforcement mechanism from being subverted by a compromised service. All AppArmor profiles are written manually; `aa-genprof` and `aa-logprof` do not work reliably inside user namespaces.

Path-based MAC complements namespace isolation: if an attacker escapes the private mount namespace, AppArmor still blocks access to real user data and sensitive kernel interfaces.

### Service Isolation

OpenRC runscripts use `rc-sandbox` to compose four kernel-level containment layers per service, each configured via declarative `rc_*` variables in `/etc/conf.d/<service>`:

| Layer | Mechanism | Key variables |
|-------|-----------|----------------|
| Namespace + filesystem isolation | bwrap | `rc_private_tmp`, `rc_protect_home`, `rc_protect_system` |
| Resource limits | cgroup-v2 | `rc_memory_max`, `rc_cpu_quota`, `rc_pids_max` |
| Syscall filtering | seccomp-bpf | `rc_seccomp_profile` |
| Path enforcement (LSM-level backstop) | Landlock | `rc_landlock_ro`, `rc_landlock_rw` |
| PID namespace control | bwrap | `rc_unshare_pid` |

`rc_capability_bounding_set` is preserved through bwrap and dropped inside `lainos-sandbox-wrap`, before its seccomp filter loads ~ dropping capabilities requires syscalls the filter would otherwise block. `no_new_privs` is always on and is not configurable, closing a setuid-based filter bypass.

Primary containment layers (bwrap, seccomp) fail closed: if either can't be applied, the service does not start. Backstop/hardening layers (cgroup limits, Landlock) fail open with a visible warning in `openrc-security-status`, since their absence narrows containment without removing it.

Sandboxed services must be configured to run in the foreground; daemonizing breaks host-visible PID tracking because `--unshare-pid` places the service in a new PID namespace.

Verified via direct `/proc/<pid>/mounts` inspection, `/sys/fs/cgroup/openrc/<service>/` limit files, `/proc/<pid>/status` (`Seccomp`, `NoNewPrivs`), Landlock out-of-scope access attempts, and adversarial testing.

### hardened_malloc

GrapheneOS hardened_malloc (light variant) is preloaded via `LD_PRELOAD` wrappers for:
- alacritty, gnome-keyring-daemon, keepassxc, kleopatra, mpv, tor

Incompatible applications: librewolf, torbrowser-launcher, signal, thunar. Launched through `tor1`-`tor4` automatically strips `LD_PRELOAD` for detected Electron apps.

### Full Disk Encryption

LUKS encryption is supported and confirmed working via Calamares (opt-in at install time). Unlock at boot is handled by dracut's `crypt` module.

---

## Network Configuration

### DNS

All applications resolve through `127.0.0.1:53` (dnsmasq). The operating system controls resolver policy; applications require no configuration changes.

**Modes:**
- **Plaintext** (default) ~ DHCP-provided resolver with fallbacks to 1.1.1.1/9.9.9.9
- **Encrypted** ~ `dnsmasq` → `unbound` (DNSSEC, caching) → `dnscrypt-proxy` (encrypted transport, anonymized relay). No single component sees both your IP and your query.
- **Private** ~ Tor DNSPort on `127.0.0.1:9059` (`private-mode on`)

Mode transitions are explicit and stateful; `private-mode` remembers and restores your previous mode on exit.

```bash
lainos-dns plaintext    # Plaintext fallbacks
lainos-dns encrypted    # Encrypted: unbound + dnscrypt-proxy
lainos-dns private      # Tor DNSPort, via private-mode
lainos-dns status       # Show current mode and full chain status
```

### WiFi (iwd)

**Off by default.** Turn on and connect using lainos-utils:
```bash
wifi on
wscan
```

Or manually:
```bash
doas rc-service iwd start
iwctl
[iwd]# station wlan0 scan
[iwd]# station wlan0 get-networks
[iwd]# station wlan0 connect SSID
```

MAC randomization happens automatically every time iwd starts.

### Wired (dhcpcd)

Automatic on boot via dhcpcd in the default runlevel. Toggle the interface and randomize its MAC with `eth0 {on|off|status}`.

---

## lainos-utils

A set of utility scripts included with lainOS Layer 03:

| Script | Command | Purpose |
|--------|---------|---------|
| wscan | `wscan` | iwd scan and connect with numbered menu |
| wifi | `wifi {on\|off\|status}` | Full WiFi radio on/off |
| wifi-autostart | `wifi-autostart {enable\|disable\|status}` | Auto-start iwd at boot |
| eth0 | `eth0 {on\|off\|status}` | Wired interface + MAC randomization |
| wg-vpn | `wg1`/`wg1d` ~ `wg4`/`wg4d` | WireGuard VPN tunnel up/down |
| torctl | `torctl` | Start/stop Tor routing |
| snowflake | `snowflake {enable\|disable\|status}` | Toggle Snowflake bridge |
| obfs4 | `obfs4 {enable\|disable\|status}` | Toggle obfs4 bridge |
| lainos-sdwdate | `lainos-sdwdate {enable\|disable\|status}` | Toggle chrony/sdwdate |
| tor-tunnel | `tor1`/`tor2`/`tor3`/`tor4` | Dedicated isolated Tor circuits |
| private-mode | `private-mode {on\|off\|status}` | Sensitive-work sequence |
| ram-wipe | `ram-wipe {enable\|disable\|status}` | Toggle shutdown RAM wipe |
| kloak | `kloak` | Keystroke anonymization |
| lainos-dns | `lainos-dns {plaintext\|encrypted\|private\|status}` | DNS mode toggle |
| maybenot-tunnel | `maybenot-tunnel {enable\|disable\|status}` | Traffic analysis resistance |
| lainos-hardened-malloc | `lainos-hardened-malloc {enable\|disable\|status}` | Toggle system-wide |
| lainos-help | `lainos-help` | Open user guide |
| lainos-privacy-help | `lainos-privacy-help` | Privacy guide for sensitive sessions |
| lainos-quickstart-help | `lainos-quickstart-help` | Quick-start checklist |

---

## Troubleshooting

### Build Logs

Build output is logged to `/var/tmp/lainos-build.log` on the build host.

### Daemon Logs

Services log to `/var/log/daemon.log`:
```bash
doas tail -f /var/log/daemon.log
```

### Verifying Security Posture

```bash
doas lainos-security-status              # read-only status dashboard for lainos-apparmor, kernel hardening, etc.
doas openrc-security-status              # Adversarial test suite for OpenRC process isolation verification
```

`openrc-security-status` reports three states per service: **sandboxed** (full stack active), **opted-out** (`rc_sandbox='NO'` set), or **unsandboxed** (runscript does not use `rc-sandbox`).

### WiFi Soft-Blocked After Boot

On Libreboot systems with `thinkpad_acpi`, wifi may be soft-blocked on boot. Handled automatically via `rfkill-unblock` in the `boot` runlevel. If manual intervention is needed:
```bash
doas rfkill unblock all
doas rc-service iwd restart
```

### Electron App Keyring Mismatch

If an Electron app reports a keyring backend mismatch, it was launched with different `XDG_CURRENT_DESKTOP` values. Always launch consistently, or reset the app's local profile.

---

## Project Structure

```
lainos-iso-layer-03-catalyst
.
├── .forgejo
│   └── workflows
│       └── validate-overlay.yml
├── LICENSE
├── README.md
├── VERSION
├── build
│   └── build.conf
├── catalyst
│   ├── make.conf
│   ├── stage1.spec
│   └── stage3.spec
├── docs
│   ├── architecture
│   │   ├── boot.md
│   │   ├── dns-mediation-architecture.md
│   │   ├── networking.md
│   │   ├── openrc-isolation.md
│   │   └── threat-model.md
│   ├── build.md
│   ├── release.md
│   └── testing.md
├── iso
│   ├── calamares
│   │   ├── modules
│   │   │   ├── bootloader.conf
│   │   │   ├── partition.conf
│   │   │   └── unpackfs.conf
│   │   └── settings.conf
│   ├── dracut
│   │   └── dracut.conf.d
│   │       └── 99-lainos.conf
│   └── grub
│       └── grub.cfg
├── overlay
│   ├── app-lainos
│   │   ├── bootclockrandomization
│   │   │   └── bootclockrandomization-9999.ebuild
│   │   ├── kloak
│   │   │   └── kloak-9999.ebuild
│   │   ├── lainos-apparmor
│   │   │   └── lainos-apparmor-9999.ebuild
│   │   ├── lainos-calamares-config
│   │   │   └── lainos-calamares-config-9999.ebuild
│   │   ├── lainos-calamares-dracut
│   │   │   └── lainos-calamares-dracut-9999.ebuild
│   │   ├── lainos-desktop
│   │   │   └── lainos-desktop-9999.ebuild
│   │   ├── lainos-hardened-malloc
│   │   │   └── lainos-hardened-malloc-9999.ebuild
│   │   ├── lainos-kernel-backup
│   │   │   └── lainos-kernel-backup-9999.ebuild
│   │   ├── lainos-keyring
│   │   │   └── lainos-keyring-9999.ebuild
│   │   ├── lainos-ram-wipe
│   │   │   └── lainos-ram-wipe-9999.ebuild
│   │   ├── lainos-rescue
│   │   │   └── lainos-rescue-9999.ebuild
│   │   ├── lainos-utils
│   │   │   └── lainos-utils-9999.ebuild
│   │   ├── maybenot-tunnel
│   │   │   └── maybenot-tunnel-9999.ebuild
│   │   ├── openrc-isolation
│   │   │   └── openrc-isolation-9999.ebuild
│   │   └── sdwdate
│   │       └── sdwdate-9999.ebuild
│   ├── metadata
│   │   └── layout.conf
│   └── profiles
│       └── lainos
│           ├── make.defaults
│           ├── package.mask
│           ├── package.use
│           ├── package.use.force
│           ├── packages
│           ├── parent
│           └── use.mask
├── rootfs-overlay
│   ├── etc
│   │   ├── acpi
│   │   │   ├── events
│   │   │   └── handlers
│   │   ├── conf.d
│   │   │   ├── dnsmasq
│   │   │   └── tor
│   │   ├── default
│   │   │   └── grub
│   │   ├── dhcpcd.conf
│   │   ├── dnscrypt-proxy
│   │   │   └── dnscrypt-proxy.toml
│   │   ├── dnsmasq.conf
│   │   ├── dnsmasq.conf.encrypted
│   │   ├── dnsmasq.conf.plaintext
│   │   ├── dnsmasq.conf.private
│   │   ├── doas
│   │   │   └── doas.conf
│   │   ├── doas.conf
│   │   ├── dracut.conf.d
│   │   ├── fonts
│   │   │   └── conf.d
│   │   ├── greetd
│   │   │   ├── config.toml
│   │   │   └── config.toml.lainos-default
│   │   ├── group
│   │   ├── gtk-3.0
│   │   │   └── settings.ini
│   │   ├── hostname
│   │   ├── init.d
│   │   │   ├── dbus
│   │   │   ├── dnscrypt-proxy
│   │   │   ├── dnsmasq
│   │   │   ├── lainos-dns-setup
│   │   │   ├── lainos-live-keyring
│   │   │   ├── lainos-machine-id
│   │   │   ├── lainos-wifi-dns
│   │   │   ├── polkit
│   │   │   ├── resolvconf
│   │   │   ├── rfkill-unblock
│   │   │   └── unbound
│   │   ├── iwd
│   │   │   └── main.conf
│   │   ├── locale.conf
│   │   ├── localtime -> /usr/share/zoneinfo/UTC
│   │   ├── machine-id
│   │   ├── modprobe.d
│   │   │   └── broadcom-wl.conf
│   │   ├── motd
│   │   ├── os-release
│   │   ├── pam.d
│   │   │   ├── greetd
│   │   │   ├── login
│   │   │   └── swaylock
│   │   ├── passwd
│   │   ├── profile.d
│   │   │   └── lainos-runtime-dir.sh
│   │   ├── resolv.conf -> /run/resolvconf/resolv.conf
│   │   ├── resolv.dnsmasq
│   │   ├── resolvconf.conf
│   │   ├── runlevels
│   │   │   ├── boot
│   │   │   ├── default
│   │   │   └── sysinit
│   │   ├── shadow
│   │   ├── skel
│   │   │   ├── .Xresources
│   │   │   ├── .aliases
│   │   │   ├── .bash_logout
│   │   │   ├── .bash_profile
│   │   │   ├── .bashrc
│   │   │   ├── .config
│   │   │   ├── .gnupg
│   │   │   ├── .gtkrc-2.0
│   │   │   ├── .local
│   │   │   ├── .p10k.zsh
│   │   │   ├── .p10k.zsh.bak-20260712084033
│   │   │   ├── .p10k.zsh.save
│   │   │   ├── .zshrc
│   │   │   ├── Snowflake-Bridges-for-Tor-Browser.txt
│   │   │   ├── apparmor-test.sh
│   │   │   ├── lainos-wallpapers.sh
│   │   │   ├── libreboot_thinkpad_fan.sh
│   │   │   ├── no-sleep-till-brooklyn.sh
│   │   │   └── rfkill-unblock
│   │   ├── ssh
│   │   │   └── sshd_config.d
│   │   ├── stubby
│   │   │   └── stubby.yml
│   │   ├── sysctl.d
│   │   │   └── 99-lainos-hardening.conf
│   │   ├── syslog-ng
│   │   │   └── syslog-ng.conf
│   │   ├── tor
│   │   │   └── torrc
│   │   ├── unbound
│   │   │   └── unbound.conf
│   │   └── xdg
│   │       └── openbox
│   └── usr
│       ├── bin
│       │   └── dmsquash-live-root
│       └── share
│           ├── LainOS
│           ├── applications
│           ├── dbus-1
│           ├── fonts
│           ├── gitstatus
│           ├── glib-2.0
│           ├── grub
│           ├── icons
│           ├── lainos
│           ├── plymouth
│           ├── themes
│           └── wayland-sessions
├── scripts
│   ├── 00-build-stage3.sh
│   ├── 10-prepare-chroot.sh
│   ├── 20-build-rootfs.sh
│   ├── 30-build-squashfs.sh
│   ├── 40-build-iso.sh
│   ├── 50-hash-iso.sh
│   ├── 60-verify-build.sh
│   └── build-lainos.sh
└── tests
    ├── dns
    │   └── README.md
    ├── installer
    │   └── README.md
    ├── isolation
    │   └── README.md
    ├── live
    │   └── README.md
    ├── packages
    │   └── README.md
    └── profile
        └── README.md

94 directories, 122 files
```

---

## Contributing

lainOS is developed by Grayson Giles and the lainOS community.

- **Website:** https://lainos.net
- **Forgejo:** https://forgejo.lain.rocks/lainOS/lainos-iso-layer-03

### Reporting Issues

Please include:
- ISO version/date
- Hardware/VM configuration
- `rc-status` output
- Relevant logs from `/var/log/daemon.log` or `dmesg`

---

## License

lainOS Layer 03 is released under the **GNU General Public License v3.0**.

Individual components (Sway, OpenRC, Calamares, etc.) retain their respective licenses.

---

## Acknowledgments

- **Gentoo Linux** ~ The foundation everything is built on
- **OpenRC** ~ Reliable, predictable init system
- **GrapheneOS** ~ hardened_malloc
- **Sway/wlroots** ~ Modern Wayland compositor ecosystem
- **Calamares** ~ User-friendly system installer
- **Kicksecure/Whonix** ~ sdwdate, bootclockrandomization, ram-wipe, and security architecture inspiration


---

*Last updated: 2026-08-12*
*Status: In Development*
