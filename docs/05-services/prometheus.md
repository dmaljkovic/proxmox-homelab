# Prometheus Configuration

## Overview

Prometheus is the metrics collection and time-series database for the homelab.
It scrapes **node_exporter** on every LXC and on the Proxmox host, plus
service-specific exporters (Nginx, Gitea, Authelia). Grafana then reads
Prometheus as its default data source.

Prometheus is **internal-only** - it is not published through the Cloudflare
Tunnel. Reach the UI through an SSH tunnel (see [Access](#access)).

## Deployment

- **Container**: LXC 106
- **IP**: 10.0.0.60
- **RAM**: 512 MB
- **CPU**: 1 core
- **Disk**: 8 GB
- **OS**: Debian 12 (Bookworm)
- **Version**: Prometheus 3.15.0
- **Retention**: 15 days
- **Listen**: `0.0.0.0:9090`

## Installation

```bash
# On Proxmox host
VER=3.15.0
curl -fsSL -O https://github.com/prometheus/prometheus/releases/download/v${VER}/prometheus-${VER}.linux-amd64.tar.gz
tar -xzf prometheus-${VER}.linux-amd64.tar.gz

# Create user and directories inside LXC 106
pct exec 106 -- bash -c '
  id prometheus >/dev/null 2>&1 || useradd --system --no-create-home --shell /usr/sbin/nologin prometheus
  mkdir -p /etc/prometheus /var/lib/prometheus
  chown -R prometheus:prometheus /var/lib/prometheus'

# Copy the binaries and config into the container
pct push 106 prometheus-${VER}.linux-amd64/prometheus /usr/local/bin/prometheus --perms 0755
pct push 106 prometheus-${VER}.linux-amd64/promtool   /usr/local/bin/promtool   --perms 0755
pct push 106 prometheus.yml /etc/prometheus/prometheus.yml --perms 0644
```

## Configuration (`/etc/prometheus/prometheus.yml`)

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: homelab

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: node
    static_configs:
      - targets:
          - 10.0.0.99:9100   # Proxmox host
          - 10.0.0.10:9100   # nginx
          - 10.0.0.20:9100   # openldap
          - 10.0.0.30:9100   # authelia
          - 10.0.0.40:9100   # gitea
          - 10.0.0.50:9100   # grafana
          - 10.0.0.60:9100   # prometheus
        labels:
          env: homelab

  - job_name: nginx
    static_configs:
      - targets: ['10.0.0.10:9113']

  - job_name: gitea
    metrics_path: /metrics
    authorization:
      credentials: <GITEA_METRICS_TOKEN>
    static_configs:
      - targets: ['10.0.0.40:3000']

  - job_name: authelia
    static_configs:
      - targets: ['10.0.0.30:9959']

  - job_name: grafana
    static_configs:
      - targets: ['10.0.0.50:3000']
```

## Systemd Service (`/etc/systemd/system/prometheus.service`)

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=15d \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

`--web.enable-lifecycle` allows a config reload without a restart:

```bash
curl -X POST http://10.0.0.60:9090/-/reload
```

## Exporters

### node_exporter (host + every LXC)

[node_exporter](https://github.com/prometheus/node_exporter) v1.12.1 exposes
host metrics on `:9100`. Install it on the Proxmox host and push the binary into
each container:

```bash
# On Proxmox host
VER=1.12.1
curl -fsSL -O https://github.com/prometheus/node_exporter/releases/download/v${VER}/node_exporter-${VER}.linux-amd64.tar.gz
tar -xzf node_exporter-${VER}.linux-amd64.tar.gz
install -m 0755 node_exporter-${VER}.linux-amd64/node_exporter /usr/local/bin/node_exporter
id node_exporter >/dev/null 2>&1 || useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter

# Push into each container (the unit file lives at /etc/systemd/system/node_exporter.service)
for id in 101 102 103 104 105 106; do
  pct push $id node_exporter-${VER}.linux-amd64/node_exporter /usr/local/bin/node_exporter --perms 0755
  pct push $id /etc/systemd/system/node_exporter.service /etc/systemd/system/node_exporter.service --perms 0644
  pct exec $id -- bash -c 'id node_exporter >/dev/null 2>&1 || useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter; systemctl daemon-reload; systemctl enable --now node_exporter'
done
```

`node_exporter.service`:

```ini
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

### Nginx (LXC 101, :9113)

`stub_status` provides the raw numbers; `nginx-prometheus-exporter` translates
them to Prometheus format. Add a local-only `stub_status` server:

```nginx
# /etc/nginx/sites-available/stub_status
server {
    listen 127.0.0.1:8080;
    server_name _;
    access_log off;
    location = /stub_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
    location / { return 404; }
}
```

Then install `nginx-prometheus-exporter` v1.5.3:

```bash
curl -fsSL -o /tmp/nginx-exporter.tar.gz \
  https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v1.5.3/nginx-prometheus-exporter_1.5.3_linux_amd64.tar.gz
tar -xzf /tmp/nginx-exporter.tar.gz -C /tmp
pct push 101 /tmp/nginx-prometheus-exporter /usr/local/bin/nginx-prometheus-exporter --perms 0755
pct exec 101 -- bash -c '
  id nginx-prometheus-exporter >/dev/null 2>&1 || useradd --system --no-create-home --shell /usr/sbin/nologin nginx-prometheus-exporter
  systemctl daemon-reload; systemctl enable --now nginx-prometheus-exporter'
```

`nginx-prometheus-exporter.service`:

```ini
[Unit]
Description=NGINX Prometheus Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=nginx-prometheus-exporter
Group=nginx-prometheus-exporter
Type=simple
ExecStart=/usr/local/bin/nginx-prometheus-exporter \
  --nginx.scrape-uri=http://127.0.0.1:8080/stub_status \
  --web.listen-address=:9113
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

### Gitea (LXC 104, :3000/metrics)

Gitea exposes `/metrics` once enabled. Protect it with a bearer token so only
Prometheus can scrape it:

```ini
# /etc/gitea/app.ini
[metrics]
ENABLED = true
TOKEN = <GITEA_METRICS_TOKEN>
ALLOWED_IPS = 10.0.0.60
```

```bash
pct exec 104 -- systemctl restart gitea
```

Prometheus sends the token via `authorization.credentials` (see the `gitea`
job above). A request without the token is rejected with `401`.

### Authelia (LXC 103, :9959)

Enable the telemetry metrics listener:

```yaml
# /etc/authelia/configuration.yml
telemetry:
  metrics:
    enabled: true
    address: tcp://0.0.0.0:9959
```

```bash
pct exec 103 -- su -s /bin/bash -c \
  '/usr/local/bin/authelia --config /etc/authelia/configuration.yml config validate' authelia
pct exec 103 -- systemctl restart authelia
```

At rest the endpoint exposes Go/process metrics; the `authelia_*` counters
appear once there is authentication activity.

> **OpenLDAP exporter is not deployed.** The commonly used
> `tomcz/ldap_exporter` upstream is no longer available; LDAP-specific metrics
> are a future addition. LDAP is still covered by node_exporter at the OS level.

## Access

Prometheus is intentionally **not** exposed publicly. Open a temporary SSH
tunnel from your workstation:

```bash
ssh -L 9090:10.0.0.60:9090 root@10.0.0.99
# then browse http://localhost:9090
```

## Firewall

- **Inbound on the Proxmox host**: allow `9100/tcp` from the LXC subnet so the
  host's `node_exporter` is scrapeable.

  ```bash
  ufw allow from 10.0.0.0/24 to any port 9100 proto tcp comment 'node_exporter'
  ```

- **Per-LXC**: no extra rules are required (the containers have no host
  firewall); the metrics ports stay on the internal bridge only.

## Verification

```bash
# Validate the config before (re)starting
pct exec 106 -- /usr/local/bin/promtool check config /etc/prometheus/prometheus.yml

# Service state
pct exec 106 -- systemctl is-active prometheus

# All scrape targets (expect health "up")
curl -s 'http://10.0.0.60:9090/api/v1/targets?state=active' | \
  jq -r '.data.activeTargets[] | "\(.labels.job)\t\(.scrapeUrl)\t\(.health)"'

# Retention
curl -s http://10.0.0.60:9090/api/v1/status/flags | \
  jq -r '.data | "retention=\(."storage.tsdb.retention.time")"'
```

Expected jobs: `prometheus`, `node` (7 targets), `nginx`, `gitea`, `authelia`,
`grafana` — 12 scrape targets total.

## Backup

```bash
# Stop, snapshot the TSDB, restart
pct exec 106 -- systemctl stop prometheus
tar -czf /var/lib/vz/dump/prometheus-$(date +%F).tar.gz \
  -C /var/lib/prometheus .
pct exec 106 -- systemctl start prometheus
```

## References

- [Prometheus Docs](https://prometheus.io/docs/)
- [node_exporter](https://github.com/prometheus/node_exporter)
- [nginx-prometheus-exporter](https://github.com/nginxinc/nginx-prometheus-exporter)
- [Prometheus Config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
