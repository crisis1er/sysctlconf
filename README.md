# sysctl-tumbleweed-config

![Platform](https://img.shields.io/badge/platform-openSUSE%20Tumbleweed-73BA25)
![Shell](https://img.shields.io/badge/kernel-sysctl-4EAA25)
![License](https://img.shields.io/badge/license-MIT-green)
![Security](https://img.shields.io/badge/hardened-kernel%20%2B%20network-red)
![BBR](https://img.shields.io/badge/TCP-BBR%20%2B%20FQ-blue)

Production-ready custom sysctl configuration for **openSUSE Tumbleweed** — tuned for desktop multimedia, fibre broadband, KVM virtualization, and a hardened privacy-first network stack (Squid + Unbound).

Every parameter is commented with the reason it was changed from the kernel default. Deployed and validated on a live system — not a copy-paste collection.

> **Drop-in design** — placed in `/etc/sysctl.d/99-custom.conf`, it loads last and overrides all distribution defaults without touching any system-managed file. Update-safe, instantly reversible.

---

## Why a drop-in file instead of editing sysctl.conf?

On modern Linux distributions, `/etc/sysctl.conf` is managed by the distribution or by tools like YaST. Editing it directly risks having your changes overwritten by a system update or a reconfiguration tool.

The correct approach is to place a custom file in `/etc/sysctl.d/` with a high numeric prefix (here `99-`) so it is loaded **last** and takes precedence over all system defaults. This gives you:

| Advantage | Details |
|-----------|---------|
| **Update-safe** | Distribution updates never touch your file |
| **Fully isolated** | Your parameters are in one identifiable place |
| **Load order control** | `99-` prefix guarantees your values win |
| **Easy rollback** | Remove or rename the file — system defaults restore instantly |
| **Audit-friendly** | `sysctl --system` shows each file loaded and in which order |

---

## Overview

This configuration covers 9 areas, each with inline comments explaining the reason behind every parameter.

| Section | Focus |
|---------|-------|
| 1 | Memory management |
| 2 | Network buffers and queues |
| 3 | TCP/IP and connections |
| 4 | Network security — IPv4 |
| 5 | IPv6 |
| 6 | Netfilter / Conntrack |
| 7 | Kernel hardening |
| 8 | Filesystem — inotify |
| 9 | Kernel log filtering |

---

## Section details

### 1 — Memory management

| Parameter | Value | Why |
|-----------|-------|-----|
| `vm.swappiness` | 30 | Balanced for zram/zstd — low enough to favor RAM, high enough for zram to compress efficiently |
| `vm.vfs_cache_pressure` | 40 | Keeps inode/dentry cache in RAM longer — faster filesystem lookups |
| `vm.dirty_background_bytes` | 64 MB | Writeback in bytes (not ratio) for predictable behavior on USB SSD |
| `vm.dirty_bytes` | 256 MB | Hard limit before processes block on writeback |
| `vm.dirty_writeback_centisecs` | 1500 | Flush every 15s instead of default 5s — reduces USB SSD write cycles |
| `vm.max_map_count` | 262144 | Required by Chromium, Firefox, and Kodi (large mmap usage) |
| `vm.min_free_kbytes` | 131072 | Prevents kernel memory exhaustion under Btrfs pressure |
| `vm.oom_kill_allocating_task` | 1 | OOM killer targets the offending process, not a random victim |

### 2 — Network buffers and queues

| Parameter | Value | Why |
|-----------|-------|-----|
| `net.core.netdev_max_backlog` | 4096 | Queues packets for the kernel to process — avoids drops under burst |
| `net.core.somaxconn` | 4096 | Max pending connections — relevant for Squid and Unbound |
| `net.core.rmem_max` / `wmem_max` | 8 MB | Socket buffer ceiling — enough for fibre without wasting RAM |
| `net.core.default_qdisc` | fq | Fair Queue: isolates flows and reduces latency under load |
| `net.core.bpf_jit_harden` | 2 | Mitigates Spectre-class attacks via BPF JIT |

### 3 — TCP/IP and connections

| Parameter | Value | Why |
|-----------|-------|-----|
| `tcp_congestion_control` | bbr | Google BBR: significantly better throughput and latency on fibre |
| `tcp_tw_reuse` | 1 | Reuse TIME_WAIT sockets immediately — reduces port exhaustion |
| `tcp_fin_timeout` | 10s | Default 60s is excessive; 10s frees sockets faster |
| `tcp_slow_start_after_idle` | 0 | Keeps full bandwidth after a pause (streaming, Squid, DoT) |
| `tcp_fastopen` | 3 | Reduces connection latency by sending data in the SYN |
| `tcp_ecn` | 1 | Explicit Congestion Notification — avoids packet loss signaling |
| `tcp_keepalive_time` | 60s | Detects dead connections in 60s instead of default 2 hours |
| `tcp_max_tw_buckets` | 65536 | Limits TIME_WAIT sockets in memory |

### 4 — Network security — IPv4

| Parameter | Value | Why |
|-----------|-------|-----|
| `conf.all.rp_filter` | 1 | Strict reverse path filter — drops spoofed source addresses |
| `accept_redirects` | 0 | Disables ICMP redirect acceptance — prevents route injection |
| `send_redirects` | 0 | Does not send redirects — not a router on LAN interfaces |
| `accept_source_route` | 0 | Source routing is a legacy attack vector — disabled |
| `tcp_rfc1337` | 1 | Drops RST packets that arrive in TIME_WAIT — RFC 1337 fix |
| `ip_forward` | 1 | Required for KVM bridge networking and Squid transparent proxy |
| `log_martians` | 0 | Disabled — generates excessive noise on desktop with KVM |

### 5 — IPv6

Full dual-stack support with the same security hardening as IPv4: redirects and source routing disabled. `accept_ra = 2` is set so the interface accepts Router Advertisements **even when forwarding is enabled** (required for KVM bridge + SLAAC).

### 6 — Netfilter / Conntrack

| Parameter | Value | Why |
|-----------|-------|-----|
| `nf_conntrack_max` | 131072 | Default (65536) is too low with Squid + multiple VMs + firewalld |
| `nf_conntrack_tcp_timeout_established` | 86400s | Default is 5 days — pointless for desktop; 24h is sufficient |

### 7 — Kernel hardening

| Parameter | Value | Why |
|-----------|-------|-----|
| `dmesg_restrict` | 1 | Kernel log readable by root only — prevents information leaks |
| `kptr_restrict` | 2 | Hides kernel symbol addresses from all users |
| `perf_event_paranoid` | 2 | Restricts performance counters to root |
| `unprivileged_bpf_disabled` | 1 | Blocks unprivileged eBPF — major attack surface on desktop |
| `randomize_va_space` | 2 | Full ASLR including heap — default on most distros, verified here |
| `yama.ptrace_scope` | 1 | Restricts ptrace to parent processes — mitigates local privilege escalation |
| `kernel.sysrq` | 244 | Selective SysRq: SAK + sync + remount-ro + reboot — excludes raw memory dump |
| `kernel.panic` | 10 | Auto-reboot 10s after kernel panic |
| `kernel.core_uses_pid` | 1 | Appends PID to core dump filename — prevents overwrites on concurrent crashes |
| `fs.suid_dumpable` | 0 | Disables core dumps for setuid processes — reduces attack surface (Lynis KRNL-6000) |

### 8 — Filesystem — inotify

Raised from kernel defaults to support IDEs, file managers, systemd journal, and real-time streaming apps simultaneously.

| Parameter | Value |
|-----------|-------|
| `fs.inotify.max_user_watches` | 131072 |
| `fs.inotify.max_user_instances` | 2048 |

### 9 — Kernel log filtering

```
kernel.printk = 3 4 1 3
```
Limits console output to warnings and errors. Reduces noise in journald and avoids log flooding during normal operation.

---

## Requirements

| Component | Version |
|-----------|---------|
| openSUSE | Tumbleweed (also compatible with Leap) |
| Kernel | 5.x+ (BBR and eBPF hardening required) |

---

## Installation

### 1. Backup existing configuration

```bash
sudo cp /etc/sysctl.d/99-custom.conf /etc/sysctl.d/99-custom.conf.bak
```

### 2. Deploy the configuration

```bash
sudo cp 99-custom.conf /etc/sysctl.d/99-custom.conf
sudo chmod 644 /etc/sysctl.d/99-custom.conf
```

### 3. Apply immediately without reboot

```bash
sudo sysctl --system
```

### 4. Verify active values

```bash
sysctl vm.swappiness net.ipv4.tcp_congestion_control kernel.kptr_restrict
```

---

## Verify load order

```bash
sudo sysctl --system 2>&1 | grep "Applying\|99-custom"
```

This shows every file loaded by the kernel and confirms `99-custom.conf` is applied last.

---

## After uninstall

To restore system defaults, remove the file and reload:

```bash
sudo rm /etc/sysctl.d/99-custom.conf
sudo sysctl --system
```

---

## Lynis audit

This configuration is validated against [Lynis](https://cisofy.com/lynis/). After running an audit, filter results with:

```bash
# Suggestions only
grep Suggestion /var/log/lynis.log

# Warnings only
grep Warning /var/log/lynis.log
```

---

## Contributing

Issues and pull requests are welcome.  
Please include your kernel version (`uname -r`) and openSUSE version in bug reports.

---

## License

MIT License — see [LICENSE](LICENSE) for details.
