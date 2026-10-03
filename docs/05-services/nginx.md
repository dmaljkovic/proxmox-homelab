# Nginx Configuration

## Role

- TLS termination (at Cloudflare edge)
- Reverse proxy for all web services
- `auth_request` integration with Authelia
- Host-based routing (git, grafana, auth)
- Security headers, rate limiting

## Architecture

```
Cloudflare -> cloudflared -> Nginx (10.0.0.10:80)
    |
    |-- git.example.com -> auth_request -> Authelia -> proxy_pass -> Gitea (10.0.0.40:3000)
    |-- grafana.example.com -> auth_request -> Authelia -> proxy_pass -> Grafana (10.0.0.50:3000)
    |-- auth.example.com -> proxy_pass -> Authelia portal (10.0.0.30:9091)
```

## Installation (LXC 101)

```bash
# On Proxmox: create Debian 12 LXC (101), 256 MB RAM, 4 GB disk, static IP 10.0.0.10

# Inside LXC 101
apt update && apt install -y nginx

# Enable and start
systemctl enable --now nginx
```

## Main Config (`/etc/nginx/nginx.conf`)

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
    multi_accept on;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;

    # Trust Cloudflare headers for real IP
    set_real_ip_from 103.21.244.0/22;
    set_real_ip_from 103.22.200.0/22;
    set_real_ip_from 103.31.4.0/22;
    set_real_ip_from 104.16.0.0/13;
    set_real_ip_from 104.24.0.0/14;
    set_real_ip_from 108.162.192.0/18;
    set_real_ip_from 131.0.72.0/22;
    set_real_ip_from 141.101.64.0/18;
    set_real_ip_from 162.158.0.0/15;
    set_real_ip_from 172.64.0.0/13;
    set_real_ip_from 173.245.48.0/20;
    set_real_ip_from 188.114.96.0/20;
    set_real_ip_from 190.93.240.0/20;
    set_real_ip_from 197.234.240.0/22;
    set_real_ip_from 198.41.128.0/17;
    set_real_ip_from 2400:cb00::/32;
    set_real_ip_from 2405:8100::/32;
    set_real_ip_from 2405:b500::/32;
    set_real_ip_from 2606:4700::/32;
    set_real_ip_from 2803:f800::/32;
    set_real_ip_from 2c0f:f248::/32;
    set_real_ip_from 2a06:98c0::/29;
    real_ip_header CF-Connecting-IP;
    real_ip_recursive on;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    # Gzip
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml application/json;

    # Include site configs
    include /etc/nginx/sites-enabled/*;
}
```

## Site: git.example.com (`/etc/nginx/sites-available/git.example.com`)

```nginx
server {
    listen 80;
    server_name git.example.com;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; frame-ancestors 'self';" always;

    # Auth request to Authelia
    location / {
        auth_request /auth;
        auth_request_set $user $upstream_http_remote_user;
        auth_request_set $groups $upstream_http_remote_groups;
        auth_request_set $email $upstream_http_remote_email;

        proxy_pass http://10.0.0.40:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Remote-User $user;
        proxy_set_header Remote-Groups $groups;
        proxy_set_header Remote-Email $email;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        
        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Internal auth endpoint
    location = /auth {
        internal;
        proxy_pass http://10.0.0.30:9091/api/verify;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URL $scheme://$host$request_uri;
        proxy_set_header X-Original-Method $request_method;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Health check (no auth)
    location /healthz {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

## Site: grafana.example.com (`/etc/nginx/sites-available/grafana.example.com`)

```nginx
server {
    listen 80;
    server_name grafana.example.com;

    # Security headers (same as git)
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    location / {
        auth_request /auth;
        auth_request_set $user $upstream_http_remote_user;
        auth_request_set $groups $upstream_http_remote_groups;
        auth_request_set $email $upstream_http_remote_email;

        proxy_pass http://10.0.0.50:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Remote-User $user;
        proxy_set_header Remote-Groups $groups;
        proxy_set_header Remote-Email $email;
        
        # WebSocket for live updates
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Timeouts
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }

    location = /auth {
        internal;
        proxy_pass http://10.0.0.30:9091/api/verify;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URL $scheme://$host$request_uri;
        proxy_set_header X-Original-Method $request_method;
    }

    location /healthz {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

## Site: auth.example.com (`/etc/nginx/sites-available/auth.example.com`)

```nginx
server {
    listen 80;
    server_name auth.example.com;

    # No auth_request here - this IS the auth portal
    location / {
        proxy_pass http://10.0.0.30:9091;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Required for Authelia
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        
        # Buffering
        proxy_buffering off;
        proxy_request_buffering off;
    }

    location /healthz {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

## Enable Sites

```bash
ln -s /etc/nginx/sites-available/git.example.com /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/grafana.example.com /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/auth.example.com /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default

# Test and reload
nginx -t
systemctl reload nginx
```

## Rate Limiting (Optional)

```nginx
# In http block
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/s;
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;

# In server block for git/grafana
location / {
    limit_req zone=api burst=50 nodelay;
    # ... rest of config
}

location /user/login {
    limit_req zone=login burst=5 nodelay;
    # ... rest of config
}
```

## Verification

```bash
# Test config
nginx -t

# Check listening
ss -tlnp | grep :80

# Test locally
curl -H "Host: git.example.com" http://10.0.0.10:80
# Should return 401 (no auth) or redirect to auth

# Test via Cloudflare tunnel
curl -I https://git.example.com
# Should work end-to-end
```

## Next Steps

- [Authelia Configuration](../04-identity/authelia.md)
- [Gitea Configuration](../05-services/gitea.md)
- [Grafana Configuration](../05-services/grafana.md)
