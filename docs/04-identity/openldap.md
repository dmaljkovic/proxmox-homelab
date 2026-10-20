# OpenLDAP Configuration

## Overview

OpenLDAP serves as the centralized identity store for the homelab. It provides:

- User authentication backend for Authelia
- Group-based authorization
- LDAPS (LDAP over TLS) on port 636
- Standard RFC 2307 schema (posixAccount, posixGroup)

## Deployment

- **Container**: LXC 102
- **IP**: 10.0.0.20
- **RAM**: 256 MB
- **Disk**: 4 GB
- **OS**: Debian 12 (Bookworm)

## Installation

```bash
# On Proxmox host - enter container
pct enter 102

# Inside container
apt update && apt install -y slapd ldap-utils
```

### Configuration via dpkg-reconfigure

```bash
dpkg-reconfigure slapd

# Settings:
# - Omit OpenLDAP server configuration? -> No
# - DNS domain name: example.com
# - Organization name: Homelab
# - Admin password: (strong password)
# - Database backend: MDB
# - Remove database when purged? -> No
# - Move old database? -> Yes
# - Allow LDAPv2? -> No
```

## Base DN Structure

```
dc=example,dc=com
├── ou=people
│   └── uid=admin (uid=1000)
└── ou=groups
    ├── cn=developers (gid=1000)
    │   └── memberUid: admin
    └── cn=admins (gid=1001)
        └── memberUid: admin
```

## LDAPS (TLS) Configuration

### Generate Certificate

```bash
mkdir -p /etc/ldap/certs

cat > /etc/ldap/certs/openssl.cnf << 'EOF'
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = openldap.example.com

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = openldap.example.com
DNS.2 = ldap.example.com
IP.1 = 127.0.0.1
IP.2 = 10.0.0.20
EOF

openssl req -new -x509 -nodes -out /etc/ldap/certs/ldap.crt \
  -keyout /etc/ldap/certs/ldap.key \
  -days 3650 -config /etc/ldap/certs/openssl.cnf \
  -extensions v3_req -subj "/CN=openldap.example.com"

chown openldap:openldap /etc/ldap/certs/ldap.*
chmod 640 /etc/ldap/certs/ldap.key
```

### Configure TLS in slapd

```bash
cat > /etc/ldap/tls.ldif << 'EOF'
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ldap/certs/ldap.crt
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ldap/certs/ldap.crt
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ldap/certs/ldap.key
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/ldap/tls.ldif
```

### Enable LDAPS

```bash
sed -i 's/SLAPD_SERVICES="ldap:\/\/\/ ldapi:\/\/\/"/SLAPD_SERVICES="ldap:\/\/\/ ldaps:\/\/\/ ldapi:\/\/\/"/' /etc/default/slapd
systemctl restart slapd
```

### Configure Client Trust

```bash
cat > /etc/ldap/ldap.conf << 'EOF'
BASE dc=example,dc=com
URI ldaps://127.0.0.1
TLS_CACERT /etc/ldap/certs/ldap.crt
TLS_REQCERT allow
EOF
```

## Organizational Structure

```text
dn: dc=example,dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: Unseen University
dc: example

# Organizational Units
ou=people,dc=example,dc=com
ou=groups,dc=example,dc=com

# Admin User
uid=admin,ou=people,dc=example,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: admin
cn: Admin User
sn: Admin
uidNumber: 1000
gidNumber: 1000
homeDirectory: /home/admin
mail: admin@example.com
userPassword: {SSHA}...

# Groups
cn=developers,ou=groups,dc=example,dc=com
cn=admins,ou=groups,dc=example,dc=com
```

> **`mail` is required for OIDC.** Authelia maps the LDAP `mail` attribute to the
> OIDC `email` claim (`attributes.mail: mail`), and services like Gitea reject
> OIDC auto-registration when that claim is empty. To add it to an existing user:
>
> ```bash
> ldapmodify -x -D "cn=admin,dc=example,dc=com" -W << 'EOF'
> dn: uid=admin,ou=people,dc=example,dc=com
> changetype: modify
> add: mail
> mail: admin@example.com
> EOF
> ```

## Access Control (ACLs)

```bash
cat > /etc/ldap/acl.ldif << 'EOF'
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by self write by anonymous auth by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by self write by users read by * none
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/ldap/acl.ldif
```

## Password Policy (ppolicy)

```bash
cat > /etc/ldap/ppolicy.ldif << 'EOF'
dn: cn=module,cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: ppolicy.la

dn: ou=policies,dc=example,dc=com
objectClass: organizationalUnit
ou: policies

dn: cn=default,ou=policies,dc=example,dc=com
objectClass: pwdPolicy
objectClass: person
cn: default
pwdAttribute: userPassword
pwdMaxAge: 7776000
pwdExpireWarning: 432000
pwdInHistory: 5
pwdCheckQuality: 1
pwdMinLength: 12
pwdMaxFailure: 5
pwdLockout: TRUE
pwdLockoutDuration: 900
pwdGraceAuthNLimit: 3
pwdFailureCountInterval: 300
pwdMustChange: TRUE
pwdAllowUserChange: TRUE
pwdSafeModify: FALSE
EOF

ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/ppolicy.ldif
```

## User/Group Management

```bash
# Add user
ldapadd -Y EXTERNAL -H ldapi:/// -f user.ldif

# Change admin password
ldapmodify -Y EXTERNAL -H ldapi:/// << 'EOF'
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}NEW_HASH_HERE
EOF
```

### Adding a User (LDAP -> Authelia -> Gitea / Grafana)

A single LDAP entry powers every service: Authelia authenticates against LDAP,
Gitea auto-registers on first OIDC login (`--admin-group admins`), and Grafana
maps group membership to a role through `ROLE_ATTRIBUTE_PATH`. No service config
changes are required when onboarding someone.

1. **Hash the password** (run interactively so the plaintext never enters
   history; it prints a `{SSHA}...` hash):

   ```bash
   slappasswd
   ```

2. **Create the entry** — use the next free `uidNumber` and include `mail`,
   which OIDC needs for the `email` claim:

   ```bash
   umask 077
   HASH='{SSHA}PASTE_HASH_HERE'

   cat > /root/user.ldif <<EOF
   dn: uid=USER,ou=people,dc=example,dc=com
   objectClass: inetOrgPerson
   objectClass: posixAccount
   objectClass: shadowAccount
   uid: USER
   cn: Full Name
   sn: Last
   displayName: Full Name
   uidNumber: 1001
   gidNumber: 1000
   homeDirectory: /home/USER
   loginShell: /bin/bash
   mail: USER@example.com
   userPassword: $HASH
   EOF

   ldapadd -x -H ldapi:/// -D "cn=admin,dc=example,dc=com" -W -f /root/user.ldif
   shred -u /root/user.ldif
   ```

3. **Grant group membership** — this drives the Authelia access rules, Gitea
   admin rights, and the Grafana role:

   ```bash
   ldapmodify -x -H ldapi:/// -D "cn=admin,dc=example,dc=com" -W <<'EOF'
   dn: cn=admins,ou=groups,dc=example,dc=com
   changetype: modify
   add: memberUid
   memberUid: USER
   EOF
   ```

4. **Verify** — password bind, plus the exact filters Authelia uses:

   ```bash
   ldapwhoami -x -H ldapi:/// -D "uid=USER,ou=people,dc=example,dc=com" -W

   ldapsearch -x -H ldapi:/// -D "cn=admin,dc=example,dc=com" -W \
     -b "dc=example,dc=com" "(&(uid=USER)(objectClass=posixAccount))" uid mail

   ldapsearch -x -H ldapi:/// -D "cn=admin,dc=example,dc=com" -W \
     -b "dc=example,dc=com" "(&(memberUid=USER)(objectClass=posixGroup))" cn
   ```

5. **Finish in the browser**: sign in at the Authelia portal and register a TOTP
   device (first login only), then log in to Gitea and Grafana via SSO. Gitea
   creates the account automatically; membership in `admins` makes it a Gitea
   administrator and a Grafana `Admin`.

> **Writes need the root DN bind.** The SASL `EXTERNAL` peercred identity is only
> authorized for `cn=config`, not for the data database, so `ldapadd`/`ldapmodify`
> must bind as `cn=admin,dc=example,dc=com`.
```

## Firewall

```bash
ufw allow from 10.0.0.0/24 to any port 636 proto tcp
ufw reload
```

## Verification

```bash
# From Proxmox host - anonymous search
ldapsearch -x -H ldaps://10.0.0.20 -b "dc=example,dc=com" -TLS_REQCERT never

# Authenticated search
ldapsearch -x -H ldaps://10.0.0.20 -b "dc=example,dc=com" \
  -TLS_REQCERT never -D "cn=admin,dc=example,dc=com" -w "admin-password"

# Test admin bind
ldapwhoami -x -H ldaps://10.0.0.20 \
  -D "uid=admin,ou=people,dc=example,dc=com" -w "admin-password" \
  -TLS_REQCERT never
```

## Firewall Rules

| Source | Destination | Port | Protocol |
|--------|-------------|------|----------|
| 10.0.0.0/24 | 10.0.0.20 | 636 | TCP |

## Authelia Integration

Authelia (LXC 103) connects to OpenLDAP over LDAPS:

```yaml
# authelia configuration.yml snippet
authentication_backend:
  ldap:
    implementation: custom
    address: ldaps://10.0.0.20:636
    start_tls: false
    user: cn=admin,dc=example,dc=com
    password: admin-password
    base_dn: dc=example,dc=com
    users_filter: "(&({username_attribute}={input})(objectClass=posixAccount))"
    groups_filter: "(&(memberUid={username})(objectClass=posixGroup))"
    attributes:
      username: uid
      display_name: cn
      mail: mail
      member_of: memberOf
```

> `{username_attribute}` is substituted with the attribute name from
> `attributes.username` (i.e. `uid`), while `{input}` is the value the user
> typed. `{username}` in `groups_filter` is the resolved username.

## Backup

```bash
# Daily backup via cron
0 2 * * * slapcat -l /backup/ldap-$(date +%F).ldif
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `No such object (32)` | Check base DN, verify slapcat has data |
| `Invalid credentials (49)` | Reset root DN password via ldapmodify |
| `Can't contact LDAP server` | Check firewall, TLS config, service status |
| `TLS: peer cert untrusted` | Add cert to /etc/ldap/ldap.conf, TLS_REQCERT allow |