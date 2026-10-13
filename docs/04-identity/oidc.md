# OpenID Connect (OIDC) Configuration

## Overview

OpenID Connect (OIDC) is an identity layer on top of OAuth 2.0. Authelia acts as the OIDC Provider (OP), and services like Gitea and Grafana act as Relying Parties (RP).

## Flow

```
User -> Service (Gitea/Grafana)
        |
        v
    Authelia (OIDC authorization endpoint)
        |
        v
    OpenLDAP (Authentication)
        |
        v
    Authelia -> authorization code -> ID Token -> Service
```

The service sends the user straight to Authelia's authorization endpoint (the
user's browser is redirected to the Authelia portal). There is no Nginx
`auth_request` forward-auth in front of the service.

## Authelia as OIDC Provider

### Configuration

```yaml
# In authelia configuration.yml
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

> **v4.38+ changes:** `issuer_private_key` was replaced by the `jwks` list, and
> `hmac_secret` is now required. Client secrets should be stored as a PBKDF2
> digest (`authelia crypto hash generate pbkdf2 --variant sha512 --password ...`).
> `offline_access` must be present for any client that requests `refresh_token`.

## Client Registration

### Gitea

Configure the OAuth2 source from the CLI (`gitea admin auth add-oauth`):

```bash
gitea admin auth add-oauth \
  --name oidc \
  --provider openidConnect \
  --key gitea \
  --secret '<PLAINTEXT_CLIENT_SECRET>' \
  --auto-discover-url https://auth.example.com/.well-known/openid-configuration \
  --scopes openid --scopes profile --scopes email --scopes groups --scopes offline_access \
  --group-claim-name groups \
  --admin-group admins \
  --skip-local-2fa
```

The source **name** determines the callback path, so it must match the
`redirect_uri` registered in Authelia
(`https://git.example.com/user/oauth2/oidc/callback`).

To allow new users to be created on first OIDC login (and link an existing local
account with the same email automatically), add to `/etc/gitea/app.ini`:

```ini
[oauth2_client]
ENABLE_AUTO_REGISTRATION = true
ACCOUNT_LINKING = auto
USERNAME = preferred_username
```

> The `email` scope is populated from the LDAP `mail` attribute, and Gitea
> refuses auto-registration when the `email` claim is empty. Every LDAP user must
> therefore have a `mail` attribute. See [OpenLDAP](openldap.md).

### Grafana

1. grafana.ini or environment variables:

```ini
[auth.generic_oauth]
enabled = true
name = Authelia
allow_sign_up = true
client_id = grafana
client_secret = grafana-secret
scopes = openid profile email offline_access
auth_url = https://auth.example.com/api/oidc/authorization
token_url = https://auth.example.com/api/oidc/token
api_url = https://auth.example.com/api/oidc/userinfo
allowed_domains = example.com
team_ids = 1
role_attribute_path = groups
```

## Scopes

| Scope | Description | Claims |
|-------|-------------|--------|
| openid | Required for OIDC | sub |
| profile | User profile | name, preferred_username |
| email | Email address | email, email_verified |
| groups | Group membership | groups |
| offline_access | Enables refresh tokens | (refresh_token issued) |

## ID Token Claims

```json
{
  "iss": "https://auth.example.com",
  "sub": "<opaque-subject-id>",
  "aud": "gitea",
  "exp": 1699999999,
  "iat": 1699996399,
  "auth_time": 1699996399,
  "name": "Admin User",
  "preferred_username": "admin",
  "email": "admin@example.com",
  "email_verified": true,
  "groups": ["admins", "developers"]
}
```

## PKCE (Proof Key for Code Exchange)

Required for public clients, recommended for all:

- Challenge Method: S256
- Code Verifier: 43-128 chars (base64url)
- Code Challenge: SHA256(verifier) base64url

## Token Endpoint

```
POST https://auth.example.com/api/oidc/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=AUTH_CODE&
redirect_uri=https://git.example.com/user/oauth2/oidc/callback&
client_id=gitea&
client_secret=SECRET&
code_verifier=VERIFIER
```

Response:
```json
{
  "access_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "eyJ...",
  "id_token": "eyJ...",
  "scope": "openid profile email groups"
}
```

## UserInfo Endpoint

```
GET https://auth.example.com/api/oidc/userinfo
Authorization: Bearer ACCESS_TOKEN
```

## Discovery Document

```bash
curl https://auth.example.com/.well-known/openid-configuration
```

Response:
```json
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/api/oidc/authorization",
  "token_endpoint": "https://auth.example.com/api/oidc/token",
  "userinfo_endpoint": "https://auth.example.com/api/oidc/userinfo",
  "jwks_uri": "https://auth.example.com/jwks.json",
  "scopes_supported": ["openid", "profile", "email", "groups", "offline_access"],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "claims_supported": ["sub", "name", "preferred_username", "email", "email_verified", "groups"]
}
```

## JWKS (JSON Web Key Set)

```bash
curl https://auth.example.com/jwks.json
```

## Logout

### RP-Initiated Logout

```
GET https://auth.example.com/api/oidc/logout?id_token_hint=ID_TOKEN&post_logout_redirect_uri=https://git.example.com
```

### Front-Channel Logout

Configure logout URL in client application.

## Security Considerations

- Use PKCE for all clients
- Rotate client secrets periodically
- Use short access token lifetimes (1h)
- Store client secrets securely (SOPS/age)
- Validate redirect URIs exactly
- Monitor token endpoint for abuse
