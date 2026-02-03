# Service Tokens

Machine-to-machine authentication for APIs and automated systems.

## Overview

Service tokens provide:
- **Non-interactive authentication** - No user login required
- **API access** - Authenticate automated systems
- **Time-limited access** - Configurable expiration
- **Rotatable credentials** - Refresh without downtime

## Creating Service Tokens

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/service_tokens" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "CI/CD Pipeline",
    "duration": "8760h"
  }'
```

**Response (save immediately!):**
```json
{
  "result": {
    "id": "token-uuid",
    "name": "CI/CD Pipeline",
    "client_id": "abc123.access",
    "client_secret": "secret-value-only-shown-once",
    "expires_at": "2025-01-15T00:00:00Z"
  }
}
```

**Duration options:** `8760h` (1 year), `17520h` (2 years), `26280h` (3 years), `forever`

## Using Service Tokens

### HTTP Headers

```bash
curl -H "CF-Access-Client-Id: abc123.access" \
     -H "CF-Access-Client-Secret: secret-value" \
     https://app.example.com/api/data
```

### In Code (Node.js)

```javascript
const response = await fetch('https://app.example.com/api/data', {
  headers: {
    'CF-Access-Client-Id': process.env.CF_ACCESS_CLIENT_ID,
    'CF-Access-Client-Secret': process.env.CF_ACCESS_CLIENT_SECRET
  }
});
```

### In Code (Python)

```python
import requests

response = requests.get(
    'https://app.example.com/api/data',
    headers={
        'CF-Access-Client-Id': os.environ['CF_ACCESS_CLIENT_ID'],
        'CF-Access-Client-Secret': os.environ['CF_ACCESS_CLIENT_SECRET']
    }
)
```

## Configuring Application Policies

### Allow Only Service Tokens

```json
{
  "name": "API-Only Access",
  "decision": "non_identity",
  "include": [
    {"service_token": {}}
  ]
}
```

### Allow Specific Token

```json
{
  "name": "CI/CD Pipeline Only",
  "decision": "non_identity",
  "include": [
    {"service_token": {"token_id": "token-uuid"}}
  ]
}
```

### Allow Both Users and Service Tokens

```json
{
  "name": "Users and Services",
  "decision": "allow",
  "include": [
    {"group": {"id": "developers-uuid"}},
    {"service_token": {"token_id": "cicd-token-uuid"}}
  ]
}
```

## Token Management

### List All Tokens

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/service_tokens" | \
  jq '.result[] | {id, name, expires_at}'
```

### Rotate Token (Get New Secret)

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/service_tokens/$TOKEN_ID/rotate" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

**Important:** Rotation generates a new secret. Old secret remains valid until the new one is used, allowing graceful rotation.

### Refresh Token (Extend Expiration)

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/service_tokens/$TOKEN_ID/refresh" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

### Delete Token

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/service_tokens/$TOKEN_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Best Practices

### 1. Use Descriptive Names

```bash
# Good
{"name": "GitHub Actions - Production Deploy"}
{"name": "Jenkins - Staging Tests"}

# Bad
{"name": "token1"}
{"name": "temp"}
```

### 2. Scope Tokens to Specific Apps

Create separate tokens for different services/applications rather than one "master" token.

### 3. Set Appropriate Expiration

- CI/CD pipelines: 1 year with automated rotation
- Short-term access: Match project duration
- Critical infrastructure: Use `forever` with regular audits

### 4. Store Securely

- Use secrets managers (Vault, AWS Secrets Manager, 1Password)
- Never commit to source control
- Use environment variables in CI/CD

### 5. Implement Rotation

Automate token rotation before expiration:

```bash
#!/bin/bash
# Rotate token and update secrets manager
NEW_SECRET=$(curl -X POST ".../service_tokens/$TOKEN_ID/rotate" ... | jq -r '.result.client_secret')
# Update your secrets manager with $NEW_SECRET
```

## Monitoring Token Usage

### Check Token Last Used

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/service_tokens/$TOKEN_ID" | \
  jq '{name, last_seen_at, expires_at}'
```

### Audit Token Access in Logs

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/logs/access_requests?limit=100" | \
  jq '.result[] | select(.is_service_token == true)'
```

## Terraform Example

```hcl
resource "cloudflare_zero_trust_access_service_token" "cicd" {
  account_id = var.cloudflare_account_id
  name       = "GitHub Actions"
  duration   = "8760h"
}

output "client_id" {
  value = cloudflare_zero_trust_access_service_token.cicd.client_id
}

output "client_secret" {
  value     = cloudflare_zero_trust_access_service_token.cicd.client_secret
  sensitive = true
}
```

## See Also

- [Policies](./policies.md) - Configuring service token policies
- [API](./api.md) - Full API reference
