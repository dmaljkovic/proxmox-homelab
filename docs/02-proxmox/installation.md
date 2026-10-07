# Proxmox VE Installation

## Prerequisites

- Generic Laptop connected via Ethernet
- 4 GB+ USB stick
- Proxmox VE 9.x ISO
- Target disk: 128 GB M.2 SSD

## Download ISO

```bash
# On your PC
wget https://enterprise.proxmox.com/iso/proxmox-ve_9.0-1.iso -O proxmox-ve_9.0.iso
# Verify checksum
sha256sum proxmox-ve_9.0.iso
```

## Create Bootable USB

```bash
# Find USB device (CAREFUL - this DESTROYS data on target)
lsblk
# Example: /dev/sdb

sudo dd if=proxmox-ve_9.0.iso of=/dev/sdX bs=4M status=progress oflag=sync
sync
```

## Install on Generic Laptop

1. Boot Generic Laptop from USB (F12 → select USB)
2. **Welcome** → Next
3. **EULA** → Accept
4. **Target Disk** → Select 128 GB SSD
   - Filesystem: **ext4** (NOT ZFS)
   - Options: Default
5. **Location/Time** → Select timezone, keyboard
6. **Admin Password** → Set strong root password
   - *Will disable password auth after SSH key setup*
7. **Network Configuration**
   - Interface: `eno1` (Ethernet)
   - IP: `10.0.0.100/24` (static)
   - Gateway: `10.0.0.1`
   - DNS: `10.0.0.100` (Pi-hole) + `1.1.1.1` fallback
   - Hostname: `proxmox.example.com`
8. **Confirm** → Install
9. **Reboot** → Remove USB

## Post-Install Verification

```bash
# From your PC
ssh root@10.0.0.100
# Should prompt for password (first time)

# Check network
ip a show vmbr0
# Should show 10.0.0.100/24 on vmbr0

# Check DNS
cat /etc/resolv.conf
# nameserver 10.0.0.100
# nameserver 1.1.1.1
```

## Post-Install Hardening

### 1. SSH Key Setup (from your PC)

```bash
# Generate key if needed
ssh-keygen -t ed25519 -C "your@email.com"

# Copy to Proxmox
ssh-copy-id root@10.0.0.100

# Test key-only login
ssh root@10.0.0.100
```

### 2. Harden SSH

```bash
# On Proxmox
cat > /etc/ssh/sshd_config.d/99-hardening.conf << 'EOF'
PermitRootLogin prohibit-password
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding no
PrintMotd no
ClientAliveInterval 300
ClientAliveCountMax 2
MaxAuthTries 3
LoginGraceTime 30
EOF

systemctl reload sshd
```

### 3. Configure UFW

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow from 10.0.0.0/24 to any port 22 proto tcp
ufw allow from 10.0.0.0/24 to any port 8006 proto tcp
ufw enable

# Verify
ufw status numbered
```

### 4. Automatic Security Updates

```bash
apt update && apt install -y unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
# Select "Yes"

# Verify
cat /etc/apt/apt.conf.d/20auto-upgrades
```

### 5. Fail2ban

```bash
apt install -y fail2ban

cat > /etc/fail2ban/jail.local << 'EOF'
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 1h
findtime = 10m
EOF

systemctl enable --now fail2ban
```

### 6. Time Synchronization

```bash
apt install -y chrony
systemctl enable --now chrony

# Verify
chronyc tracking
chronyc sources -v
```

### 7. Disable Unneeded Services

```bash
# Disable Wi-Fi (not used)
systemctl disable --now wpa_supplicant 2>/dev/null || true

# Disable Bluetooth
systemctl disable --now bluetooth 2>/dev/null || true

# Verify only needed services
systemctl list-units --type=service --state=running
```

## Verification Checklist

- [ ] SSH key-only login works: `ssh root@10.0.0.100`
- [ ] Proxmox GUI: `https://10.0.0.100:8006` (LAN only)
- [ ] `ufw status` shows only 22/8006 from 10.0.0.0/24
- [ ] `systemctl status fail2ban` → active
- [ ] `chronyc tracking` → synchronized
- [ ] `ip a` shows vmbr0 with 10.0.0.100/24
- [ ] No password auth possible (test from another machine)

## Next Steps

Proceed to [Cloudflare Tunnel Setup](../03-networking/cloudflare-tunnel.md)