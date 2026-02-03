# Cloudflare DLP (Data Loss Prevention)

Detect and protect sensitive data in HTTP traffic.

## Overview

Cloudflare DLP provides:
- **Content inspection** - Detect sensitive data in uploads/downloads
- **Predefined profiles** - PII, PCI, HIPAA patterns
- **Custom patterns** - Regex-based detection
- **Integration** - Works with Gateway HTTP policies

**Requires:** Gateway with TLS inspection enabled

## How It Works

```
User → WARP → Gateway HTTP Policy → DLP Scan → Allow/Block
```

1. HTTP traffic flows through Gateway
2. TLS inspection decrypts HTTPS
3. DLP profiles scan request/response bodies
4. Match triggers configured action (block, log)

## Quick Start

### 1. Enable TLS Inspection

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/configuration" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"settings": {"tls_decrypt": {"enabled": true}}}'
```

### 2. Create DLP Profile

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/profiles/custom" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Credit Card Detection",
    "entries": [
      {
        "name": "Credit Card Numbers",
        "enabled": true,
        "pattern": {
          "regex": "\\b(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14}|3[47][0-9]{13})\\b",
          "validation": "luhn"
        }
      }
    ]
  }'
```

### 3. Create Gateway Policy with DLP

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/rules" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Block Credit Card Uploads",
    "enabled": true,
    "action": "block",
    "filters": ["http"],
    "traffic": "http.request.method == \"POST\" and any(dlp.profiles[*] in {\"PROFILE_UUID\"})",
    "rule_settings": {
      "block_page_enabled": true,
      "block_reason": "Upload blocked: sensitive data detected"
    }
  }'
```

## Predefined Profiles

Cloudflare provides ready-to-use profiles:

| Profile | Detects |
|---------|---------|
| Financial | Credit cards, bank accounts |
| PII | SSN, passport, driver's license |
| Credentials | API keys, passwords, tokens |
| Source Code | Code patterns, secrets in code |
| Health | Medical record numbers (HIPAA) |

### Enable Predefined Profile

```bash
# List predefined profiles
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/profiles/predefined"

# Enable entries in a predefined profile
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/profiles/predefined/$PROFILE_ID/config" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "entries": [
      {"id": "ENTRY_ID_1", "enabled": true},
      {"id": "ENTRY_ID_2", "enabled": true}
    ]
  }'
```

## Custom Profiles

### Create Custom Profile

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/profiles/custom" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Company Confidential",
    "entries": [
      {
        "name": "Project Codename",
        "enabled": true,
        "pattern": {
          "regex": "\\b(PROJECT[_-]?FALCON|CLASSIFIED)\\b",
          "validation": "regex"
        }
      },
      {
        "name": "Internal Document ID",
        "enabled": true,
        "pattern": {
          "regex": "\\bDOC-[A-Z]{3}-[0-9]{6}\\b",
          "validation": "regex"
        }
      }
    ]
  }'
```

### Pattern Validation Types

| Type | Description |
|------|-------------|
| `regex` | Standard regex match |
| `luhn` | Credit card checksum validation |

## Detection Locations

DLP can scan:
- HTTP request body (uploads)
- HTTP response body (downloads)
- File contents
- Form data
- JSON payloads

## Datasets (Exact Match)

For known sensitive values (employee IDs, account numbers):

### Create Dataset

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/datasets" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Employee IDs",
    "description": "List of employee IDs to protect"
  }'
```

### Upload Data to Dataset

```bash
# Get upload URL
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/datasets/$DATASET_ID/upload" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"

# Upload CSV (one value per line)
curl -X PUT "$UPLOAD_URL" \
  -H "Content-Type: text/csv" \
  --data-binary @employee_ids.csv
```

### Use Dataset in Policy

```json
{
  "traffic": "any(dlp.datasets[*] in {\"DATASET_UUID\"})"
}
```

## Gateway Integration

### Block on DLP Match

```json
{
  "name": "Block Sensitive Data Uploads",
  "action": "block",
  "filters": ["http"],
  "traffic": "http.request.method in {\"POST\" \"PUT\"} and any(dlp.profiles[*] in {\"pii-profile-uuid\" \"financial-profile-uuid\"})"
}
```

### Log Only (Audit Mode)

```json
{
  "name": "Log Sensitive Data",
  "action": "allow",
  "filters": ["http"],
  "traffic": "any(dlp.profiles[*] in {\"pii-profile-uuid\"})",
  "rule_settings": {
    "add_headers": {
      "X-DLP-Match": "true"
    }
  }
}
```

### Block Uploads, Allow Downloads

```json
{
  "name": "Block PII Uploads Only",
  "action": "block",
  "filters": ["http"],
  "traffic": "http.request.method in {\"POST\" \"PUT\"} and any(dlp.profiles[*] in {\"pii-uuid\"})"
}
```

## Payload Logging

Capture matched content for investigation:

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/dlp/payload_log" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "public_key": "YOUR_PUBLIC_KEY_FOR_ENCRYPTION"
  }'
```

**Note:** Payloads are encrypted with your public key for security.

## Common Patterns

### Credit Cards

```regex
\b(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14}|3[47][0-9]{13}|6(?:011|5[0-9]{2})[0-9]{12})\b
```

### US SSN

```regex
\b\d{3}-\d{2}-\d{4}\b
```

### Email Addresses

```regex
\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b
```

### API Keys (Generic)

```regex
\b[A-Za-z0-9]{32,}\b
```

### AWS Access Keys

```regex
\bAKIA[0-9A-Z]{16}\b
```

## In This Reference

- [Profiles](./profiles.md) - Profile configuration details
- [API](./api.md) - Full API reference

## See Also

- [Gateway HTTP Policies](../gateway/http-policies.md) - Using DLP in policies
- [Gateway Gotchas](../gateway/gotchas.md) - TLS inspection requirements
