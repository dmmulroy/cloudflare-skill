# Cloudflare Access

Zero Trust application access control. Secure web apps, SSH, RDP, and APIs without a VPN.

## Overview

Cloudflare Access provides:
- **Identity-aware proxy** - Authenticate users before they reach your app
- **Multiple app types** - Self-hosted, SaaS, infrastructure (SSH/RDP/VNC)
- **Flexible policies** - Based on identity, device posture, location, and more
- **Service tokens** - Machine-to-machine authentication

**Architecture**: User → Cloudflare Edge → Access Policy Check → Origin Application

## Application Types

| Type | Use Case | Example |
|------|----------|---------|
| `self_hosted` | Web apps you control | Internal dashboard at `app.example.com` |
| `saas` | Third-party SaaS apps | Okta, Salesforce, GitHub |
| `infrastructure` | SSH/RDP/VNC/SMB | Server at `ssh.example.com` |
| `bookmark` | Quick links in App Launcher | Link to external tool |
| `app_launcher` | User portal | Central access hub |

## Quick Start

### 1. Create Self-Hosted Application

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Internal Dashboard",
    "domain": "dashboard.example.com",
    "type": "self_hosted",
    "session_duration": "24h"
  }'
```

### 2. Create Policy for Application

```bash
# Get the app_id from step 1
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps/$APP_ID/policies" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Allow Engineering Team",
    "decision": "allow",
    "include": [
      {"group": {"id": "GROUP_UUID"}}
    ],
    "precedence": 1
  }'
```

### 3. Connect Application via Tunnel

```bash
# Create tunnel
cloudflared tunnel create dashboard-tunnel

# Configure ingress (in config.yml)
# tunnel: <TUNNEL_ID>
# ingress:
#   - hostname: dashboard.example.com
#     service: http://localhost:8080
#   - service: http_status:404

# Route DNS
cloudflared tunnel route dns dashboard-tunnel dashboard.example.com

# Run
cloudflared tunnel run dashboard-tunnel
```

## Policy Decisions

| Decision | Effect |
|----------|--------|
| `allow` | Grant access if rules match |
| `deny` | Block access if rules match |
| `bypass` | Skip authentication (public access) |
| `non_identity` | Allow service tokens only |

## Policy Rules

Policies use `include`, `exclude`, and `require` arrays:

```json
{
  "name": "Engineering Access",
  "decision": "allow",
  "include": [
    {"group": {"id": "engineering-group-uuid"}},
    {"email": {"email": "admin@example.com"}}
  ],
  "exclude": [
    {"ip_range": {"ip": "192.168.1.0/24"}}
  ],
  "require": [
    {"device_posture": {"integration_uid": "posture-check-uuid"}}
  ]
}
```

**Logic:**
- `include` - User must match at least one (OR)
- `require` - User must match all (AND)
- `exclude` - User must not match any (NOT)

## Common Rule Types

| Type | Example | Description |
|------|---------|-------------|
| `email` | `{"email": "user@example.com"}` | Specific email |
| `email_domain` | `{"email_domain": "example.com"}` | Email domain |
| `group` | `{"id": "uuid"}` | Access Group |
| `everyone` | `{}` | All authenticated users |
| `ip_range` | `{"ip": "10.0.0.0/8"}` | IP/CIDR range |
| `country` | `{"code": "US"}` | Country code |
| `device_posture` | `{"integration_uid": "uuid"}` | Posture check |
| `service_token` | `{"id": "uuid"}` | Service token |
| `certificate` | `{}` | mTLS certificate |

## Session Duration

```json
{
  "session_duration": "24h",      // How long before re-auth
  "same_site_cookie_attribute": "lax"
}
```

**Values:** `30m`, `1h`, `6h`, `12h`, `24h`, `168h` (1 week)

## In This Reference

- [Policies](./policies.md) - Policy configuration details
- [Groups](./groups.md) - Access Groups for rule reuse
- [Identity Providers](./identity-providers.md) - IdP configuration
- [Service Tokens](./service-tokens.md) - Machine authentication
- [API](./api.md) - Full API reference
- [Gotchas](./gotchas.md) - Common issues and limits

## See Also

- [Gateway](../gateway/README.md) - DNS/HTTP filtering
- [Tunnel](../tunnel/README.md) - Secure connectivity
- [Devices](../devices/README.md) - Device posture integration
