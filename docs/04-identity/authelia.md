# Authelia SSO/OIDC Configuration

## Overview

Authelia provides Single Sign-On (SSO) and Multi-Factor Authentication (MFA) for the homelab. It acts as an OpenID Connect (OIDC) provider, integrating with OpenLDAP for authentication.

## Deployment

- **Container**: LXC 103
- **IP**: 10.0.0.30
- **RAM**: 256 MB
- **Disk**: 4 GB
- **OS**: Debian 12 (Bookworm)
- **Port**: 9091 (internal), 9090 (health)

## Installation

```bash
# On Proxmox host - enter container
pct enter 103

# Inside container
apt update && apt install -y curl gnupg2

# Add Authelia repository
curl -fsSL https://apt.authelia.com/authelia.gpg | gpg --dearmor -o /usr/share/keyrings/authelia.gpg
echo "deb [signed-by=/usr/share/keyrings/authelia.gpg] https://apt.authelia.com/debian stable main" > /etc/apt/sources.list.d/authelia.list

apt update && apt install -y authelia
```

## Configuration

### Main Config (`/etc/authelia/configuration.yml`)

```yaml
host: 0.0.0.0
port: 9091

log:
  level: info
  format: json

jwt_secret: "GENERATE_WITH_openssl_rand_base64_32"

default_redirection_url: https://auth.example.com

totp:
  issuer: example.com
  period: 30
  skew: 1

authentication_backend:
  ldap:
    address: ldaps://10.0.0.20:636
    base_dn: dc=unseen-uni,dc=xyz
    user_filter: (uid={input})
    bind_dn: cn=admin,dc=unseen-uni,dc=xyz
    bind_password: admin-password
    attributes:
      username: uid
      display_name: cn
      email: mail
      group: memberOf
    group_search:
      base_dn: ou=groups,dc=unseen-uni,dc=xyz
      filter: (memberUid={username})
      attribute: cn
    timeout: 5s

access_control:
  default_policy: deny
  rules:
    - domain: auth.example.com
      policy: bypass
    - domain:
        - git.example.com
        - grafana.example.com
      policy: two_factor
      subject:
        - group: admins
        - group: developers

session:
  name: authelia_session
  domain: example.com
  same_site: lax
  expiration: 1h
  inactivity: 5m
  remember_me_duration: 1M
  cookies:
    - domain: example.com
      authelia_url: https://auth.example.com
      default_redirection_url: https://auth.example.com

regulation:
  max_retries: 3
  find_time: 120s
  ban_time: 300s

storage:
  local:
    path: /var/lib/authelia/db.sqlite3

notifier:
  filesystem:
    filename: /var/lib/authelia/notification.txt
```

## Generate Secrets

```bash
# JWT secret (32 bytes base64)
openssl rand -base64 32

# Session secret
openssl rand -base64 32
```

## OIDC Clients

### Gitea

```yaml
# In configuration.yml under identity_providers.oidc.clients
- client_id: gitea
  client_secret: "GENERATE_SECRET"
  redirect_uris:
    - https://git.example.com/user/oauth2/oidc/callback
  scopes:
    - openid
    - profile
    - email
    - groups
```

### Grafana

```yaml
- client_id: grafana
  client_secret: "GENERATE_SECRET"
  redirect_uris:
    - https://grafana.example.com/login/generic_oauth
  scopes:
    - openid
    - profile
    - email
```

## Systemd Service

```bash
systemctl enable --now authelia
systemctl status authelia
```

## Nginx Integration

Nginx (LXC 101) uses `auth_request` to Authelia:

```nginx
location / {
    auth_request /auth;
    auth_request_set $user $upstream_http_remote_user;
    auth_request_set $groups $upstream_http_remote_groups;
    proxy_pass http://10.0.0.40:3000;
    ...
}

location = /auth {
    internal;
    proxy_pass http://10.0.0.30:9091/api/verify;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_set_header X-Original-URL $scheme://$host$request_uri;
    proxy_set_header X-Original-Method $request_method;
}
```

## Authelia Portal (No auth_request)

```nginx
server {
    listen 80;
    server_name auth.example.com;
    location / {
        proxy_pass http://10.0.0.30:9091;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        ...
    }
}
```

## Firewall

```bash
ufw allow from 10.0.0.10 to any port 9091 proto tcp
ufw allow from 10.0.0.10 to any port 9090 proto tcp
ufw reload
```

## Verification

```bash
# Health check
curl -s http://10.0.0.30:9090/api/health

# Login page
curl -I https://auth.example.com

# Test OIDC discovery
curl -s https://auth.example.com/.well-known/openid-configuration
```

## Backup

```bash
# SQLite database
cp /var/lib/authelia/db.sqlite3 /backup/authelia-$(date +%F).sqlite3
```

## References

- [Authelia Docs](https://www.authelia.com/docs/)
- [OIDC Spec](https://openid.net/specs/openid-connect-core-1_0.html)
- [LDAP Backend](https://www.authelia.com/docs/configuration/authenticators/ldap/)
