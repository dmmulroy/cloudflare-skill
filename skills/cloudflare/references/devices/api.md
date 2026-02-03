# Devices API Reference

Base URL: `https://api.cloudflare.com/client/v4/accounts/{account_id}`

## Devices

### List All Devices

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices" | jq '.result'
```

### Search Devices by User

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices?search=user@example.com"
```

### Get Device Details

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/$DEVICE_ID"
```

### Revoke Device

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/$DEVICE_ID/revoke" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

### List Physical Devices

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/physical-devices"
```

## Device Posture

### List Posture Rules

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture" | jq '.result'
```

### Create Posture Rule

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Disk Encryption",
    "type": "disk_encryption",
    "description": "Requires full disk encryption",
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

### Update Posture Rule

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture/$RULE_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Name",
    "schedule": "15m"
  }'
```

### Delete Posture Rule

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture/$RULE_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Posture Integrations

### List Integrations

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture/integration"
```

### Create Integration

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture/integration" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "CrowdStrike",
    "type": "crowdstrike",
    "config": {
      "api_url": "https://api.crowdstrike.com",
      "client_id": "YOUR_CLIENT_ID",
      "client_secret": "YOUR_CLIENT_SECRET"
    }
  }'
```

### Update Integration

```bash
curl -X PATCH "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture/integration/$INTEGRATION_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "config": {
      "client_secret": "NEW_SECRET"
    }
  }'
```

### Delete Integration

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture/integration/$INTEGRATION_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Device Policies

### Get Default Policy

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy"
```

### Update Default Policy

```bash
curl -X PATCH "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "gateway_unique_id": "YOUR_GATEWAY_ID",
    "disable_auto_fallback": true,
    "captive_portal": 180,
    "support_url": "https://help.company.com"
  }'
```

### List All Policies

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policies"
```

### Create Device Policy

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Engineering Policy",
    "description": "Policy for engineering team",
    "match": "any(identity.groups[*] in {\"Engineering\"})",
    "precedence": 100,
    "gateway_unique_id": "YOUR_GATEWAY_ID",
    "service_mode_v2": {
      "mode": "warp"
    }
  }'
```

### Update Device Policy

```bash
curl -X PATCH "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy/$POLICY_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Policy Name"
  }'
```

### Delete Device Policy

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy/$POLICY_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Split Tunnel Configuration

### Get Default Exclude List

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy/exclude"
```

### Update Default Exclude List

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy/exclude" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[
    {"address": "192.168.1.0/24", "description": "Local network"},
    {"host": "local-only.example.com", "description": "Local service"}
  ]'
```

### Get Default Include List

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy/include"
```

### Update Default Include List

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy/include" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[
    {"address": "10.0.0.0/8", "description": "Internal network"},
    {"address": "172.16.0.0/12", "description": "VPC"}
  ]'
```

### Get Policy-Specific Split Tunnel

```bash
# Exclude list
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy/$POLICY_ID/exclude"

# Include list
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy/$POLICY_ID/include"
```

## Fallback Domains

### Get Default Fallback Domains

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy/fallback_domains"
```

### Update Default Fallback Domains

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy/fallback_domains" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[
    {"suffix": "internal.company.com", "dns_server": ["10.0.0.53"]},
    {"suffix": "corp.local", "dns_server": ["10.0.0.53", "10.0.1.53"]}
  ]'
```

## Networks

### List Device Networks

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/networks"
```

### Create Device Network

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/networks" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "tls",
    "name": "Corporate WiFi",
    "config": {
      "tls_sockaddr": "foobar.cloudflareaccess.com:443",
      "sha256": "SHA256_HASH_OF_CERT"
    }
  }'
```

### Delete Device Network

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/networks/$NETWORK_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Device Registrations

### List Registrations

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/registrations"
```
