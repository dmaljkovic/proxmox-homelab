# Proxmox Bridge (vmbr0)

## Overview

`vmbr0` is the primary Linux bridge on the Proxmox host that connects all LXC containers to the physical network.

## Configuration

### Proxmox Host (`/etc/network/interfaces`)

```bash
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 10.0.0.99/24
    gateway 10.0.0.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    dns-nameservers 10.0.0.100 1.1.1.1
    dns-search example.com
```

### Parameters Explained

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `bridge-ports eno1` | Physical NIC | Binds bridge to physical Ethernet |
| `bridge-stp off` | Disabled | No Spanning Tree Protocol needed |
| `bridge-fd 0` | 0 seconds | No forwarding delay |
| `address` | 10.0.0.99/24 | Proxmox host IP on bridge |
| `gateway` | 10.0.0.1 | Upstream router |

## LXC Network Configuration

Each container connects to vmbr0 via its config:

```bash
# /etc/pve/lxc/101.conf (example for Nginx)
net0: name=eth0,bridge=vmbr0,ip=10.0.0.10/24,gw=10.0.0.1,hwaddr=aa:bb:cc:dd:ee:01
```

| Field | Value | Purpose |
|-------|-------|---------|
| `name=eth0` | Interface name | Inside container |
| `bridge=vmbr0` | Bridge name | Connects to host bridge |
| `ip=10.0.0.10/24` | Static IP + CIDR | Container IP |
| `gw=10.0.0.1` | Gateway | Upstream router |
| `hwaddr=...` | Static MAC | Consistent identity |

## IP Assignment Table

| Container | IP | MAC Suffix |
|-----------|-----|------------|
| Nginx (101) | 10.0.0.10 | `:ee:01` |
| OpenLDAP (102) | 10.0.0.20 | `:ee:02` |
| Authelia (103) | 10.0.0.30 | `:ee:03` |
| Gitea (104) | 10.0.0.40 | `:ee:04` |
| Grafana (105) | 10.0.0.50 | `:ee:05` |
| Prometheus (106) | 10.0.0.60 | `:ee:06` |

## Verification

```bash
# On Proxmox host - check bridge
ip link show vmbr0
bridge link show dev vmbr0

# Check LXC interfaces
for i in {101..106}; do pct exec $i -- ip -br a; done

# Test connectivity
pct exec 101 -- ping -c 2 10.0.0.20
pct exec 101 -- ping -c 2 10.0.0.1
```
