# Firewall Configuration

## Proxmox Host (UFW)

```bash
# Default policies
ufw default deny incoming
ufw default allow outgoing

# LAN management (192.168.0.0/24)
ufw allow from 192.168.0.0/24 to any port 22 proto tcp    # SSH
ufw allow from 192.168.0.0/24 to any port 8006 proto tcp  # Proxmox GUI

# Cloudflare Tunnel (outbound only)
ufw allow out 443 proto tcp  # cloudflared to Cloudflare
ufw allow out 53 proto udp   # DNS

# Enable
ufw enable

# Verify
ufw status numbered
```

Expected output:
```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    192.168.0.0/24
[ 2] 8006/tcp                   ALLOW IN    192.168.0.0/24
[ 3] 443/tcp                    ALLOW OUT   Anywhere
[ 4] 53/udp                     ALLOW OUT   Anywhere
```

## Internal LXC Firewall Matrix

| Source | Destination | Port | Proto | Purpose |
|--------|-------------|------|-------|---------|
| Nginx (101) | Gitea (104) | 3000 | TCP | Reverse proxy |
| Nginx (101) | Grafana (105) | 3000 | TCP | Reverse proxy |
| Nginx (101) | Authelia (103) | 9091 | TCP | auth_request |
| Authelia (103) | LDAP (102) | 636 | TCP | LDAPS bind |
| Gitea (104) | Authelia (103) | 443 | TCP | OIDC discovery |
| Grafana (105) | Authelia (103) | 443 | TCP | OIDC discovery |
| Prometheus (106) | All LXCs | 9090,9100,etc | TCP | Metrics scrape |
| Security (107) | All | Various | TCP/UDP | Testing/audit |

## LXC-Level Firewall (iptables/nftables)

Each LXC should run its own firewall. Example for Gitea (192.168.0.40):

```bash
# On Gitea LXC
apt install -y iptables-persistent

# Allow only from Nginx and Prometheus
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -s 192.168.0.10 -p tcp --dport 3000 -j ACCEPT  # Nginx
iptables -A INPUT -s 192.168.0.60 -p tcp --dport 3000 -j ACCEPT  # Prometheus
iptables -A INPUT -p tcp --dport 22 -j ACCEPT  # SSH from LAN (optional)
iptables -A INPUT -j DROP

# Save
iptables-save > /etc/iptables/rules.v4
```

## Cloudflare WAF Rules (Dashboard)

Recommended rules to enable:

1. **Managed Rulesets**
   - Cloudflare Managed Ruleset: ON
   - OWASP Managed Ruleset: ON (paranoia level 1)

2. **Custom Rules**
   - Block known bad user agents
   - Rate limit login endpoints
   - Block access to sensitive paths

Example custom rule:
```
Field: URI Path
Operator: contains
Value: /admin
Action: Block (or Challenge)
```

## Fail2ban (Proxmox Host)

```bash
# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 1h
findtime = 10m
backend = systemd
```

## Verification

```bash
# Test from LAN (should work)
ssh root@192.168.0.99
curl https://192.168.0.99:8006

# Test from internet (should fail)
# nmap -p 22,80,443,8006 <public-ip>
# All ports should be filtered/closed

# Test Cloudflare tunnel
curl -I https://git.unseen-uni.xyz
# Should work via tunnel
```
