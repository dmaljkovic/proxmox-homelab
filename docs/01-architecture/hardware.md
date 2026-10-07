# Hardware Specification

## Generic Laptop

| Component | Specification |
|-----------|---------------|
| **CPU** | Intel Core Intel i7 (4C/8T) (4C/8T, 2.8-3.8 GHz) |
| **RAM** | 16 GB DDR4-2400 (2x8 GB, max 32 GB) |
| **Storage** | 128 GB M.2 PCIe SSD (system) + 2.5" SATA bay (empty) |
| **Network** | Gigabit Ethernet (Realtek RTL8111) + Wi-Fi (unused) |
| **GPU** | NVIDIA GTX 1050 4 GB (unused) |
| **Display** | 15.6" 1920x1080 (headless operation) |

## Proxmox Resource Budget

```
Total RAM: 16 GB
├── Proxmox Host: ~2 GB
├── LXC Containers: ~3.5 GB (allocated)
├── ZFS ARC: N/A (ext4)
├── Page Cache: ~4 GB
└── Free/Buffers: ~6.5 GB
```

## Storage Layout

```
128 GB M.2 SSD (ext4 + LVM-thin)
├── / (root): ~20 GB
├── /var/lib/vz (LVM-thin pool): ~100 GB
│   ├── CT-101 (nginx): 4 GB
│   ├── CT-102 (openldap): 4 GB
│   ├── CT-103 (authelia): 4 GB
│   ├── CT-104 (gitea): 8 GB
│   ├── CT-105 (grafana): 8 GB
│   ├── CT-106 (prometheus): 8 GB
│   ├── CT-107 (security): 4 GB
│   ├── backups: ~40 GB
│   └── templates/ISOs: ~10 GB
```

## Planned Storage Upgrade

| Priority | Upgrade | Purpose |
|----------|---------|---------|
| 1 | 2.5" SATA SSD (500 GB - 1 TB) | Backups, Gitea data, Prometheus retention |
| 2 | Larger M.2 NVMe (500 GB - 1 TB) | Root + containers, replace 128 GB |

## Power & Thermal

- **Idle**: ~15-20 W
- **Load**: ~45-65 W (CPU + containers)
- **Thermal**: Adequate for 24/7 operation (clean fans, replace paste if needed)
- **UPS**: Not currently used (laptop battery = built-in UPS)

## Why This Hardware Works

- **16 GB RAM**: Comfortable for 7 LXCs + host + cache
- **4C/8T CPU**: Handles concurrent compiles, Prometheus scraping, Git operations
- **Ethernet**: Stable, low-latency bridge networking
- **M.2 + 2.5" bay**: Upgrade path for storage
- **Laptop form factor**: Built-in battery (UPS), low power, quiet
