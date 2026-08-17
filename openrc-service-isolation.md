# OpenRC Service Isolation Stack ~ Modular Sandboxing for lainOS Layer 03

> Systemd-equivalent service isolation on native OpenRC, without forking OpenRC. Inspired by Kicksecure/Whonix.

**Status:** Beta Testing

**Target:** lainOS Layer 03 (Gentoo base)

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Design Principle](#design-principle)
- [Architecture](#architecture)
- [Component 1: Namespace Isolation (bwrap)](#component-1-namespace-isolation-bwrap)
- [Component 2: cgroup-v2 Resource Delegation](#component-2-cgroup-v2-resource-delegation)
- [Component 3: seccomp-bpf Syscall Filtering](#component-3-seccomp-bpf-syscall-filtering)
- [Component 4: Landlock Path Scoping](#component-4-landlock-path-scoping)
- [Variable Passing: rc-sandbox -> bwrap -> lainos-sandbox-wrap](#variable-passing-rc-sandbox--bwrap--lainos-sandbox-wrap)
- [lainos-sandbox-wrap: The In-Namespace Stage](#lainos-sandbox-wrap-the-in-namespace-stage)
- [rc-sandbox: The Runscript Wrapper](#rc-sandbox-the-runscript-wrapper)
- [Declarative Variable Reference](#declarative-variable-reference)
- [Execution Order](#execution-order)
- [Comparison to systemd](#comparison-to-systemd)
- [Why Not Fork OpenRC](#why-not-fork-openrc)
- [Build System: Direct Cargo Integration](#build-system-direct-cargo-integration)
- [AppArmor Integration](#apparmor-integration)
- [Implementation Summary](#implementation-summary)
- [Testing / Verification](#testing--verification)
- [Open Questions](#open-questions)

---

## Problem Statement

Systemd provides optional per-unit sandboxing through three cooperating subsystems: mount/PID/network namespaces (`ProtectSystem=`, `PrivateTmp=`, `PrivateNetwork=`), cgroup-v2 resource accounting (`MemoryMax=`, `CPUQuota=`), and seccomp-bpf syscall filtering (`SystemCallFilter=`). OpenRC has none of this built in ~ it starts and supervises processes, but does not itself construct namespaces, delegate cgroups, or load syscall filters per service.

| Pillar | systemd mechanism | OpenRC native support |
|--------|-------------------|----------------------|
| Namespace isolation | mount/PID/net namespaces | None |
| Filesystem isolation | mount namespace + bind mounts | None |
| Resource accounting/limits | cgroup-v2 (`MemoryMax=`, `CPUQuota=`) | Partial |
| Syscall/capability filtering | seccomp-bpf (`SystemCallFilter=`) | None |

This document specifies four components that close this gap without changing OpenRC itself. All components are now implemented and integrated into the `::lainos` overlay.

```
rc-sandbox            -> cgroup-v2 setup + bwrap invocation      (Rust binary)
bwrap                 -> namespace + filesystem isolation        (existing tool)
lainos-sandbox-wrap   -> Landlock + capabilities + seccomp       (Rust binary)
```

---

## Design Principle

No single tool replaces systemd's sandboxing. Four primitives are composed together:

```
bwrap          -> mount/PID/net namespace isolation, filesystem containment
cgroup-v2      -> resource accounting + limits
seccomp-bpf    -> syscall allowlisting with caching
Landlock LSM   -> path-scoped access, survives ns escape
```

Each layer is invoked from a new OpenRC runscript wrapper (`rc-sandbox`) before `exec`, driven by declarative `rc_*` variables read from `/etc/conf.d/<service>`.

---

## Architecture

```
OpenRC runscript
  -> rc-sandbox (Rust binary): cgroup-v2 setup (join delegated cgroup, write limits)
     -> bwrap: construct namespaces, mount views
        -> lainos-sandbox-wrap (Rust binary, inside the new namespace):
             1. Landlock: apply path ruleset against sandboxed view
             2. no_new_privs: set PR_SET_NO_NEW_PRIVS
             3. Capability drop
             4. seccomp-bpf: load syscall filter (last, since filter
                also restricts what this wrapper itself can do)
           -> execve() into target binary
```

Both `rc-sandbox` and `lainos-sandbox-wrap` are written in Rust and compiled to statically-linked binaries. No shell scripts or interpreted code exist in the isolation chain between OpenRC and the target service.

---

## Component 1: Namespace + Filesystem Isolation (rc-sandbox + bwrap)

**Status:** Implemented.

### What this component provides

**Namespace isolation:** each service runs inside its own mount namespace, and optionally PID and network namespaces, constructed by `bwrap` (bubblewrap).

**Filesystem isolation:** provided by the combination of bwrap's mount namespace and Landlock:
- bwrap constructs the sandboxed filesystem view with restricted bind mounts
- Landlock provides a second, independent enforcement layer at the LSM level

### rc-sandbox: what it is and what it does

`rc-sandbox` is a Rust binary invoked from each service's OpenRC runscript in place of a direct `start-stop-daemon` call. It has three jobs, performed in order:

0. **Check `rc_sandbox`** ~ if `rc_sandbox="NO"` is set, `rc-sandbox` execs the target binary directly without any sandboxing.

1. **cgroup-v2 setup** ~ join/configure the service's resource cgroup and write limits.

2. **Construct and invoke the `bwrap` command line** ~ translate `rc_*` variables from `/etc/conf.d/<service>` into bwrap flags, then `exec` into `bwrap ... -- lainos-sandbox-wrap <target binary> [args]`.

### Default behavior and opt-out

All lainOS-shipped runscripts point their `command=` line at `/usr/libexec/rc-sandbox` by default. A service is unsandboxed only if `rc_sandbox='NO'` is set in its `/etc/conf.d/<service>` file, or if its runscript does not call `rc-sandbox` at all (e.g., user-installed third-party services). `openrc-security-status` distinguishes between sandboxed, opted-out, and unsandboxed services.

### Fallback behavior when a layer is unavailable

| Layer unavailable | Behavior | Rationale |
|-------------------|----------|-----------|
| bwrap missing/fails | **Service fails to start.** | Primary containment layer |
| cgroup-v2 unsupported | **Starts without limits; `openrc-security-status` reports warning.** | Hardening layer |
| Landlock unavailable | **Starts; `openrc-security-status` reports warning.** | Backstop layer |
| seccomp-bpf load fails | **Service fails to start.** | Primary containment layer |
| no_new_privs fails | **Service fails to start.** | Closes setuid-based seccomp bypass |

---

## Component 2: cgroup-v2 Resource Delegation

**Status:** Implemented.

OpenRC creates `/sys/fs/cgroup/openrc/<service>/` when `rc_cgroup_mode="unified"` is set. `rc-sandbox` writes resource limits from `rc_*` variables into that cgroup's control files:

```bash
cgroup_path="/sys/fs/cgroup/openrc/${RC_SVCNAME}"
[ -n "${rc_memory_max}" ] && echo "${rc_memory_max}" > "${cgroup_path}/memory.max"
[ -n "${rc_cpu_quota}"  ] && echo "${rc_cpu_quota}"  > "${cgroup_path}/cpu.max"
[ -n "${rc_pids_max}"   ] && echo "${rc_pids_max}"   > "${cgroup_path}/pids.max"
```

---

## Component 3: seccomp-bpf Syscall Filtering

**Status:** Implemented.

`lainos-sandbox-wrap` uses `libseccomp-rs` to parse profile files, compile them to BPF, and cache the results to `/var/cache/lainos/seccomp/<profile>.bpf`.

### Profile authoring

Profiles are flat `.list` files, one syscall name per line. Three profiles are provided:

**lainos-base.list** ~ Baseline syscalls required by almost all services. This is a floor, not a ceiling ~ intentionally permissive to keep services from breaking. Tighten further per-service as needed.

```
read
write
openat
close
mmap
mprotect
munmap
brk
futex
exit
exit_group
nanosleep
clock_gettime
rt_sigaction
rt_sigreturn
set_tid_address
```

**lainos-network.list** ~ Syscalls required for network I/O and DNS resolution. Includes socket operations, connect, bind, accept, and send/recv.

```
socket
connect
bind
listen
accept
sendto
recvfrom
sendmsg
recvmsg
getsockopt
setsockopt
```

**lainos-privileged.list** ~ For services that need raw capabilities and extended networking. This profile must be audited individually per service. Services like `iwd`, `dhcpcd`, and `polkit` require this profile. The profile is a complete list of syscalls for privileged services.

```
brk
execve
exit_group
getuid
mmap
mprotect
set_tid_address
write
openat
read
close
fstat
fcntl
prctl
capget
capset
getrandom
munmap
madvise
rt_sigaction
rt_sigprocmask
sigaltstack
ppoll
getpid
getgid
geteuid
getegid
getppid
getpgrp
setsid
setpgid
getgroups
setgroups
setuid
setgid
setreuid
setregid
setresuid
setresgid
socket
connect
bind
listen
accept
sendto
recvfrom
sendmsg
recvmsg
getsockopt
setsockopt
```

### Cache validation

The wrapper validates the cache using file modification times:

1. Source profile: `/etc/lainos/seccomp/<profile>.list`
2. Cache file: `/var/cache/lainos/seccomp/<profile>.bpf`
3. If the cache file's mtime is newer than or equal to the source file's mtime, the cache is valid and loaded directly
4. If the source file is newer, or the cache file does not exist, the wrapper recompiles and writes a new cache file
5. The wrapper creates the cache directory if it does not exist

This ensures that profile changes are applied immediately without manual intervention, while avoiding recompilation on every service start.

**Cache invalidation with includes (future consideration):** If profile includes (e.g., `@lainos-base` shared across services) are introduced in a future version, mtime-based invalidation would need to change to hash-based invalidation. A single-file mtime check would become stale because editing `base.list` would not invalidate the cached `.bpf` for every service that includes it. v1 uses flat, non-including profile files to avoid this problem entirely.

### Implementation

```rust
// Load seccomp filter from profile
let mut ctx = ScmpFilterContext::new_filter(ScmpAction::KillProcess)?;
for name in syscalls {
    let sysno = ScmpSyscall::from_name(&name)?;
    ctx.add_rule(ScmpAction::Allow, sysno)?;
}
let bpf = ctx.export_bpf(&mut cache_file)?;
// Load via prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, ...)
```

---

## Component 4: Landlock Path Scoping

**Status:** Implemented.

Landlock provides LSM-level path enforcement that survives a namespace escape. It is applied inside `lainos-sandbox-wrap` using direct syscall invocations.

### Implementation

```rust
// syscall numbers (same for x86_64 and aarch64)
const SYS_LANDLOCK_CREATE_RULESET: i64 = 444;
const SYS_LANDLOCK_ADD_RULE: i64 = 445;
const SYS_LANDLOCK_RESTRICT_SELF: i64 = 446;

// Create ruleset
let ruleset_fd = landlock_create_ruleset(&attr, size, 0);
// Add read-only paths
for path in ro_paths {
    landlock_add_rule(ruleset_fd, LANDLOCK_RULE_PATH_BENEATH, &rule, 0);
}
// Apply
landlock_restrict_self(ruleset_fd, 0);
```

### Fallback

If Landlock is unavailable (`ENOSYS`), the wrapper continues without it and `openrc-security-status` reports a warning.

---

## Variable Passing: rc-sandbox -> bwrap -> lainos-sandbox-wrap

`rc-sandbox` passes environment variables to `lainos-sandbox-wrap` via bwrap's `--setenv`:

```bash
bwrap [namespace flags] --cap-add ALL \
  --setenv RC_SECCOMP_PROFILE "${rc_seccomp_profile}" \
  --setenv RC_LANDLOCK_RO "${rc_landlock_ro}" \
  --setenv RC_LANDLOCK_RW "${rc_landlock_rw}" \
  --setenv RC_CAPABILITY_BOUNDING_SET "${rc_capability_bounding_set}" \
  -- /usr/libexec/lainos-sandbox-wrap /usr/bin/dnsmasq [args...]
```

`lainos-sandbox-wrap` reads these environment variables and applies the corresponding restrictions.

---

## lainos-sandbox-wrap: The In-Namespace Stage

**Status:** Implemented in Rust.

A statically-linked Rust binary executed by `bwrap` as the sandboxed process's entrypoint. Responsibilities, in order:

1. Apply Landlock ruleset (fail open if unavailable)
2. Set `PR_SET_NO_NEW_PRIVS` (hard failure)
3. Drop capabilities per `rc_capability_bounding_set` (hard failure)
4. Load (cached, if available) seccomp-bpf filter (hard failure)
5. `execve()` into the target binary

### Rust rationale

Rust was chosen for this binary because:

- **Memory safety** ~ Eliminates buffer overflows, use-after-free, and other memory corruption vulnerabilities
- **Shifted security analysis** ~ The compiler catches memory bugs before production
- **Strong static analysis** ~ `clippy` enforces best practices
- **Dependency auditing** ~ `cargo-audit` scans for known CVEs
- **Performance** ~ Rust compiles to native code with zero runtime overhead

---

## rc-sandbox: The Runscript Wrapper

**Status:** Implemented in Rust.

`rc-sandbox` is a Rust binary that handles:

1. `rc_sandbox="NO"` opt-out check (passthrough)
2. cgroup-v2 limit writing
3. bwrap command line construction
4. `exec` into bwrap (not fork)

It must `exec` into bwrap directly after cgroup setup, with no fork and no work after, so that signals from OpenRC's supervisor (`TERM` on `rc-service <n> stop`) land on bwrap directly.

---

## Declarative Variable Reference

All `rc_*` variables introduced by this design:

| Variable | Effect | Backing mechanism |
|----------|--------|-------------------|
| `rc_sandbox` | `"YES"`: full isolation (default). `"NO"`: passthrough | N/A |
| `rc_private_tmp` | Private `/tmp` | bwrap `--tmpfs /tmp` |
| `rc_protect_home` | Hidden `/home` and `/root` | bwrap `--tmpfs` over each |
| `rc_protect_system` | Read-only `/usr` and `/boot` | bwrap `--ro-bind` |
| `rc_network_access` | Host network namespace instead of isolated | bwrap: omit `--unshare-net` |
| `rc_unshare_pid` | `"YES"`: new PID namespace (default). `"NO"`: host PID | bwrap `--unshare-pid` |
| `rc_capability_bounding_set` | Capabilities to keep after drop | dropped by `lainos-sandbox-wrap` |
| `rc_memory_max` | Hard memory ceiling | cgroup-v2 `memory.max` |
| `rc_cpu_quota` | CPU quota | cgroup-v2 `cpu.max` |
| `rc_pids_max` | Max process count | cgroup-v2 `pids.max` |
| `rc_seccomp_profile` | Named syscall profile (`lainos-base`, `lainos-network`, `lainos-privileged`) | seccomp-bpf |
| `rc_landlock_ro` | Colon-separated read-only paths | Landlock |
| `rc_landlock_rw` | Colon-separated read-write paths | Landlock |

`no_new_privs` is intentionally **not** an `rc_*` variable ~ it is always-on in `lainos-sandbox-wrap`.

---

## Execution Order

Order is security-critical:

1. **cgroup-v2** (in `rc-sandbox`, pre-bwrap) ~ join/configure cgroup first
2. **bwrap** ~ construct namespaces, mount views
3. **lainos-sandbox-wrap**, inside the namespace:
   a. **Landlock** ~ applied against the sandboxed path view
   b. **no_new_privs** ~ set before capability drop and seccomp
   c. **Capability drop** ~ before seccomp load
   d. **seccomp-bpf** ~ loaded last

---

## Comparison to systemd

| Guarantee | systemd | This design |
|-----------|---------|-------------|
| Namespace isolation | Yes | Yes (bwrap) |
| cgroup resource accounting | Yes, native | Yes, via OpenRC cgroup support |
| Syscall filtering | Yes, per-unit, mature | Yes, custom profiles with caching |
| Path restriction surviving ns escape | Namespaces only | **Stronger** ~ Landlock is LSM-level |
| Dependency/transaction-aware unit graph | Yes | Not attempted ~ OpenRC's model unchanged |
| Live introspection | Yes (`systemctl status`) | `openrc-security-status` extension |

Landlock provides one guarantee that is stronger than stock systemd sandboxing (LSM enforcement independent of namespace integrity).

---

## Why Not Fork OpenRC

- The gap is **exposed kernel surface**, not OpenRC's service/dependency model
- Forking creates permanent maintenance burden and divergence from upstream
- All four components are driven from userspace wrapper code invoked from unmodified OpenRC runscripts

---

## Build System: Direct Cargo Integration

The OpenRC isolation stack uses a direct build approach integrated into the Gentoo ebuild:

- **`rc-sandbox`**: Rust binary, built with Cargo alongside `lainos-sandbox-wrap` in the same workspace, installed to `libexecdir`.
- **`lainos-sandbox-wrap`**: Rust binary, built with Cargo, statically linked with musl.
- **Seccomp profiles**: Plain text files, installed directly by the ebuild to `/etc/lainos/seccomp/`.
- **`openrc-security-status`**: Shell script (separate package).

### Rust Toolchain and Static Linking

`rc-sandbox` and `lainos-sandbox-wrap` are written in Rust and compiled to statically-linked binaries. Static linking is required because the binaries run inside the sandbox before shared libraries are available.

### Gentoo Ebuild Integration

```bash
src_compile() {
    export RUSTFLAGS="-C target-feature=+crt-static"
    cd "${S}" || die
    cargo build --release --target x86_64-unknown-linux-musl || die
}

src_install() {
    exeinto /usr/libexec
    doexe "${S}/target/x86_64-unknown-linux-musl/release/rc-sandbox"
    doexe "${S}/target/x86_64-unknown-linux-musl/release/lainos-sandbox-wrap"

    insinto /etc/lainos/seccomp
    doins "${S}/profiles/lainos-base.list"
    doins "${S}/profiles/lainos-network.list"
    doins "${S}/profiles/lainos-privileged.list"
}
```

### Why Direct Cargo Instead of Meson

- **Simplicity**: The project has only two Rust binaries and a handful of shell scripts and data files.
- **Gentoo alignment**: The ebuild already controls the build environment. Using Cargo directly fits the Gentoo way.
- **Reduced dependencies**: No need for Meson or Ninja in the build dependencies.

---

## AppArmor Integration

AppArmor provides an independent, path-based MAC layer that complements the namespace-based isolation. It is loaded at boot and enforced alongside the four-layer stack.

### Profiles for Services

AppArmor profiles exist for the full service stack:
- DNS chain: dnsmasq, unbound, dnscrypt-proxy
- Networking: tor, iwd, dhcpcd, chronyd
- Media: pipewire, wireplumber, mpv, vlc
- Crypto/Secrets: gpg, gpg-agent, keepassxc
- Browsers: librewolf, Tor Browser
- System: ssh, sshd, nft, syslog-ng

**All AppArmor profiles for services are written manually.** This was proven during Layer 02 development, where all AppArmor profiles had to be authored manually.

### Profiles for Sandboxing Components

Separate AppArmor profiles exist for the sandboxing components themselves:

- **`rc-sandbox`**: The compiled binary that constructs bwrap commands.
- **`lainos-sandbox-wrap`**: The Rust binary that runs inside the namespace.
- **bwrap**: The setuid-root binary that constructs namespaces.

---

## Implementation Summary

### Repository Structure

```
openrc-isolation/
├── Cargo.toml              # Workspace root
├── rc-sandbox/
│   ├── Cargo.toml
│   └── src/main.rs
├── lainos-sandbox-wrap/
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs
│       └── landlock.rs
└── profiles/
    ├── lainos-base.list
    ├── lainos-network.list
    └── lainos-privileged.list
```

### Build

```bash
cargo build --release --target x86_64-unknown-linux-musl
```

### Implementation Status

| Component | Status |
|-----------|--------|
| cgroup-v2 limit writing | ✅ |
| bwrap command line construction | ✅ |
| `rc_sandbox="NO"` opt-out check | ✅ |
| Landlock ruleset application | ✅ |
| `no_new_privs` | ✅ |
| Capability dropping | ✅ |
| Seccomp profile parsing | ✅ |
| Seccomp caching | ✅ |
| Seccomp loading | ✅ |
| Seccomp profiles (3 files) | ✅ |
| `openrc-security-status` script | ✅ |
| AppArmor profiles for sandboxing components | ❌ |

---

## Testing / Verification

`openrc-security-status` will automate verification of:

- Mount namespace isolation (`/proc/<pid>/mounts`)
- cgroup-v2 limits (`/sys/fs/cgroup/openrc/<service>/`)
- seccomp (`/proc/<pid>/status | grep Seccomp`)
- `no_new_privs` (`/proc/<pid>/status | grep NoNewPrivs`)
- Landlock (out-of-scope path access attempts)
- AppArmor (profile enforcement)

---

## Open Questions

- OpenRC cgroup-v2 ownership behavior on the target Gentoo version
- Landlock + AppArmor interaction under load (performance)
- Whether/when to introduce seccomp profile includes (`@lainos-base` shared across services) ~ v1 uses flat profiles

---

*Implementation complete. Ready for ISO integration.*
