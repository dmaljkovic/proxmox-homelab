# Project Sanitization Rules

## CRITICAL: Never commit these to GitHub

### Forbidden Patterns
| Pattern | Replace With | Context |
|---------|--------------|---------|
| `example.com` | `example.com` | Public domain |
| `10.0.0.x` | `10.0.0.x` | Private network IPs |
| `10.0.0.0/24` | `10.0.0.0/24` | Private network CIDR |
| `10.0.0.1` | `10.0.0.1` | Gateway IP |
| `10.0.0.100` | `10.0.0.10` | DNS IP |
| `10.0.0.99` | `10.0.0.100` | Proxmox host IP |
| `10.0.0.10` | `10.0.0.10` | Nginx LXC IP |
| `10.0.0.20` | `10.0.0.20` | OpenLDAP LXC IP |
| `10.0.0.30` | `10.0.0.30` | Authelia LXC IP |
| `10.0.0.40` | `10.0.0.40` | Gitea LXC IP |
| `10.0.0.50` | `10.0.0.50` | Grafana LXC IP |
| `10.0.0.60` | `10.0.0.60` | Prometheus LXC IP |
| `10.0.0.70` | `10.0.0.70` | Security LXC IP |

### Allowed (Internal Only)
| Pattern | Keep As | Reason |
|---------|---------|--------|
| `dc=unseen-uni,dc=xyz` | Keep | LDAP base DN (internal identifier) |
| `admin-password` | Keep | Placeholder password |
| `<TUNNEL_ID>` | Keep | Placeholder for Cloudflare Tunnel UUID |

### Hardware (Sanitize)
| Original | Replace With |
|----------|--------------|
| Generic Laptop / Generic Laptop-15IKBN | Generic Laptop |
| Intel i7 (4C/8T) / Intel Intel i7 (4C/8T) | Intel i7 (4C/8T) |

### GitHub/User Identity
| Original | Replace With |
|----------|--------------|
| yourusername | yourusername |

---

## Pre-Commit Check

Run this before every commit:
```bash
#!/bin/bash
# Check for forbidden patterns
forbidden=(
  "unseen-uni\.xyz"
  "192\.168\.0\."
  "Generic Laptop"
  "Generic Laptop"
  "Intel i7 (4C/8T)"
  "yourusername"
)

for pattern in "${forbidden[@]}"; do
  if git diff --cached --name-only | xargs grep -l "$pattern" 2>/dev/null; then
    echo "ERROR: Forbidden pattern '$pattern' found in staged files"
    exit 1
  fi
done
```

---

## Quick Sanitization Commands

```bash
# Replace all occurrences in docs/
find docs -name "*.md" -exec sed -i \
  -e 's/unseen-uni\.xyz/example.com/g' \
  -e 's/192\.168\.0\./10.0.0./g' \
  -e 's/192\.168\.0\//10.0.0./g' \
  -e 's/Generic Laptop/Generic Laptop/g' \
  -e 's/Generic Laptop/Generic Laptop/g' \
  -e 's/Intel i7 (4C/8T)/Intel i7 (4C\/8T)/g' \
  -e 's/yourusername/yourusername/g' \
  {} \;

# Verify clean
grep -r "unseen-uni\|192.168.0\." docs/ || echo "Clean!"
```