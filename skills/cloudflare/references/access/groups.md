# Access Groups

Reusable collections of rules for Access policies. Define once, use across multiple applications.

## Overview

Access Groups allow you to:
- Define user/device criteria once and reuse across applications
- Simplify policy management at scale
- Sync with identity provider groups (Okta, Azure AD, etc.)

## Group Structure

```json
{
  "name": "Engineering Team",
  "include": [
    {"email_domain": {"domain": "eng.example.com"}},
    {"email": {"email": "contractor@external.com"}}
  ],
  "exclude": [
    {"email": {"email": "former-employee@eng.example.com"}}
  ],
  "require": [
    {"device_posture": {"integration_uid": "posture-check-uuid"}}
  ]
}
```

**Logic:**
- `include` - Must match at least one (OR)
- `exclude` - Must NOT match any
- `require` - Must match ALL (AND)

## Rule Types

### Identity-Based Rules

```json
// Specific email
{"email": {"email": "user@example.com"}}

// Email domain
{"email_domain": {"domain": "example.com"}}

// Everyone (any authenticated user)
{"everyone": {}}

// Identity Provider Group (from IdP)
{"saml": {"attribute_name": "groups", "attribute_value": "Engineering"}}

// Azure AD Group
{"azure_ad": {"id": "group-object-id", "identity_provider_id": "idp-uuid"}}

// GitHub Organization
{"github_organization": {"name": "my-org", "identity_provider_id": "idp-uuid"}}

// GitHub Team
{"github_organization": {"name": "my-org", "team": "engineering", "identity_provider_id": "idp-uuid"}}

// Okta Group
{"okta": {"name": "Engineering", "identity_provider_id": "idp-uuid"}}

// Google Workspace Group
{"gsuite": {"email": "group@example.com", "identity_provider_id": "idp-uuid"}}
```

### Network-Based Rules

```json
// IP Range (CIDR)
{"ip_range": {"ip": "10.0.0.0/8"}}

// Country
{"country": {"code": "US"}}

// Multiple countries
{"geo": {"country_code": "US"}}
```

### Device-Based Rules

```json
// Device posture check
{"device_posture": {"integration_uid": "posture-rule-uuid"}}

// Managed device (WARP)
{"warp": {}}

// Gateway managed
{"gateway": {}}
```

### Certificate-Based Rules

```json
// mTLS certificate
{"certificate": {}}

// Specific certificate CN
{"common_name": {"common_name": "device-001.example.com"}}
```

### Service Token Rules

```json
// Any service token
{"service_token": {}}

// Specific service token
{"service_token": {"token_id": "token-uuid"}}
```

## Common Patterns

### Corporate Users

```json
{
  "name": "Corporate Users",
  "include": [
    {"email_domain": {"domain": "company.com"}}
  ],
  "require": [
    {"warp": {}}
  ]
}
```

### Engineering Team (Okta Synced)

```json
{
  "name": "Engineering",
  "include": [
    {"okta": {"name": "Engineering", "identity_provider_id": "IDP_UUID"}}
  ]
}
```

### Contractors with Device Posture

```json
{
  "name": "Contractors",
  "include": [
    {"email_domain": {"domain": "contractor.com"}},
    {"email": {"email": "special@external.com"}}
  ],
  "require": [
    {"device_posture": {"integration_uid": "antivirus-check-uuid"}},
    {"device_posture": {"integration_uid": "disk-encryption-uuid"}}
  ]
}
```

### Regional Access

```json
{
  "name": "US Employees",
  "include": [
    {"email_domain": {"domain": "company.com"}}
  ],
  "require": [
    {"country": {"code": "US"}}
  ]
}
```

### Service Accounts

```json
{
  "name": "Service Accounts",
  "include": [
    {"service_token": {}}
  ]
}
```

## Using Groups in Policies

Once created, reference groups in application policies:

```json
{
  "name": "Allow Engineering",
  "decision": "allow",
  "include": [
    {"group": {"id": "engineering-group-uuid"}}
  ]
}
```

## IdP Group Sync

### Okta SCIM Sync

Groups from Okta can be automatically synced:

```bash
# View synced groups from IdP
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/identity_providers/$IDP_ID/scim/groups"
```

### Azure AD Sync

Enable SCIM provisioning in Azure AD to sync groups automatically.

## Best Practices

1. **Use IdP groups when possible** - Sync from Okta/Azure AD for single source of truth
2. **Layer groups** - Create broad groups (all employees) and specific groups (engineering)
3. **Combine with device posture** - Add `require` rules for sensitive applications
4. **Name clearly** - Use descriptive names like "Engineering - Production Access"
5. **Document membership** - Keep track of who manages group membership in IdP

## Limitations

- Maximum 100 rules per group
- Group names must be unique within account
- Changes propagate within 60 seconds
