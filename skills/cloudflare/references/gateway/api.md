# Gateway API Reference

Base URL: `https://api.cloudflare.com/client/v4/accounts/{account_id}`

## Rules (Policies)

### List Rules

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/rules" | jq '.result'
```

### Get Rule

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/rules/$RULE_ID"
```

### Create DNS Rule

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/rules" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Block Social Media",
    "description": "Block social media during work hours",
    "enabled": true,
    "action": "block",
    "filters": ["dns"],
    "traffic": "any(dns.content_category[*] in {\"Social Networks\"})",
    "rule_settings": {
      "block_page_enabled": true,
      "block_reason": "Social media is blocked during work hours"
    },
    "precedence": 100
  }'
```

### Create HTTP Rule

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/rules" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Block File Uploads",
    "enabled": true,
    "action": "block",
    "filters": ["http"],
    "traffic": "any(http.upload.file_type[*] in {\"exe\" \"dll\" \"bat\"})",
    "rule_settings": {
      "block_page_enabled": true
    },
    "precedence": 50
  }'
```

### Create Network Rule

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/rules" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Block SSH to External",
    "enabled": true,
    "action": "block",
    "filters": ["l4"],
    "traffic": "net.dst.port == 22 and not net.dst.ip in $internal_networks",
    "precedence": 25
  }'
```

### Update Rule

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/rules/$RULE_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Rule Name",
    "enabled": false
  }'
```

### Delete Rule

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/rules/$RULE_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Lists

### List All Lists

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists" | jq '.result'
```

### Create List

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "blocked_domains",
    "description": "Custom blocked domains",
    "type": "DOMAIN",
    "items": [
      {"value": "malware.example.com"},
      {"value": "phishing.example.com"}
    ]
  }'
```

**List types:** `DOMAIN`, `URL`, `IP`, `SERIAL`, `EMAIL`, `FILE`

### Get List Items

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists/$LIST_ID/items"
```

### Add Items to List

```bash
curl -X PATCH "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists/$LIST_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "append": [
      {"value": "newdomain.com"},
      {"value": "anotherdomain.com"}
    ]
  }'
```

### Remove Items from List

```bash
curl -X PATCH "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists/$LIST_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "remove": ["malware.example.com"]
  }'
```

### Delete List

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists/$LIST_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Locations (DNS Endpoints)

### List Locations

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/locations"
```

### Create Location

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/locations" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Office NYC",
    "client_default": false,
    "networks": [
      {"network": "203.0.113.0/24"}
    ]
  }'
```

### Update Location

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/locations/$LOCATION_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Office NYC Updated",
    "networks": [
      {"network": "203.0.113.0/24"},
      {"network": "198.51.100.0/24"}
    ]
  }'
```

### Delete Location

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/locations/$LOCATION_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Categories

### List Content Categories

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/categories" | \
  jq '.result[] | {id, name, class}'
```

### List Application Types

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/app_types" | \
  jq '.result[] | {id, name}'
```

## Configuration

### Get Gateway Configuration

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/configuration"
```

### Update Configuration

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/configuration" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "settings": {
      "antivirus": {
        "enabled_download_phase": true,
        "enabled_upload_phase": true,
        "fail_closed": false
      },
      "tls_decrypt": {
        "enabled": true
      },
      "activity_log": {
        "enabled": true
      },
      "block_page": {
        "enabled": true,
        "name": "Security Block",
        "header_text": "This site is blocked"
      }
    }
  }'
```

## Certificates (TLS Inspection)

### List Certificates

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/certificates"
```

### Create Certificate

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/certificates" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "validity_period_days": 1825
  }'
```

### Activate Certificate

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/certificates/$CERT_ID/activate" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Logging

### Get Logging Configuration

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/logging"
```

### Update Logging

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/logging" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "settings_by_rule_type": {
      "dns": {
        "log_all": true
      },
      "http": {
        "log_all": true
      },
      "l4": {
        "log_all": true
      }
    },
    "redact_pii": true
  }'
```

## Proxy Endpoints

### List Proxy Endpoints

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/proxy_endpoints"
```

### Create Proxy Endpoint

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/proxy_endpoints" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "PAC File Endpoint",
    "networks": [
      {"network": "10.0.0.0/8"}
    ]
  }'
```
