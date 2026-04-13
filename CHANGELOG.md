# Changelog

All notable changes to this configuration are documented here.

---

## [1.1] — 2026-04-13

### Added
- `kernel.core_uses_pid = 1` — append PID to core dump filenames (Lynis KRNL-6000)
- `fs.suid_dumpable = 0` — disable core dumps for setuid processes (Lynis KRNL-6000)

---

## [1.0] — 2026-04-04

### Added
- Full rewrite as `99-custom.conf` — drop-in design for `/etc/sysctl.d/`
- Memory management: `vm.swappiness=10`, `vm.dirty_ratio=15`, dirty writeback tuning
- Network stack: `rmem_max`, `wmem_max`, `net.core.netdev_max_backlog` for streaming/proxy
- Security hardening: `kernel.dmesg_restrict=1`, `kernel.kptr_restrict=2`, SYN flood protection
- IPv4/IPv6 forwarding for KVM bridge (`net.ipv4.ip_forward=1`)
- Conntrack tuning for Squid proxy workload
- `fs.inotify.max_user_watches` — extended for KDE/monitoring stack
- Full inline rationale for every parameter — explains the why, not just the what

### Changed
- Renamed from `sysct_configuration` to `99-custom.conf`
- Translated all comments to English
- Removed outdated streaming-specific hacks from 2024 draft

---

## [0.1] — 2024-11-16

### Added
- Initial draft — memory, network, streaming, security parameters
- French comments
