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
- **Version**: Grafana 13.2.3
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

## Configuration

The Debian package ships a systemd unit that starts `grafana server` with
`--config /etc/grafana/grafana.ini` and loads environment variables from
`/etc/default/grafana-server`. Rather than editing `grafana.ini`, homelab
settings are applied as `GF_*` overrides appended to that environment file. This
keeps the packaged defaults intact and makes the custom configuration easy to
review.

`/etc/default/grafana-server`:

```bash
# homelab overrides
GF_SERVER_DOMAIN=grafana.example.com
GF_SERVER_ROOT_URL=https://grafana.example.com/
GF_SERVER_SERVE_FROM_SUB_PATH=false

GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=CHANGE_ME

GF_AUTH_GENERIC_OAUTH_ENABLED=true
GF_AUTH_GENERIC_OAUTH_NAME=Authelia
GF_AUTH_GENERIC_OAUTH_ALLOW_SIGN_UP=true
GF_AUTH_GENERIC_OAUTH_CLIENT_ID=grafana
GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET=CHANGE_ME
GF_AUTH_GENERIC_OAUTH_SCOPES=openid profile email groups offline_access
GF_AUTH_GENERIC_OAUTH_AUTH_URL=https://auth.example.com/api/oidc/authorization
GF_AUTH_GENERIC_OAUTH_TOKEN_URL=https://auth.example.com/api/oidc/token
GF_AUTH_GENERIC_OAUTH_API_URL=https://auth.example.com/api/oidc/userinfo
GF_AUTH_GENERIC_OAUTH_USE_REFRESH_TOKEN=true
GF_AUTH_GENERIC_OAUTH_LOGIN_ATTRIBUTE_PATH=preferred_username
GF_AUTH_GENERIC_OAUTH_ROLE_ATTRIBUTE_PATH=contains(groups[*], 'admins') && 'Admin' || contains(groups[*], 'developers') && 'Editor' || 'Viewer'
```

After editing, restart the service:

```bash
systemctl restart grafana-server
```

> **Note:** `GF_SECURITY_ADMIN_PASSWORD` only takes effect on first
> initialisation. If Grafana already created the `admin` user, reset it with
> `grafana cli admin reset-admin-password "<new-password>"`.

## OIDC Configuration (Authelia)

Grafana authenticates against Authelia via the generic OAuth provider. The
`groups` scope is required so the `ROLE_ATTRIBUTE_PATH` expression can map LDAP
group membership to Grafana roles:

- `admins` → `Admin`
- `developers` → `Editor`
- otherwise → `Viewer`

`USE_REFRESH_TOKEN=true` is paired with the `offline_access` scope so Grafana
can refresh tokens without re-prompting.

## Authelia OIDC Client Configuration

In Authelia `configuration.yml`:

```yaml
identity_providers:
  oidc:
    clients:
      - client_id: grafana
        client_name: Grafana
        client_secret: "$pbkdf2-sha512$..."
        redirect_uris:
          - https://grafana.example.com/login/generic_oauth
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
        public: false
        require_pkce: true
        pkce_challenge_method: S256
```

Generate the hashed secret and store the plaintext in Grafana's
`GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET`:

```bash
su -s /bin/bash -c '/usr/local/bin/authelia crypto hash generate pbkdf2 --password=<secret>' authelia
```

## Provisioning

Provisioning makes the Prometheus data source and dashboards available on a
fresh Grafana instance without manual import.

### Data Source (`/etc/grafana/provisioning/datasources/prometheus.yaml`)

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    uid: prometheus
    type: prometheus
    access: proxy
    url: http://10.0.0.60:9090
    isDefault: true
    editable: false
```

The `uid` is fixed so provisioned dashboards can reference it reliably.

### Dashboards (`/etc/grafana/provisioning/dashboards/homelab.yaml`)

```yaml
apiVersion: 1

providers:
  - name: homelab
    orgId: 1
    folder: Homelab
    folderUid: homelab
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
```

Dashboard JSON files are placed in `/var/lib/grafana/dashboards/`. When
importing community dashboards, replace any `${DS_...}` datasource placeholders
with `prometheus` (the provisioned uid) and remove the `__inputs` block, since
provisioning does not resolve dashboard `__inputs`.

## Metrics

Grafana exposes Prometheus metrics on its HTTP port at `/metrics`, scraped by
the `grafana` job in the [Prometheus](prometheus.md) configuration:

```yaml
- job_name: grafana
  static_configs:
    - targets: ['10.0.0.50:3000']
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

# OIDC redirect (expect 302 to Authelia)
curl -sI https://grafana.example.com/login/generic_oauth

# Data source health (admin credentials)
curl -s -u admin:<password> \
  http://10.0.0.50:3000/api/datasources/uid/prometheus/health
```

Then visit https://grafana.example.com, click **Sign in with Authelia**, and
confirm dashboards render live data.

## Dashboards

| Dashboard ID | Name | Provisioned file |
|--------------|------|------------------|
| 1860 | Node Exporter Full | `/var/lib/grafana/dashboards/node-exporter-full.json` |
| 3662 | Prometheus 2.0 Overview | `/var/lib/grafana/dashboards/prometheus-overview.json` |

Download from grafana.com and adapt the datasource placeholder:

```bash
curl -fsSL https://grafana.com/api/dashboards/1860/revisions/latest/download \
  -o node-exporter-full.json
# then replace "${ds_prometheus}" with "prometheus" and set "id": null
```

## Backup

```bash
# SQLite database
cp /var/lib/grafana/grafana.db /backup/grafana-$(date +%F).db

# Dashboards (JSON)
grafana cli admin export-dashboard <uid> > /backup/dashboard-<uid>.json
```

## References

- [Grafana Docs](https://grafana.com/docs/)
- [Generic OAuth](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/generic-oauth/)
- [Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
