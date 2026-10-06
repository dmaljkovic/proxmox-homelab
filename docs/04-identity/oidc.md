# OpenID Connect (OIDC) Configuration

## Overview

OpenID Connect (OIDC) is an identity layer on top of OAuth 2.0. Authelia acts as the OIDC Provider (OP), and services like Gitea and Grafana act as Relying Parties (RP).

## Flow

```
User -> Service (Gitea/Grafana)
        |
        v
    Nginx (auth_request)
        |
        v
    Authelia (OIDC Provider)
        |
        v
    OpenLDAP (Authentication)
        |
        v
    Authelia -> ID Token -> Service
```

## Authelia as OIDC Provider

### Configuration

```yaml
# In authelia configuration.yml
identity_providers:
  oidc:
    issuer_private_key: "/config/certs/authelia.key"
    issuer_public_key: "/config/certs/authelia.pub"
    cors:
      endpoints:
        - authorization
        - token
        - revocation
        - introspection
    clients:
      - client_id: gitea
        client_name: Gitea
        client_secret: "gitea-secret"
        redirect_uris:
          - https://git.unseen-uni.xyz/user/oauth2/oidc/callback
        scopes:
          - openid
          - profile
          - email
          - groups
        grant_types:
          - authorization_code
          - refresh_token
        response_types:
          - code
        public: false
        require_pkce: true
        pkce_challenge_method: S256

      - client_id: grafana
        client_name: Grafana
        client_secret: "grafana-secret"
        redirect_uris:
          - https://grafana.unseen-uni.xyz/login/generic_oauth
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

## Client Registration

### Gitea

1. Admin Panel -> Authentication -> OAuth2
2. Add OAuth2 Provider:
   - Name: Authelia
   - Client ID: gitea
   - Client Secret: (from Authelia config)
   - Redirect URI: https://git.unseen-uni.xyz/user/oauth2/oidc/callback
   - OpenID Connect Discovery URL: https://auth.unseen-uni.xyz/.well-known/openid-configuration
   - Scopes: openid profile email groups
   - Group Attribute Path: groups
   - Admin Group: admins

### Grafana

1. grafana.ini or environment variables:

```ini
[auth.generic_oauth]
enabled = true
name = Authelia
allow_sign_up = true
client_id = grafana
client_secret = grafana-secret
scopes = openid profile email
auth_url = https://auth.unseen-uni.xyz/api/oidc/authorization
token_url = https://auth.unseen-uni.xyz/api/oidc/token
api_url = https://auth.unseen-uni.xyz/api/oidc/userinfo
allowed_domains = unseen-uni.xyz
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

## ID Token Claims

```json
{
  "iss": "https://auth.unseen-uni.xyz",
  "sub": "admin",
  "aud": "gitea",
  "exp": 1699999999,
  "iat": 1699996399,
  "auth_time": 1699996399,
  "name": "Admin User",
  "preferred_username": "admin",
  "email": "admin@unseen-uni.xyz",
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
POST https://auth.unseen-uni.xyz/api/oidc/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=AUTH_CODE&
redirect_uri=https://git.unseen-uni.xyz/user/oauth2/oidc/callback&
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
GET https://auth.unseen-uni.xyz/api/oidc/userinfo
Authorization: Bearer ACCESS_TOKEN
```

## Discovery Document

```bash
curl https://auth.unseen-uni.xyz/.well-known/openid-configuration
```

Response:
```json
{
  "issuer": "https://auth.unseen-uni.xyz",
  "authorization_endpoint": "https://auth.unseen-uni.xyz/api/oidc/authorization",
  "token_endpoint": "https://auth.unseen-uni.xyz/api/oidc/token",
  "userinfo_endpoint": "https://auth.unseen-uni.xyz/api/oidc/userinfo",
  "jwks_uri": "https://auth.unseen-uni.xyz/api/oidc/jwks",
  "scopes_supported": ["openid", "profile", "email", "groups"],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "claims_supported": ["sub", "name", "preferred_username", "email", "email_verified", "groups"]
}
```

## JWKS (JSON Web Key Set)

```bash
curl https://auth.unseen-uni.xyz/api/oidc/jwks
```

## Logout

### RP-Initiated Logout

```
GET https://auth.unseen-uni.xyz/api/oidc/logout?id_token_hint=ID_TOKEN&post_logout_redirect_uri=https://git.unseen-uni.xyz
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
