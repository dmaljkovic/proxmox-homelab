# Ansible Automation

## Layout

```
ansible/
├── ansible.cfg                        # inventory path, roles path, pipelining
├── inventory/
│   ├── hosts.yml                      # sanitized, committed (placeholders)
│   ├── hosts.local.yml                # real values, gitignored
│   └── group_vars/
│       └── all/
│           ├── main.yml               # public defaults (versions, timezone)
│           └── secrets.local.yml      # real secrets, gitignored
├── playbooks/
│   └── site.yml                       # the single entry point
└── roles/
    ├── base/                          # packages, timezone, unattended-upgrades
    ├── node_exporter/                 # node_exporter on every host
    ├── nginx/                         # reverse proxy + vhosts
    ├── prometheus/                    # server + scrape config
    └── grafana/                       # server + datasource + dashboards
```

## Prerequisites

On the control node, install Ansible into a virtual environment (no system-wide
install needed):

```bash
python3 -m venv ~/.venvs/ansible
~/.venvs/ansible/bin/pip install ansible-core
export PATH="$HOME/.venvs/ansible/bin:$PATH"
```

Ensure your SSH public key is authorized on the Proxmox host and every LXC
(`ssh root@10.0.0.99` works, and key auth to each container works). Copy the
sanitized inventory and fill in your own values:

```bash
cp ansible/inventory/hosts.yml ansible/inventory/hosts.local.yml
```

## Inventory

`ansible/inventory/hosts.local.yml` (real values, gitignored):

```yaml
all:
  vars:
    ansible_user: root
    ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
    domain: example.com
    tunnel_id: "<TUNNEL_ID_PLACEHOLDER>"

  children:
    proxmox:
      hosts:
        proxmox-host:
          ansible_host: 10.0.0.99
    lxc_containers:
      children:
        nginx:
          hosts:
            nginx:
              ansible_host: 10.0.0.10
        openldap:
          hosts:
            openldap:
              ansible_host: 10.0.0.20
        authelia:
          hosts:
            authelia:
              ansible_host: 10.0.0.30
        gitea:
          hosts:
            gitea:
              ansible_host: 10.0.0.40
        grafana:
          hosts:
            grafana:
              ansible_host: 10.0.0.50
        prometheus:
          hosts:
            prometheus:
              ansible_host: 10.0.0.60
```

`ansible/inventory/group_vars/all/main.yml` holds public defaults (component
versions, retention, timezone) and the **ordered** `node_exporters` list that is
rendered into the Prometheus scrape config. Secrets are overridden in
`secrets.local.yml`:

```yaml
grafana_admin_password: "<...>"
grafana_secret_key: "<...>"
grafana_oidc_client_secret: "<...>"
gitea_metrics_token: "<...>"
```

`.gitignore` excludes `*.local.yml`, `ansible/vault/`, and `*.vault.yml`, so real
secrets never reach the repository. For a shared/production setup, encrypt the
secrets file with `ansible-vault` instead.

## Roles

| Role | Applies to | What it does |
| --- | --- | --- |
| `base` | all | apt cache, baseline packages (curl, gnupg, rsync, unattended-upgrades), timezone, unattended security upgrades |
| `node_exporter` | all | node_exporter binary + systemd unit on the host and every LXC |
| `nginx` | `nginx` | `nginx.conf`, vhosts (auth, git, grafana), local `stub_status` |
| `prometheus` | `prometheus` | binaries, `prometheus.yml` (validated with `promtool`), systemd unit |
| `grafana` | `grafana` | apt repo, server env overrides (OIDC), datasource + dashboard provisioning |

> **Scope:** the playbook automates the core stack — `base`, `node_exporter`,
> `nginx`, `prometheus`, and `grafana`. The `openldap`, `authelia`, `gitea`, and
> `proxmox-host` roles are intentionally deferred (they hold the identity-stack
> secrets and are higher-risk to re-apply). The `grafana` role already renders its
> OIDC client secret and the Prometheus scrape config from inventory variables.

## Running

From the `ansible/` directory:

```bash
export PATH="$HOME/.venvs/ansible/bin:$PATH"

# Validate syntax
ansible-playbook playbooks/site.yml --syntax-check

# Dry run — show what would change, change nothing
ansible-playbook playbooks/site.yml --check --diff

# Apply everything
ansible-playbook playbooks/site.yml

# Apply a single host
ansible-playbook playbooks/site.yml --limit grafana
```

## Secrets & the Grafana dashboards

The Grafana role downloads the community dashboards (Node Exporter Full, Prometheus
Overview) into a **scratch** directory and copies them into the provisioned path
only on first install, then rewrites the datasource placeholders to the
provisioned Prometheus UID. This keeps repeated runs from re-downloading and
clobbering the patched files. Because the dashboard provider reloads on an
interval, a restart is only triggered on an actual config change.

## Idempotency

The playbook is designed so a second run reports no changes (aside from the apt
cache, which refreshes hourly by `cache_valid_time`):

```bash
ansible-playbook playbooks/site.yml   # run once
ansible-playbook playbooks/site.yml   # expect changed=0 on all hosts
```

## References

- [Ansible Documentation](https://docs.ansible.com/)
- [ansible-core](https://github.com/ansible/ansible)
- [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
