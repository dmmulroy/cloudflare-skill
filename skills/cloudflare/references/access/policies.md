# Access Policies

Control who can access applications and under what conditions.

## Overview

Policies define access rules for applications:
- **Application-specific policies** - Attached to individual apps
- **Reusable policies** - Shared across multiple applications
- **Precedence ordering** - Lower number = higher priority

## Policy Structure

```json
{
  "name": "Allow Engineering",
  "decision": "allow",
  "precedence": 1,
  "include": [
    {"group": {"id": "engineering-group-uuid"}}
  ],
  "exclude": [
    {"email": {"email": "contractor@example.com"}}
  ],
  "require": [
    {"warp": {}},
    {"device_posture": {"integration_uid": "antivirus-uuid"}}
  ],
  "purpose_justification_required": false,
  "purpose_justification_prompt": "",
  "approval_required": false,
  "approval_groups": []
}
```

## Decisions

| Decision | Effect |
|----------|--------|
| `allow` | Grant access if rules match |
| `deny` | Block access if rules match |
| `bypass` | Skip authentication entirely |
| `non_identity` | Service tokens only (no user auth) |

## Rule Logic

```
Final Decision = (matches include) AND (matches all require) AND NOT (matches any exclude)
```

- **include** - User must match at least ONE rule (OR logic)
- **require** - User must match ALL rules (AND logic)
- **exclude** - User must NOT match ANY rule

## Rule Examples

### Allow by Email Domain

```json
{
  "name": "Company Employees",
  "decision": "allow",
  "include": [
    {"email_domain": {"domain": "company.com"}}
  ]
}
```

### Allow Specific Users

```json
{
  "name": "Admins",
  "decision": "allow",
  "include": [
    {"email": {"email": "admin@company.com"}},
    {"email": {"email": "backup-admin@company.com"}}
  ]
}
```

### Allow Group with Device Requirements

```json
{
  "name": "Engineering with Secure Device",
  "decision": "allow",
  "include": [
    {"group": {"id": "engineering-group-uuid"}}
  ],
  "require": [
    {"warp": {}},
    {"device_posture": {"integration_uid": "disk-encryption-uuid"}}
  ]
}
```

### Block Specific Countries

```json
{
  "name": "Block Sanctioned Countries",
  "decision": "deny",
  "precedence": 1,
  "include": [
    {"country": {"code": "KP"}},
    {"country": {"code": "IR"}},
    {"country": {"code": "CU"}}
  ]
}
```

### Service Token Access

```json
{
  "name": "API Access",
  "decision": "non_identity",
  "include": [
    {"service_token": {"token_id": "token-uuid"}}
  ]
}
```

### Public Access (Bypass)

```json
{
  "name": "Public Endpoints",
  "decision": "bypass",
  "include": [
    {"everyone": {}}
  ]
}
```

## Precedence

Policies are evaluated in order of precedence (lowest number first):

```
1. Block Sanctioned Countries (deny, precedence: 1)
2. Allow Admins (allow, precedence: 2)
3. Allow Engineering (allow, precedence: 3)
4. Allow Everyone Else (allow, precedence: 10)
```

**Best practice:** Use gaps in precedence (1, 10, 20) to allow inserting policies later.

## Purpose Justification

Require users to explain why they need access:

```json
{
  "name": "Sensitive Data Access",
  "decision": "allow",
  "include": [{"group": {"id": "data-team-uuid"}}],
  "purpose_justification_required": true,
  "purpose_justification_prompt": "Please explain why you need access to this data"
}
```

## Approval Workflows

Require manager/admin approval before granting access:

```json
{
  "name": "Production Access with Approval",
  "decision": "allow",
  "include": [{"group": {"id": "engineers-uuid"}}],
  "approval_required": true,
  "approval_groups": [
    {
      "email_list_uuid": "approvers-uuid",
      "approvals_needed": 1
    }
  ]
}
```

## Reusable Policies

Create policies that can be attached to multiple apps:

```bash
# Create reusable policy
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/policies" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Standard Corporate Access",
    "decision": "allow",
    "include": [{"email_domain": {"domain": "company.com"}}],
    "require": [{"warp": {}}]
  }'

# List reusable policies
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/policies"
```

## Common Patterns

### Tiered Access

```
App: Production Dashboard
├─ Policy 1 (precedence: 1): Block non-US (deny)
├─ Policy 2 (precedence: 2): Allow SRE team (allow)
└─ Policy 3 (precedence: 10): Allow viewers read-only (allow, different app path)
```

### Contractor Access

```json
{
  "name": "Contractors",
  "decision": "allow",
  "include": [
    {"email_domain": {"domain": "contractor-company.com"}}
  ],
  "require": [
    {"device_posture": {"integration_uid": "managed-device-uuid"}},
    {"country": {"code": "US"}}
  ],
  "exclude": [
    {"email": {"email": "terminated@contractor-company.com"}}
  ]
}
```

### Time-Limited Access

Use service tokens with short duration for temporary access:

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/service_tokens" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Temp Contractor Access", "duration": "24h"}'
```

## Policy Testing

### Test Policy Evaluation

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/policy-tests" \
  -d '{"email": "user@example.com", "app_id": "app-uuid"}'
```

### View User's Effective Access

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps/$APP_ID/user_policy_checks"
```

## See Also

- [Groups](./groups.md) - Reusable rule collections
- [Service Tokens](./service-tokens.md) - Machine authentication
- [Gotchas](./gotchas.md) - Common policy issues
