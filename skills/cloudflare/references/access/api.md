# Access API Reference

Base URL: `https://api.cloudflare.com/client/v4/accounts/{account_id}`

## Applications

### List Applications

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps" | jq '.result'
```

### Get Application

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps/$APP_ID"
```

### Create Application

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "App Name",
    "domain": "app.example.com",
    "type": "self_hosted",
    "session_duration": "24h",
    "auto_redirect_to_identity": false,
    "app_launcher_visible": true,
    "logo_url": "https://example.com/logo.png",
    "allowed_idps": ["IDP_UUID"],
    "cors_headers": {
      "allowed_methods": ["GET", "POST"],
      "allowed_origins": ["https://example.com"],
      "allow_credentials": true
    }
  }'
```

### Update Application

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps/$APP_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Name",
    "session_duration": "12h"
  }'
```

### Delete Application

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps/$APP_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

### Revoke Application Tokens

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps/$APP_ID/revoke_tokens" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Policies

### List Policies for Application

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps/$APP_ID/policies"
```

### Create Policy

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps/$APP_ID/policies" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Allow Engineering",
    "decision": "allow",
    "precedence": 1,
    "include": [
      {"group": {"id": "GROUP_UUID"}}
    ],
    "exclude": [],
    "require": []
  }'
```

### Update Policy

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps/$APP_ID/policies/$POLICY_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Policy",
    "decision": "allow",
    "precedence": 1,
    "include": [
      {"email_domain": {"domain": "example.com"}}
    ]
  }'
```

### Delete Policy

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps/$APP_ID/policies/$POLICY_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Reusable Policies

### List Reusable Policies

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/policies"
```

### Create Reusable Policy

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/policies" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Corporate Users",
    "decision": "allow",
    "include": [
      {"email_domain": {"domain": "company.com"}}
    ]
  }'
```

## Groups

### List Groups

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/groups"
```

### Create Group

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/groups" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Engineering Team",
    "include": [
      {"email_domain": {"domain": "eng.example.com"}},
      {"email": {"email": "contractor@example.com"}}
    ],
    "exclude": [],
    "require": []
  }'
```

### Update Group

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/groups/$GROUP_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Group Name",
    "include": [{"email_domain": {"domain": "example.com"}}]
  }'
```

### Delete Group

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/groups/$GROUP_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Identity Providers

### List Identity Providers

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/identity_providers"
```

### Create Identity Provider (Okta Example)

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/identity_providers" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Okta",
    "type": "okta",
    "config": {
      "client_id": "YOUR_CLIENT_ID",
      "client_secret": "YOUR_CLIENT_SECRET",
      "okta_account": "https://your-org.okta.com"
    }
  }'
```

### Delete Identity Provider

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/identity_providers/$IDP_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Service Tokens

### List Service Tokens

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/service_tokens"
```

### Create Service Token

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/service_tokens" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "CI/CD Token",
    "duration": "8760h"
  }'
```

**Response includes `client_id` and `client_secret` - save these immediately!**

### Rotate Service Token

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/service_tokens/$TOKEN_ID/rotate" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

### Delete Service Token

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/service_tokens/$TOKEN_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Users & Sessions

### List Access Users

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/users"
```

### Get User's Active Sessions

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/users/$USER_ID/active_sessions"
```

### Get User's Failed Logins

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/users/$USER_ID/failed_logins"
```

### Revoke User's Sessions

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/organizations/revoke_user" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com"}'
```

## Access Logs

### Get Access Request Logs

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/logs/access_requests?limit=50"
```

## Certificates

### List mTLS Certificates

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/certificates"
```

### Upload mTLS Certificate

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/certificates" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Client CA",
    "certificate": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
    "associated_hostnames": ["app.example.com"]
  }'
```

## Tags

### List Tags

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/tags"
```

### Create Tag

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/tags" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "production"}'
```
