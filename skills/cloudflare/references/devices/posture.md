# Device Posture

Verify device security compliance before granting access.

## Overview

Device posture checks verify:
- **OS version** - Minimum OS requirements
- **Disk encryption** - FileVault, BitLocker enabled
- **Firewall** - Host firewall active
- **Antivirus/EDR** - Security software running
- **Domain joined** - AD/Azure AD membership
- **Device serial** - Known corporate devices

## Posture Check Types

| Type | Platforms | Description |
|------|-----------|-------------|
| `file` | All | Check if file exists |
| `application` | All | Check if app is running |
| `serial_number` | All | Match device serial |
| `unique_client_id` | All | Match WARP client ID |
| `os_version` | All | Minimum OS version |
| `disk_encryption` | All | Full disk encryption |
| `firewall` | macOS, Windows | Host firewall enabled |
| `domain_joined` | Windows | AD domain membership |
| `client_certificate` | All | mTLS certificate present |
| `workspace_one` | All | VMware Workspace ONE status |
| `crowdstrike` | All | CrowdStrike Falcon status |
| `intune` | Windows | Microsoft Intune compliance |
| `kolide` | All | Kolide device health |
| `tanium` | All | Tanium endpoint status |
| `sentinelone` | All | SentinelOne agent status |

## Creating Posture Checks

### Disk Encryption Check

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Disk Encryption Required",
    "type": "disk_encryption",
    "description": "Requires FileVault or BitLocker",
    "schedule": "5m",
    "match": [
      {"platform": "mac"},
      {"platform": "windows"}
    ],
    "input": {
      "require_all": true
    }
  }'
```

### OS Version Check

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Minimum OS Version",
    "type": "os_version",
    "match": [{"platform": "mac"}],
    "input": {
      "version": "13.0.0",
      "operator": ">="
    }
  }'
```

### Firewall Check

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Firewall Enabled",
    "type": "firewall",
    "match": [
      {"platform": "mac"},
      {"platform": "windows"}
    ],
    "input": {
      "enabled": true
    }
  }'
```

### Application Check

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Antivirus Running",
    "type": "application",
    "match": [{"platform": "windows"}],
    "input": {
      "operating_system": "windows",
      "path": {
        "windows": "C:\\Program Files\\Defender\\MsMpEng.exe"
      },
      "thumbprint": "",
      "sha256": "",
      "running": true
    }
  }'
```

### File Check

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "MDM Profile Installed",
    "type": "file",
    "match": [{"platform": "mac"}],
    "input": {
      "operating_system": "mac",
      "path": {
        "mac": "/Library/Managed Preferences/com.company.mdm.plist"
      },
      "exists": true
    }
  }'
```

### Serial Number Check

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Corporate Device",
    "type": "serial_number",
    "input": {
      "serial_numbers": ["ABC123", "DEF456", "GHI789"]
    }
  }'
```

## EDR/MDM Integrations

### CrowdStrike

```bash
# Create integration
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture/integration" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "CrowdStrike Falcon",
    "type": "crowdstrike",
    "config": {
      "api_url": "https://api.crowdstrike.com",
      "client_id": "YOUR_CLIENT_ID",
      "client_secret": "YOUR_CLIENT_SECRET"
    }
  }'

# Create posture check using integration
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "CrowdStrike Protected",
    "type": "crowdstrike_s2s",
    "input": {
      "connection_id": "INTEGRATION_UUID",
      "state": "online",
      "last_seen": "24h"
    }
  }'
```

### Microsoft Intune

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture/integration" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Microsoft Intune",
    "type": "intune",
    "config": {
      "client_id": "AZURE_APP_ID",
      "client_secret": "AZURE_CLIENT_SECRET",
      "tenant_id": "AZURE_TENANT_ID"
    }
  }'
```

### SentinelOne

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture/integration" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "SentinelOne",
    "type": "sentinelone_s2s",
    "config": {
      "api_url": "https://usea1.sentinelone.net",
      "api_key": "YOUR_API_KEY"
    }
  }'
```

## Using Posture in Access Policies

### Require Posture Check

```json
{
  "name": "Secure Access",
  "decision": "allow",
  "include": [
    {"email_domain": {"domain": "company.com"}}
  ],
  "require": [
    {"device_posture": {"integration_uid": "disk-encryption-check-uuid"}},
    {"device_posture": {"integration_uid": "firewall-check-uuid"}}
  ]
}
```

### Require WARP + Posture

```json
{
  "require": [
    {"warp": {}},
    {"device_posture": {"integration_uid": "crowdstrike-check-uuid"}}
  ]
}
```

## List Posture Checks

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture" | \
  jq '.result[] | {id, name, type}'
```

## Check Device Posture Status

```bash
# Get device posture results
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/$DEVICE_ID" | \
  jq '.result.device_posture'
```

## Schedule

Posture checks run on a schedule:
- `5m` - Every 5 minutes (recommended for critical checks)
- `15m` - Every 15 minutes
- `1h` - Every hour

```json
{
  "schedule": "5m"
}
```

## Best Practices

1. **Start with critical checks** - Disk encryption, OS version
2. **Test before enforcing** - Create policy without `require` first
3. **Use appropriate schedule** - Balance security vs. performance
4. **Document requirements** - Communicate to users what's required
5. **Provide remediation guidance** - Help users fix posture issues

## See Also

- [Policies](./policies.md) - Device settings policies
- [Access Groups](../access/groups.md) - Using posture in groups
- [Gotchas](./gotchas.md) - Common posture issues
