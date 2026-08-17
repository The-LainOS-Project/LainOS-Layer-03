```bash
doas ./openrc-security-status.sh
=== lainOS OpenRC Isolation Stack — Containment Layer Verification ===

=== dnsmasq ===
  process               running as PID 14342 (comm=dnsmasq)
  namespace: mount      ISOLATED (own mount ns)
  namespace: network    shared with host (rc_network_access=YES, correct)
  namespace: pid        ISOLATED (own PID namespace)
  cgroup limits         ENFORCED (memory.max=67108864 pids.max=20, PID confirmed member)
  seccomp-bpf           ACTIVE (filter mode, 1 filter(s) loaded, no_new_privs set)
  Landlock              cannot verify directly (no /proc/status field exists for it);
                        inferred from successful sandboxed operation below
  capabilities          NARROWED (CapEff=0x0000000000002400 CapBnd=0x00000000000024c3, conf.d requests: CAP_SETUID,CAP_SETGID,CAP_DAC_OVERRIDE,CAP_NET_BIND_SERVICE,CAP_CHOWN,CAP_NET_RAW)
  AppArmor              CONFINED (/usr/bin/dnsmasq)

=== unbound ===
  process               running as PID 6602 (comm=unbound)
  namespace: mount      ISOLATED (own mount ns)
  namespace: network    shared with host (rc_network_access=YES, correct)
  namespace: pid        ISOLATED (own PID namespace)
  cgroup limits         ENFORCED (memory.max=268435456 pids.max=20, PID confirmed member)
  seccomp-bpf           ACTIVE (filter mode, 1 filter(s) loaded, no_new_privs set)
  Landlock              cannot verify directly (no /proc/status field exists for it);
                        inferred from successful sandboxed operation below
  capabilities          NARROWED (CapEff=0x0000000001000402 CapBnd=0x0000000001000402, conf.d requests: CAP_NET_BIND_SERVICE,CAP_SYS_RESOURCE,CAP_DAC_OVERRIDE)
  AppArmor              CONFINED (/usr/bin/unbound)

=== dnscrypt-proxy ===
  process               running as PID 14281 (comm=dnscrypt-proxy)
  namespace: mount      ISOLATED (own mount ns)
  namespace: network    shared with host (rc_network_access=YES, correct)
  namespace: pid        ISOLATED (own PID namespace)
  cgroup limits         ENFORCED (memory.max=134217728 pids.max=100, PID confirmed member)
  seccomp-bpf           ACTIVE (filter mode, 1 filter(s) loaded, no_new_privs set)
  Landlock              cannot verify directly (no /proc/status field exists for it);
                        inferred from successful sandboxed operation below
  capabilities          NARROWED (CapEff=0x0000000000001406 CapBnd=0x0000000000001406, conf.d requests: CAP_DAC_OVERRIDE,CAP_DAC_READ_SEARCH,CAP_NET_ADMIN,CAP_NET_BIND_SERVICE)
  AppArmor              CONFINED (/usr/bin/dnscrypt-proxy)

=== tor ===
  process               running as PID 19905 (comm=tor)
  namespace: mount      ISOLATED (own mount ns)
  namespace: network    shared with host (rc_network_access=YES, correct)
  namespace: pid        ISOLATED (own PID namespace)
  cgroup limits         ENFORCED (memory.max=max pids.max=20, PID confirmed member)
  seccomp-bpf           ACTIVE (filter mode, 1 filter(s) loaded, no_new_privs set)
  Landlock              cannot verify directly (no /proc/status field exists for it);
                        inferred from successful sandboxed operation below
  capabilities          NARROWED (CapEff=0x0000000000000000 CapBnd=0x00000000002025c6, conf.d requests: CAP_SETUID,CAP_SETGID,CAP_SETPCAP,CAP_NET_BIND_SERVICE,CAP_NET_RAW,CAP_DAC_OVERRIDE,CAP_DAC_READ_SEARCH,CAP_SYS_ADMIN)
  AppArmor              CONFINED (/usr/bin/tor)

=== dhcpcd ===
  process               running as PID 19034 (comm=dhcpcd)
  namespace: mount      ISOLATED (own mount ns)
  namespace: network    shared with host (rc_network_access=YES, correct)
  namespace: pid        shared with host (rc_unshare_pid=NO, correct)
  cgroup limits         ENFORCED (memory.max=max pids.max=30, PID confirmed member)
  seccomp-bpf           ACTIVE (filter mode, 2 filter(s) loaded, no_new_privs set)
  Landlock              cannot verify directly (no /proc/status field exists for it);
                        inferred from successful sandboxed operation below
  capabilities          NARROWED (CapEff=0x0000000000000000 CapBnd=0x00000000002434e6, conf.d requests: CAP_NET_ADMIN,CAP_NET_RAW,CAP_NET_BIND_SERVICE,CAP_SETGID,CAP_SETUID,CAP_DAC_OVERRIDE,CAP_DAC_READ_SEARCH,CAP_SYS_ADMIN,CAP_SYS_CHROOT,CAP_KILL)
  AppArmor              CONFINED (/usr/bin/dhcpcd)

=== chrony ===
  process               running as PID 1206 (comm=chronyd)
  namespace: mount      ISOLATED (own mount ns)
  namespace: network    shared with host (rc_network_access=YES, correct)
  namespace: pid        shared with host (rc_unshare_pid=NO, correct)
  cgroup limits         ENFORCED (memory.max=67108864 pids.max=10, PID confirmed member)
  seccomp-bpf           ACTIVE (filter mode, 1 filter(s) loaded, no_new_privs set)
  Landlock              cannot verify directly (no /proc/status field exists for it);
                        inferred from successful sandboxed operation below
  capabilities          NARROWED (CapEff=0x0000000002000400 CapBnd=0x00000000020004c2, conf.d requests: CAP_NET_BIND_SERVICE,CAP_SETUID,CAP_SETGID,CAP_DAC_OVERRIDE,CAP_SYS_TIME)
  AppArmor              CONFINED (/usr/bin/chronyd)

=== acpid ===
  process               running as PID 6747 (comm=acpid)
  namespace: mount      ISOLATED (own mount ns)
  namespace: network    shared with host (rc_network_access=YES, correct)
  namespace: pid        ISOLATED (own PID namespace)
  cgroup limits         ENFORCED (memory.max=33554432 pids.max=10, PID confirmed member)
  seccomp-bpf           ACTIVE (filter mode, 1 filter(s) loaded, no_new_privs set)
  Landlock              cannot verify directly (no /proc/status field exists for it);
                        inferred from successful sandboxed operation below
  capabilities          NARROWED (CapEff=0x0000000000000000 CapBnd=0x0000000000000000, conf.d requests: none)
  AppArmor              CONFINED (/usr/bin/acpid)

=== syslog-ng ===
  process               running as PID 6804 (comm=syslog-ng-main)
  namespace: mount      ISOLATED (own mount ns)
  namespace: network    ISOLATED (rc_network_access!=YES, correct)
  namespace: pid        ISOLATED (own PID namespace)
  cgroup limits         ENFORCED (memory.max=134217728 pids.max=10, PID confirmed member)
  seccomp-bpf           ACTIVE (filter mode, 1 filter(s) loaded, no_new_privs set)
  Landlock              cannot verify directly (no /proc/status field exists for it);
                        inferred from successful sandboxed operation below
  capabilities          NARROWED (CapEff=0x0000000000000000 CapBnd=0x0000000401203ddf, conf.d requests: CAP_CHOWN,CAP_DAC_OVERRIDE,CAP_DAC_READ_SEARCH,CAP_FOWNER,CAP_FSETID,CAP_NET_BIND_SERVICE,CAP_NET_ADMIN,CAP_NET_BROADCAST,CAP_NET_RAW,CAP_SETGID,CAP_SETPCAP,CAP_SETUID,CAP_SYS_ADMIN,CAP_SYS_RESOURCE,CAP_SYSLOG)
  AppArmor              CONFINED (/usr/bin/syslog-ng)

=== Functional proof ===
  chrony                real NTP sync confirmed
  unbound               real DNS resolution confirmed
  syslog-ng             logger round-trip confirmed

=== Summary ===
All containment layers confirmed active on all services.
```
