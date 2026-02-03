# DLP API Reference

Base URL: `https://api.cloudflare.com/client/v4/accounts/{account_id}`

## Profiles

### List All Profiles

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/profiles" | jq '.result'
```

### List Predefined Profiles

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/profiles/predefined"
```

### Get Predefined Profile

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/profiles/predefined/$PROFILE_ID"
```

### Configure Predefined Profile

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/profiles/predefined/$PROFILE_ID/config" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "entries": [
      {"id": "entry-uuid-1", "enabled": true},
      {"id": "entry-uuid-2", "enabled": true},
      {"id": "entry-uuid-3", "enabled": false}
    ]
  }'
```

### List Custom Profiles

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/profiles/custom"
```

### Create Custom Profile

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/profiles/custom" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Sensitive Data Profile",
    "description": "Detects company sensitive information",
    "entries": [
      {
        "name": "Credit Card Numbers",
        "enabled": true,
        "pattern": {
          "regex": "\\b(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14})\\b",
          "validation": "luhn"
        }
      },
      {
        "name": "Internal Project Codes",
        "enabled": true,
        "pattern": {
          "regex": "\\bPROJ-[A-Z]{3}-[0-9]{4}\\b",
          "validation": "regex"
        }
      }
    ],
    "allowed_match_count": 0
  }'
```

### Get Custom Profile

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/profiles/custom/$PROFILE_ID"
```

### Update Custom Profile

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/profiles/custom/$PROFILE_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Profile Name",
    "entries": [
      {
        "name": "New Entry",
        "enabled": true,
        "pattern": {
          "regex": "\\bNEW-PATTERN\\b",
          "validation": "regex"
        }
      }
    ]
  }'
```

### Delete Custom Profile

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/profiles/custom/$PROFILE_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Entries

### List All Entries

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/entries"
```

### List Predefined Entries

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/entries/predefined"
```

### Get Entry

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/entries/$ENTRY_ID"
```

### Update Entry

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/entries/custom/$ENTRY_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Entry Name",
    "pattern": {
      "regex": "\\bUPDATED-PATTERN\\b",
      "validation": "regex"
    }
  }'
```

## Datasets

### List Datasets

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/datasets"
```

### Create Dataset

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/datasets" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Sensitive Keywords",
    "description": "List of sensitive keywords to detect"
  }'
```

### Get Dataset

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/datasets/$DATASET_ID"
```

### Delete Dataset

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/datasets/$DATASET_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

### Create Upload URL

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/datasets/$DATASET_ID/upload" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

### Upload Data

```bash
# Get upload URL first, then:
curl -X PUT "$UPLOAD_URL" \
  -H "Content-Type: text/csv" \
  --data-binary @data.csv
```

### Get Dataset Version

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/datasets/$DATASET_ID/versions/$VERSION"
```

## Payload Logging

### Get Payload Log Settings

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/payload_log"
```

### Configure Payload Logging

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/payload_log" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "public_key": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...\n-----END PUBLIC KEY-----"
  }'
```

## Pattern Validation

### Validate Pattern

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/patterns/validate" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "regex": "\\b[A-Z]{3}-[0-9]{6}\\b"
  }'
```

## Limits

### Get DLP Limits

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/limits"
```

## Using DLP in Gateway Policies

### Create Policy with DLP Profile

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/rules" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Block Sensitive Data",
    "enabled": true,
    "action": "block",
    "filters": ["http"],
    "traffic": "any(dlp.profiles[*] in {\"profile-uuid-1\" \"profile-uuid-2\"})",
    "rule_settings": {
      "block_page_enabled": true,
      "block_reason": "Sensitive data detected"
    }
  }'
```

### Create Policy with DLP Dataset

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/rules" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Block Known Sensitive Values",
    "enabled": true,
    "action": "block",
    "filters": ["http"],
    "traffic": "any(dlp.datasets[*] in {\"dataset-uuid\"})"
  }'
```

## Terraform

```hcl
resource "cloudflare_zero_trust_dlp_profile" "pii" {
  account_id  = var.cloudflare_account_id
  name        = "PII Detection"
  description = "Detects personally identifiable information"
  type        = "custom"

  entry {
    name    = "SSN"
    enabled = true
    pattern {
      regex      = "\\b\\d{3}-\\d{2}-\\d{4}\\b"
      validation = "regex"
    }
  }

  entry {
    name    = "Credit Card"
    enabled = true
    pattern {
      regex      = "\\b(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14})\\b"
      validation = "luhn"
    }
  }
}

resource "cloudflare_zero_trust_gateway_policy" "dlp_block" {
  account_id  = var.cloudflare_account_id
  name        = "Block PII Uploads"
  description = "Block uploads containing PII"
  action      = "block"
  enabled     = true
  filters     = ["http"]
  traffic     = "http.request.method in {\"POST\" \"PUT\"} and any(dlp.profiles[*] in {\"${cloudflare_zero_trust_dlp_profile.pii.id}\"})"

  rule_settings {
    block_page_enabled = true
    block_page_reason  = "Upload blocked: sensitive data detected"
  }
}
```
