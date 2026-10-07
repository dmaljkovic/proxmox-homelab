# Grafana Configuration

## Overview

Grafana is an open-source analytics and monitoring platform. It provides dashboards, alerting, and visualization for metrics from Prometheus and other data sources.

## Deployment

- **Container**: LXC 105
- **IP**: 10.0.0.50
- **RAM**: 512 MB
- **CPU**: 1 core
- **Disk**: 8 GB
- **OS**: Debian 12 (Bookworm)
- **Database**: SQLite (embedded)

## Installation

```bash
# On Proxmox host - enter container
pct enter 105

# Inside container
apt update && apt install -y curl gnupg2

# Add Grafana repository
curl -fsSL https://apt.grafana.com/gpg.key | gpg --dearmor -o /usr/share/keyrings/grafana.gpg
echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] https://apt.grafana.com stable main" > /etc/apt/sources.list.d/grafana.list

apt update && apt install -y grafana
```

## Configuration (`/etc/grafana/grafana.ini`)

```ini
[server]
domain = grafana.example.com
root_url = https://grafana.example.com/
serve_from_sub_path = false

[database]
type = sqlite3
path = /var/lib/grafana/grafana.db

[security]
admin_user = admin
admin_password = CHANGE_ME
secret_key = GENERATE_WITH_openssl_rand_base64_32
login_remember_days = 7
cookie_username = grafana_session
cookie_remember_name = grafana_remember
disable_gravatar = true

[auth]
disable_login_form = false
disable_signout_menu = false

[auth.generic_oauth]
enabled = true
name = Authelia
allow_sign_up = true
client_id = grafana
client_secret = grafana-secret
scopes = openid profile email
auth_url = https://auth.example.com/api/oidc/authorization
token_url = https://auth.example.com/api/oidc/token
api_url = https://auth.example.com/api/oidc/userinfo
allowed_domains = example.com
team_ids = 1
role_attribute_path = groups
allow_assign_grafana_admin = true

[session]
provider = file

[log]
mode = file
level = info

[paths]
data = /var/lib/grafana
logs = /var/log/grafana
plugins = /var/lib/grafana/plugins
provisioning = /etc/grafana/provisioning
```

## OIDC Configuration (Authelia)

In `/etc/grafana/grafana.ini`:

```ini
[auth.generic_oauth]
enabled = true
name = Authelia
allow_sign_up = true
client_id = grafana
client_secret = grafana-secret
scopes = openid profile email
auth_url = https://auth.example.com/api/oidc/authorization
token_url = https://auth.example.com/api/oidc/token
api_url = https://auth.example.com/api/oidc/userinfo
allowed_domains = example.com
team_ids = 1
role_attribute_path = groups
allow_assign_grafana_admin = true
```

## Authelia OIDC Client Configuration

In Authelia `configuration.yml`:

```yaml
identity_providers:
  oidc:
    clients:
      - client_id: grafana
        client_name: Grafana
        client_secret: "grafana-secret"
        redirect_uris:
          - https://grafana.example.com/login/generic_oauth
        scopes:
          - openid
          - profile
          - email
        grant_types:
          - authorization_code
          - refresh_token
        response_types:
          - code
        public: false
        require_pkce: true
        pkce_challenge_method: S256
```

## Systemd Service

```bash
systemctl daemon-reload
systemctl enable --now grafana-server
systemctl status grafana-server
```

## Provisioning (Optional)

### Data Sources (`/etc/grafana/provisioning/datasources/prometheus.yaml`)

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://10.0.0.60:9090
    isDefault: true
    editable: false
```

### Dashboards (`/etc/grafana/provisioning/dashboards/dashboards.yaml`)

```yaml
apiVersion: 1
providers:
  - name: 'Default'
    folder: 'Default'
    type: file
    options:
      path: /etc/grafana/provisioning/dashboards
```

## Firewall

```bash
ufw allow from 10.0.0.10 to any port 3000 proto tcp
ufw allow from 10.0.0.60 to any port 3000 proto tcp
ufw reload
```

## Verification

```bash
# Health check
curl -s http://10.0.0.50:3000/api/health

# Via Nginx/Cloudflare
curl -I https://grafana.example.com

# Test OIDC login
# Visit https://grafana.example.com -> should redirect to Authelia
```

## Dashboards to Import

| Dashboard ID | Name | Description |
|--------------|------|-------------|
| 1860 | Node Exporter Full | System metrics |
| 11156 | Prometheus 2.0 Stats | Prometheus self-monitoring |
| 15161 | Gitea Dashboard | Gitea metrics |
| 13978 | Authelia Dashboard | Authelia metrics |

Import via Grafana UI: **+** -> **Import** -> Enter Dashboard ID

## Backup

```bash
# SQLite database
cp /var/lib/grafana/grafana.db /backup/grafana-$(date +%F).db

# Dashboards (JSON)
grafana-cli admin export-dashboard --uid=UID > /backup/dashboard-UID.json
```

## References

- [Grafana Docs](https://grafana.com/docs/)
- [Generic OAuth](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/generic-oauth/)
- [Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
