# Authelia SSO / MFA Configuration

## Overview

Authelia provides Single Sign-On (SSO), second-factor and OpenID Connect (OIDC)
for the homelab. It authenticates users against OpenLDAP and protects reverse
proxied services through Nginx `auth_request`. Services such as Gitea and Grafana
use Authelia as their OIDC provider.

## Deployment

- **Container**: LXC 103
- **IP**: 10.0.0.30
- **RAM**: 256 MB
- **Disk**: 4 GB
- **OS**: Debian 12 (Bookworm)
- **Port**: 9091

## Installation

The Debian repository occasionally returns `404` for the current release. If that
happens, install the upstream binary from the GitHub release instead:

```bash
# On the Proxmox host - enter the container
pct enter 103

# Inside the container (binary fallback)
VER=4.39.28
curl -fsSL -o /tmp/authelia.tar.gz \
  "https://github.com/authelia/authelia/releases/download/v${VER}/authelia-v${VER}-linux-amd64.tar.gz"
tar -xzf /tmp/authelia.tar.gz -C /usr/local/bin authelia
chmod +x /usr/local/bin/authelia

# Create a dedicated service account and directories
adduser --system --group --no-create-home --home /var/lib/authelia authelia
install -d -o authelia -g authelia /var/lib/authelia /config/certs /config/ca
```

## Configuration (`/etc/authelia/configuration.yml`)

> **Version note (v4.38+):** `storage.encryption_key` is a direct child of
> `storage` (not `storage.local`), LDAP filters use the `{input}` placeholder,
> and OIDC uses a `jwks` list plus a required `hmac_secret` in place of the
> deprecated `issuer_private_key`.

```yaml
server:
  address: tcp://0.0.0.0:9091

log:
  level: info
  format: json

# Trust the self-signed CA used by OpenLDAP for LDAPS verification
certificates_directory: /config/ca

identity_validation:
  reset_password:
    jwt_secret: "<RANDOM_64_CHAR>"

totp:
  issuer: example.com
  period: 30
  skew: 1

authentication_backend:
  ldap:
    implementation: custom
    address: ldaps://10.0.0.20:636
    start_tls: false
    user: cn=admin,dc=example,dc=com
    password: admin-password
    base_dn: dc=example,dc=com
    # {username_attribute} is replaced by the attribute name (uid); {input} is
    # the value the user typed.
    users_filter: "(&({username_attribute}={input})(objectClass=posixAccount))"
    groups_filter: "(&(memberUid={username})(objectClass=posixGroup))"
    attributes:
      username: uid
      display_name: cn
      mail: mail
      member_of: memberOf

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
        - "group:admins"
        - "group:developers"

session:
  name: authelia_session
  same_site: lax
  expiration: 1h
  inactivity: 5m
  remember_me: 1M
  secret: "<RANDOM_64_CHAR>"
  cookies:
    - domain: example.com
      authelia_url: https://auth.example.com

regulation:
  max_retries: 3
  find_time: 120s
  ban_time: 300s

storage:
  encryption_key: "<RANDOM_64_CHAR>"
  local:
    path: /var/lib/authelia/db.sqlite3

notifier:
  filesystem:
    filename: /var/lib/authelia/notification.txt

identity_providers:
  oidc:
    hmac_secret: "<RANDOM_64_CHAR>"
    jwks:
      - key_id: 'authelia'
        algorithm: 'RS256'
        use: 'sig'
        key: |
          -----BEGIN PRIVATE KEY-----
          ... 2048-bit RSA private key (PKCS#8) ...
          -----END PRIVATE KEY-----
    cors:
      endpoints:
        - authorization
        - token
        - revocation
        - introspection
    clients:
      - client_id: gitea
        client_name: Gitea
        client_secret: "<HASHED_CLIENT_SECRET>"
        redirect_uris:
          - https://git.example.com/user/oauth2/oidc/callback
        scopes:
          - openid
          - profile
          - email
          - groups
          - offline_access
        grant_types:
          - authorization_code
          - refresh_token
        response_types:
          - code

      - client_id: grafana
        client_name: Grafana
        client_secret: "<HASHED_CLIENT_SECRET>"
        redirect_uris:
          - https://grafana.example.com/login/generic_oauth
        scopes:
          - openid
          - profile
          - email
          - offline_access
        grant_types:
          - authorization_code
          - refresh_token
        response_types:
          - code
```

## Generate Secrets

Generate every `<RANDOM_64_CHAR>` value independently:

```bash
# Application secrets (JWT, session, storage, OIDC HMAC) - 48 bytes base64
openssl rand -base64 48

# OIDC signing key (2048-bit RSA, PKCS#8 PEM)
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 \
  -out /config/certs/authelia.key

# OIDC client secrets must be stored hashed (Authelia hashes the presented value)
authelia crypto hash generate pbkdf2 --variant sha512 --password 'my-client-secret'
```

The plaintext client secret is configured in the client (Gitea/Grafana); the
`$pbkdf2-sha512$...` digest is what goes into `client_secret`.

> The `offline_access` scope is required whenever a client requests the
> `refresh_token` grant, otherwise Authelia logs a warning.

## Trust the LDAP CA

Authelia verifies the LDAPS server certificate. Because OpenLDAP uses a
self-signed certificate, copy it into the `certificates_directory`:

```bash
# On the Proxmox host
pct pull 102 /etc/ldap/certs/ldap.crt /root/ldap-ca.crt
pct push 103 /root/ldap-ca.crt /config/ca/ldap.crt
pct exec 103 -- chown -R authelia:authelia /config/ca
```

## Systemd Service

```bash
cat > /etc/systemd/system/authelia.service << 'EOF'
[Unit]
Description=Authelia authentication and authorization server
Documentation=https://www.authelia.com
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=authelia
Group=authelia
ExecStart=/usr/local/bin/authelia --config /etc/authelia/configuration.yml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now authelia
```

Always validate before (re)starting:

```bash
su -s /bin/bash -c \
  'authelia --config /etc/authelia/configuration.yml config validate' authelia
```

## Nginx Integration

Nginx (LXC 101) proxies public traffic and calls Authelia's `/api/verify`
endpoint via `auth_request`. See `docs/05-services/nginx.md` for the full vhosts.

Two details are essential when Nginx sits behind the Cloudflare Tunnel:

1. The tunnel terminates TLS and forwards `X-Forwarded-Proto: https`, but Nginx
   only sees plain HTTP. A `map` restores the real scheme.
2. Authelia refuses `http` target URLs, so `X-Original-URL` and the redirect must
   use the recovered scheme.

```nginx
# /etc/nginx/nginx.conf (http context)
map $http_x_forwarded_proto $forwarded_scheme {
    default $http_x_forwarded_proto;
    ""      $scheme;
}
```

```nginx
# Example protected vhost
location / {
    auth_request /auth;
    auth_request_set $user $upstream_http_remote_user;
    auth_request_set $groups $upstream_http_remote_groups;
    error_page 401 =302 https://auth.example.com/?rd=$forwarded_scheme://$host$request_uri;
    proxy_pass http://10.0.0.40:3000;
    proxy_set_header X-Forwarded-Proto $forwarded_scheme;
}

location = /auth {
    internal;
    proxy_pass http://10.0.0.30:9091/api/verify;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_set_header X-Original-URL $forwarded_scheme://$host$request_uri;
    proxy_set_header X-Original-Method $request_method;
}
```

## Firewall

```bash
ufw allow from 10.0.0.10 to any port 9091 proto tcp
ufw reload
```

## Verification

```bash
# Health (inside the container)
curl -s http://10.0.0.30:9091/api/health
# {"status":"OK"}

# Health / discovery through Nginx + Cloudflare
curl -s https://auth.example.com/api/health
curl -s https://auth.example.com/.well-known/openid-configuration

# Protected service should redirect to the portal
curl -I https://git.example.com
# HTTP/2 302 ... location: https://auth.example.com/?rd=https://git.example.com/
```

## Backup

```bash
cp /var/lib/authelia/db.sqlite3 /backup/authelia-$(date +%F).sqlite3
```

## References

- [Authelia Docs](https://www.authelia.com/configuration/prologue/introduction/)
- [OpenID Connect Provider](https://www.authelia.com/configuration/identity-providers/openid-connect/provider/)
- [OIDC Spec](https://openid.net/specs/openid-connect-core-1_0.html)
