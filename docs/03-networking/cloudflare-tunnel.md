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
