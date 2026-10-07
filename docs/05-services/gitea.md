# Gitea Configuration

## Overview

Gitea is a lightweight, self-hosted Git service written in Go. It provides Git hosting with a web UI, issue tracking, pull requests, and OAuth2/OIDC authentication.

## Deployment

- **Container**: LXC 104
- **IP**: 10.0.0.40
- **RAM**: 768 MB
- **CPU**: 2 cores
- **Disk**: 8 GB
- **OS**: Debian 12 (Bookworm)
- **Database**: SQLite (embedded)

## Installation

```bash
# On Proxmox host - enter container
pct enter 104

# Inside container
apt update && apt install -y git sqlite3

# Download Gitea binary (latest)
wget -O /usr/local/bin/gitea https://dl.gitea.io/gitea/1.22.0/gitea-1.22.0-linux-amd64
chmod +x /usr/local/bin/gitea

# Create git user
adduser --system --shell /bin/bash --gecos 'Git Version Control' --group --disabled-password --home /home/git git

# Create directories
mkdir -p /var/lib/gitea/{custom,data,log}
chown -R git:git /var/lib/gitea/
chmod -R 750 /var/lib/gitea/

mkdir -p /etc/gitea
chown root:git /etc/gitea
chmod 770 /etc/gitea
```

## Configuration (`/etc/gitea/app.ini`)

```ini
[server]
APP_NAME = Gitea
RUN_USER = git
RUN_MODE = prod
DOMAIN = git.example.com
ROOT_URL = https://git.example.com/
SSH_DOMAIN = git.example.com
SSH_PORT = 22
LFS_START_SERVER = true
LFS_CONTENT_PATH = /var/lib/gitea/data/lfs
OFFLINE_MODE = false

[database]
DB_TYPE = sqlite3
PATH = /var/lib/gitea/data/gitea.db

[repository]
ROOT = /var/lib/gitea/data/gitea-repositories

[log]
MODE = file
LEVEL = info
ROOT_PATH = /var/lib/gitea/log

[security]
INSTALL_LOCK = true
SECRET_KEY = GENERATE_WITH_openssl_rand_base64_32
INTERNAL_TOKEN = GENERATE_WITH_openssl_rand_base64_32
PASSWORD_HASH_ALGO = pbkdf2
LOGIN_REMEMBER_DAYS = 7
COOKIE_USERNAME = gitea_awesome
COOKIE_REMEMBER_NAME = gitea_incredible

[oauth2]
ENABLED = true
JWT_SECRET = GENERATE_WITH_openssl_rand_base64_32

[auth.openid]
ENABLED = true

[service]
DISABLE_REGISTRATION = false
REQUIRE_SIGNIN_VIEW = true
DEFAULT_KEEP_EMAIL_PRIVATE = false
ENABLE_NOTIFY_MAIL = false

[mailer]
ENABLED = false

[session]
PROVIDER = file
PROVIDER_CONFIG = /var/lib/gitea/data/sessions

[cache]
ADAPTER = memory

[indexer]
ISSUE_INDEXER_TYPE = bleve
```

## OIDC Configuration (Authelia)

```ini
[oauth2]
ENABLED = true

[auth.openid]
ENABLED = true

# In Gitea UI: Admin Panel -> Authentication -> OAuth2 -> Add Provider
# Name: Authelia
# Client ID: gitea
# Client Secret: (from Authelia config)
# Redirect URI: https://git.example.com/user/oauth2/oidc/callback
# OpenID Connect Discovery URL: https://auth.example.com/.well-known/openid-configuration
# Scopes: openid profile email groups
# Group Attribute Path: groups
# Admin Group: admins
```

## Systemd Service

```bash
cat > /etc/systemd/system/gitea.service << 'EOF'
[Unit]
Description=Gitea (Git with a cup of tea)
After=syslog.target
After=network.target

[Service]
RestartSec=2s
Type=simple
User=git
Group=git
WorkingDirectory=/home/git
ExecStart=/usr/local/bin/gitea web --config /etc/gitea/app.ini
Restart=always
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now gitea
systemctl status gitea
```

## GitHub Mirror (Push Mirror)

1. **Create GitHub Personal Access Token:**
   - Settings -> Developer settings -> Personal access tokens -> Classic
   - Scopes: `repo` (full control of private repos)
   - Copy token

2. **Configure Push Mirror in Gitea:**
   - Admin Panel -> Repositories -> Mirrors -> Add Push Mirror
   - Remote URL: `https://github.com/username/repo.git`
   - Authentication: `username:PAT`
   - Sync on push: Enabled

3. **Test:**
   ```bash
   # Push to Gitea
   git push origin main
   # Verify on GitHub
   ```

## SSH Access (Optional)

```bash
# Generate SSH key for Gitea
ssh-keygen -t ed25519 -f /home/git/.ssh/id_ed25519 -N ""

# Add public key to GitHub Deploy Keys
cat /home/git/.ssh/id_ed25519.pub
```

## Firewall

```bash
ufw allow from 10.0.0.10 to any port 3000 proto tcp
ufw allow from 10.0.0.10 to any port 22 proto tcp
ufw reload
```

## Backup

```bash
# Gitea built-in dump
gitea dump -c /etc/gitea/app.ini -o /backup/gitea-$(date +%F).zip

# Or via UI: Admin Panel -> Administration -> Backup
```

## Verification

```bash
# Health check
curl -s http://10.0.0.40:3000/healthz

# Via Nginx/Cloudflare
curl -I https://git.example.com

# Test OIDC login
# Visit https://git.example.com -> should redirect to Authelia
```

## References

- [Gitea Docs](https://docs.gitea.io/)
- [Gitea Config Cheat Sheet](https://docs.gitea.io/en-us/config-cheat-sheet/)
- [OIDC Setup](https://docs.gitea.io/en-us/oauth2-provider/)