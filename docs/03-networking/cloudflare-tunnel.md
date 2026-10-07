# Cloudflare Tunnel Setup

## Overview

Since the ISP uses CG-NAT (no public IPv4), we use Cloudflare Tunnel for secure, zero-trust access to internal services.

```
Internet -> Cloudflare Edge -> Tunnel (QUIC/443) -> cloudflared (Proxmox host) -> Nginx (10.0.0.10:80)
```

## Prerequisites

- Cloudflare account with `example.com` zone
- Domain already on Cloudflare (nameservers pointing to Cloudflare)
- Proxmox host at `10.0.0.100` with outbound internet access
- Nginx LXC planned at `10.0.0.10:80`

## Step 1: Install cloudflared on Your PC

```bash
# Linux (Debian/Ubuntu)
wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# macOS
brew install cloudflared

# Verify
cloudflared --version
```

## Step 2: Authenticate

```bash
cloudflared tunnel login
```

- Browser opens -> select `example.com` -> Authorize
- Creates `~/.cloudflared/cert.pem`

## Step 3: Create Tunnel

```bash
cloudflared tunnel create proxmox-homelab
```

Output:
```
Tunnel credentials written to /home/user/.cloudflared/<TUNNEL_ID>.json
```

**Save the TUNNEL_ID** (UUID).

## Step 4: Configure Tunnel

Create the configuration file:

```bash
cat > ~/.cloudflared/config.yml << 'EOF'
tunnel: <TUNNEL_ID>
credentials-file: /home/user/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: git.example.com
    service: http://10.0.0.10:80
    originRequest:
      noTLSVerify: true
  - hostname: grafana.example.com
    service: http://10.0.0.10:80
    originRequest:
      noTLSVerify: true
  - hostname: auth.example.com
    service: http://10.0.0.10:80
    originRequest:
      noTLSVerify: true
  - service: http_status:404
EOF
```

**Replace `<TUNNEL_ID>`** with the UUID from Step 3.

### Configuration Explained

| Field | Description |
|-------|-------------|
| `tunnel` | The tunnel UUID from Step 3 |
| `credentials-file` | Path to the tunnel credentials JSON |
| `ingress` | Routing rules (processed top-to-bottom) |
| `hostname` | Public hostname to match |
| `service` | Internal service URL (Nginx on LXC 101) |
| `noTLSVerify` | Skip TLS verification for internal HTTP |
| `http_status:404` | Catch-all for unmatched hostnames |

## Step 5: Create DNS Records

```bash
cloudflared tunnel route dns proxmox-homelab git.example.com
cloudflared tunnel route dns proxmox-homelab grafana.example.com
cloudflared tunnel route dns proxmox-homelab auth.example.com
```

This creates CNAME records:
- `git.example.com` -> `<TUNNEL_ID>.cfargotunnel.com`
- `grafana.example.com` -> `<TUNNEL_ID>.cfargotunnel.com`
- `auth.example.com` -> `<TUNNEL_ID>.cfargotunnel.com`

## Step 6: Deploy to Proxmox Host

```bash
# On Proxmox host
apt update && apt install -y cloudflared

# Copy credentials and config
mkdir -p /etc/cloudflared
scp ~/.cloudflared/<TUNNEL_ID>.json root@10.0.0.100:/etc/cloudflared/
scp ~/.cloudflared/config.yml root@10.0.0.100:/etc/cloudflared/

# Install as systemd service
cloudflared service install
systemctl enable --now cloudflared
```

## Step 7: Verify on Proxmox Host

```bash
# Check service status
systemctl status cloudflared

# Check tunnel connection
cloudflared tunnel list
# Should show "proxmox-homelab" with CONNECTED status

# Check logs
journalctl -u cloudflared -f
```

## Step 8: Verify End-to-End

```bash
# From your PC (outside network)
curl -I https://git.example.com
# Should return HTTP headers (even if 404/502 - Nginx not ready yet)

# Check DNS
dig git.example.com +short
# Should return Cloudflare IPs
```

## Cloudflare Dashboard Settings

### SSL/TLS
- **Overview**: Full (Strict) - requires valid cert on origin
- **Edge Certificates**: Automatic HTTPS Rewrites ON
- **Origin Server**: Create Origin CA certificate (optional, for Full Strict)

### DNS
- Records created above should show "Proxied" (orange cloud)

### Access (Optional - for additional auth)
- Can add Access policies for extra protection
- Not needed if Authelia handles auth

### Workers & Pages (Optional)
- Can add Workers for custom logic

## Troubleshooting

### Tunnel Not Connecting
```bash
# Check logs
journalctl -u cloudflared -n 50

# Common issues:
# - No internet access from Proxmox host
# - Firewall blocking outbound 443
# - Invalid credentials file
```

### 502 Bad Gateway
```bash
# Check Nginx is running on 10.0.0.10:80
ssh root@10.0.0.100
ssh root@10.0.0.10 "systemctl status nginx"
curl http://10.0.0.10:80
```

### DNS Not Resolving
```bash
# Check Cloudflare DNS tab
# Records should be CNAME to <TUNNEL_ID>.cfargotunnel.com
# Proxy status: Proxied (orange cloud)
```

## Security Notes

- Tunnel is **outbound only** - no open ports on router/firewall
- Cloudflare handles TLS termination at edge
- Origin (Nginx) uses HTTP internally (no cert management needed)
- `noTLSVerify: true` because internal HTTP only
- Cloudflare WAF protects against common attacks
- Consider Cloudflare Access for additional layer

---

## Next Steps

Proceed to [Nginx Configuration](../05-services/nginx.md)