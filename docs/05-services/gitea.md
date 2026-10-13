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
apt update && apt install -y git sqlite3 ca-certificates

# Download the Gitea binary (version.json lists the latest release)
VER=28.1.0
wget -O /usr/local/bin/gitea https://dl.gitea.com/gitea/${VER}/gitea-${VER}-linux-amd64
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
WORK_PATH = /var/lib/gitea
PROTOCOL = http
DOMAIN = git.example.com
ROOT_URL = https://git.example.com/
HTTP_ADDR = 0.0.0.0
HTTP_PORT = 3000
DISABLE_SSH = true
OFFLINE_MODE = true
LFS_START_SERVER = true
LFS_JWT_SECRET = GENERATE_WITH_openssl_rand_base64_32

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

[service]
DISABLE_REGISTRATION = true
REQUIRE_SIGNIN_VIEW = true
REGISTER_EMAIL_CONFIRM = false
ENABLE_NOTIFY_MAIL = false
DEFAULT_KEEP_EMAIL_PRIVATE = false

[oauth2]
ENABLED = true
JWT_SECRET = GENERATE_WITH_openssl_rand_base64_32

[oauth2_client]
ENABLE_AUTO_REGISTRATION = true
ACCOUNT_LINKING = auto
USERNAME = preferred_username

[session]
PROVIDER = file
PROVIDER_CONFIG = /var/lib/gitea/data/sessions

[cache]
ADAPTER = memory

[mailer]
ENABLED = false
```

Notes:

- `DISABLE_SSH = true` and only the internal Nginx may reach `:3000`, so Git is
  served over **HTTPS only** (authenticate with an app token).
- `REQUIRE_SIGNIN_VIEW = true` makes the instance private; the `/api/health`
  probe returns the sign-in page instead of JSON, so check the port/HTTP status
  rather than that endpoint.
- `[oauth2_client]` enables first-login auto-registration and account linking; see
  [OIDC Configuration](../04-identity/oidc.md).

## OIDC Configuration (Authelia)

Gitea is registered as an OIDC client in Authelia (`client_id: gitea`), and the
OAuth2 source is added on the Gitea side. Both can be created from the CLI.

```bash
# Run as the git user (Gitea refuses to run admin commands as root)
su -s /bin/bash git -c \
  'gitea -w /var/lib/gitea -c /etc/gitea/app.ini admin auth add-oauth \
     --name oidc \
     --provider openidConnect \
     --key gitea \
     --secret "<PLAINTEXT_CLIENT_SECRET>" \
     --auto-discover-url https://auth.example.com/.well-known/openid-configuration \
     --scopes openid --scopes profile --scopes email --scopes groups --scopes offline_access \
     --group-claim-name groups \
     --admin-group admins \
     --skip-local-2fa'
```

Key points:

- The source **name** (`oidc`) must match the `redirect_uri` registered in
  Authelia: `https://git.example.com/user/oauth2/oidc/callback`.
- `--admin-group admins` maps Authelia's `groups` claim to Gitea administrators.
- `--skip-local-2fa` defers two-factor to Authelia (already required there).
- `offline_access` is required because the client requests refresh tokens.

Create the first administrator before the first SSO login (a pre-existing local
account with the same email is linked automatically on first OIDC login):

```bash
su -s /bin/bash git -c \
  'gitea -w /var/lib/gitea -c /etc/gitea/app.ini admin user create \
     --username admin --email admin@example.com --admin \
     --random-password --must-change-password=false'
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

## GitHub Mirror (Push Mirror) — Optional

> **Not enabled in this build.** Publishing the documentation to GitHub Pages is
> done by pushing directly from the workstation; a Gitea push mirror is an
> optional addition.

1. **Create a GitHub Personal Access Token:**
   - Settings -> Developer settings -> Personal access tokens -> Fine-grained
   - Repository access: the target repo only; Permissions: `Contents: Read and write`
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

## SSH Access (Not used)

This build serves Git over **HTTPS only** (`DISABLE_SSH = true`), so no SSH keys
are needed for clients. (If the optional push mirror were enabled, an outbound
deploy key for GitHub would be generated here.)

## Firewall

```bash
ufw allow from 10.0.0.10 to any port 3000 proto tcp
ufw reload
```

## Backup

```bash
# Gitea built-in dump (run as the git user, with explicit work dir/config)
su -s /bin/bash git -c \
  'gitea -w /var/lib/gitea -c /etc/gitea/app.ini dump -o /var/lib/gitea/data/backup'

# Or via UI: Admin Panel -> Administration -> Backup
```

## Verification

```bash
# Health check (any 200 HTML is fine; REQUIRE_SIGNIN_VIEW makes /api/health
# return the sign-in page rather than JSON)
curl -s -o /dev/null -w '%{http_code}\n' http://10.0.0.40:3000/

# Via Nginx/Cloudflare
curl -I https://git.example.com

# Test OIDC login
# Visit https://git.example.com -> "Sign In with OIDC" -> Authelia
```

## References

- [Gitea Docs](https://docs.gitea.io/)
- [Gitea Config Cheat Sheet](https://docs.gitea.io/en-us/config-cheat-sheet/)
- [OIDC Setup](https://docs.gitea.io/en-us/oauth2-provider/)