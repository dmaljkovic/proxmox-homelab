# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| Latest (main branch) | ✅ |

## Reporting a Vulnerability

If you discover a security vulnerability in this homelab configuration, please report it by:

1. **Do not create a public issue**
2. Email: security@example.com (or create a private GitHub Security Advisory)
3. Include:
   - Description of the vulnerability
   - Steps to reproduce
   - Potential impact
   - Suggested fix (if any)

## Security Architecture

### Network Segmentation
- **Internet**: No direct access to any service
- **Cloudflare Edge**: TLS termination, WAF, DDoS protection
- **Cloudflare Tunnel**: Encrypted outbound-only connection
- **Proxmox Host**: Management only (SSH/GUI from LAN)
- **vmbr0 (10.0.0.0/24)**: Internal service network
- **LXC Containers**: Isolated services with minimal exposure

### Defense in Depth

1. **Perimeter**: Cloudflare WAF + DDoS
2. **Transport**: TLS 1.2+ everywhere (Cloudflare edge)
3. **Authentication**: Authelia SSO + MFA (TOTP)
4. **Authorization**: LDAP groups → Authelia policies → OIDC claims
5. **Network**: UFW (host) + iptables (LXCs) - default deny
6. **Host**: SSH keys only, fail2ban, auto-updates
7. **Application**: Security headers, rate limiting, CSP
8. **Secrets**: SOPS + age encryption, no plaintext secrets

### Key Security Controls

| Control | Implementation |
|---------|----------------|
| No public ports | Cloudflare Tunnel (outbound only) |
| SSH | Key-only, root prohibited, fail2ban |
| Web auth | Authelia OIDC + TOTP MFA |
| Internal auth | LDAPS (636) only |
| Reverse proxy | Nginx auth_request → Authelia |
| Headers | CSP, HSTS, X-Frame-Options, etc. |
| Rate limiting | Nginx + Cloudflare |
| Secrets | SOPS/age (encrypted in repo) |
| Updates | unattended-upgrades (security) |
| Monitoring | Prometheus + Grafana alerts |
| Backups | Encrypted, off-site (GitHub mirror) |

## Threat Model

### In Scope
- Unauthorized access to services
- Data exfiltration from Gitea/Grafana
- Credential theft via phishing
- Container escape
- Supply chain attacks

### Out of Scope
- Physical access to laptop
- Cloudflare infrastructure compromise
- ISP-level traffic interception (encrypted tunnel)
- Zero-day exploits in upstream software

## Hardening Checklist

- [ ] Proxmox: SSH key-only, UFW, fail2ban, auto-updates
- [ ] Nginx: Security headers, rate limiting, auth_request
- [ ] Authelia: MFA required, session hardening, regulation
- [ ] OpenLDAP: LDAPS only, ppolicy, ACLs
- [ ] Gitea: OIDC only, no local auth, signed commits
- [ ] Grafana: OIDC only, no admin UI exposure
- [ ] Prometheus: No external scrape, internal only
- [ ] LXCs: Individual firewalls, minimal packages
- [ ] Secrets: All encrypted with SOPS/age
- [ ] Backups: Tested restore, encrypted

## Compliance Notes

This is a personal homelab project demonstrating security practices. Not certified for any compliance framework (SOC2, ISO27001, etc.).

## Contact

Security issues: Create a GitHub Security Advisory or email security@example.com
