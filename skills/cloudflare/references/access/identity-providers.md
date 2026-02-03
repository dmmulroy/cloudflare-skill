# Identity Providers

Configure authentication sources for Cloudflare Access.

## Supported Providers

| Provider | Type | SCIM Support |
|----------|------|--------------|
| Okta | `okta` | Yes |
| Azure AD | `azure` | Yes |
| Google Workspace | `google` | No |
| GitHub | `github` | No |
| OneLogin | `onelogin` | Yes |
| Ping Identity | `ping` | No |
| SAML 2.0 (Generic) | `saml` | Varies |
| OIDC (Generic) | `oidc` | No |
| One-Time PIN | `onetimepin` | N/A |
| LinkedIn | `linkedin` | No |
| Facebook | `facebook` | No |

## Okta Configuration

### 1. Create OIDC App in Okta

In Okta Admin Console:
1. Applications → Create App Integration
2. Sign-in method: OIDC
3. Application type: Web Application
4. Sign-in redirect URI: `https://<team-name>.cloudflareaccess.com/cdn-cgi/access/callback`
5. Sign-out redirect URI: `https://<team-name>.cloudflareaccess.com`

### 2. Configure in Cloudflare

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/identity_providers" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Okta",
    "type": "okta",
    "config": {
      "client_id": "YOUR_OKTA_CLIENT_ID",
      "client_secret": "YOUR_OKTA_CLIENT_SECRET",
      "okta_account": "https://your-org.okta.com",
      "claims": ["groups", "email", "name"],
      "email_claim_name": "email"
    }
  }'
```

### 3. Enable SCIM Provisioning (Optional)

In Okta: Applications → Your App → Provisioning → Enable SCIM

Cloudflare SCIM endpoint: `https://your-team.cloudflareaccess.com/scim/v2`

## Azure AD Configuration

### 1. Create App Registration in Azure

1. Azure Portal → App registrations → New registration
2. Redirect URI: `https://<team-name>.cloudflareaccess.com/cdn-cgi/access/callback`
3. Certificates & secrets → New client secret
4. API permissions → Add `openid`, `email`, `profile`, `User.Read`

### 2. Configure in Cloudflare

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/identity_providers" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Azure AD",
    "type": "azure",
    "config": {
      "client_id": "YOUR_AZURE_APP_ID",
      "client_secret": "YOUR_AZURE_CLIENT_SECRET",
      "directory_id": "YOUR_AZURE_TENANT_ID",
      "support_groups": true
    }
  }'
```

## Google Workspace Configuration

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/identity_providers" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Google Workspace",
    "type": "google",
    "config": {
      "client_id": "YOUR_GOOGLE_CLIENT_ID",
      "client_secret": "YOUR_GOOGLE_CLIENT_SECRET"
    }
  }'
```

## GitHub Configuration

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/identity_providers" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "GitHub",
    "type": "github",
    "config": {
      "client_id": "YOUR_GITHUB_OAUTH_CLIENT_ID",
      "client_secret": "YOUR_GITHUB_OAUTH_CLIENT_SECRET"
    }
  }'
```

**Using GitHub in policies:**
```json
{
  "include": [
    {"github_organization": {"name": "my-org", "identity_provider_id": "idp-uuid"}}
  ]
}
```

## Generic SAML 2.0

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/identity_providers" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Corporate SAML",
    "type": "saml",
    "config": {
      "issuer_url": "https://idp.company.com",
      "sso_target_url": "https://idp.company.com/sso/saml",
      "idp_public_cert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
      "sign_request": true,
      "attributes": ["email", "name", "groups"],
      "email_attribute_name": "email"
    }
  }'
```

**Cloudflare SP Metadata:**
- Entity ID: `https://<team-name>.cloudflareaccess.com`
- ACS URL: `https://<team-name>.cloudflareaccess.com/cdn-cgi/access/callback`

## Generic OIDC

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/identity_providers" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Custom OIDC",
    "type": "oidc",
    "config": {
      "client_id": "YOUR_CLIENT_ID",
      "client_secret": "YOUR_CLIENT_SECRET",
      "auth_url": "https://idp.example.com/authorize",
      "token_url": "https://idp.example.com/token",
      "certs_url": "https://idp.example.com/.well-known/jwks.json",
      "scopes": ["openid", "email", "profile"],
      "claims": ["email", "name", "groups"]
    }
  }'
```

## One-Time PIN (Email OTP)

No IdP required - users receive a code via email:

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/identity_providers" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Email OTP",
    "type": "onetimepin"
  }'
```

## Multiple IdPs

You can configure multiple identity providers. Users will see a selection screen unless:
- `auto_redirect_to_identity: true` on the app (redirects to first IdP)
- `allowed_idps` specified on the app (limits choices)

```json
{
  "name": "Internal App",
  "domain": "app.example.com",
  "allowed_idps": ["okta-idp-uuid", "azure-idp-uuid"]
}
```

## SCIM User/Group Sync

### View Synced Users

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/identity_providers/$IDP_ID/scim/users"
```

### View Synced Groups

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/identity_providers/$IDP_ID/scim/groups"
```

## Troubleshooting

### Common Issues

1. **Redirect URI mismatch** - Ensure exact match including trailing slash
2. **Invalid client secret** - Regenerate and update
3. **Missing scopes** - Add required scopes in IdP app config
4. **SAML signature verification failed** - Update IdP certificate

### Test IdP Connection

```bash
# Get IdP details
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/identity_providers/$IDP_ID"
```

## See Also

- [Groups](./groups.md) - Using IdP groups in policies
- [Gotchas](./gotchas.md) - Common IdP issues
