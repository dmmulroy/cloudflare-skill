# Gateway HTTP Policies

Inspect and filter HTTP/HTTPS traffic with deep content inspection.

## Overview

HTTP policies require:
- **WARP client** with Gateway proxy enabled
- **TLS inspection** for HTTPS traffic (optional but recommended)
- **Root CA** installed on devices for TLS inspection

## Policy Structure

```json
{
  "name": "Block Uploads to Cloud Storage",
  "enabled": true,
  "action": "block",
  "filters": ["http"],
  "traffic": "http.request.method == \"PUT\" and any(http.application[*] in {\"Google Drive\" \"Dropbox\" \"OneDrive\"})",
  "rule_settings": {
    "block_page_enabled": true,
    "block_reason": "File uploads to cloud storage are not allowed"
  },
  "precedence": 50
}
```

## Actions

| Action | Description |
|--------|-------------|
| `allow` | Permit request |
| `block` | Block with optional block page |
| `isolate` | Open in remote browser isolation |
| `do_not_inspect` | Skip TLS decryption (pass through) |
| `do_not_scan` | Skip antivirus scanning |

## Traffic Expressions

### URL/Host Matching

```
# Exact host
http.host == "dropbox.com"

# Host contains
http.host contains "google"

# URL path
http.request.uri.path == "/upload"

# Full URL
http.request.uri matches "https://example\\.com/api/.*"

# URL in list
http.host in $blocked_hosts
```

### HTTP Method

```
# Block uploads
http.request.method in {"PUT" "POST"}

# Block specific methods
http.request.method == "DELETE"
```

### Application Detection

```
# Single application
any(http.application[*] in {"Slack"})

# Multiple applications
any(http.application[*] in {"Slack" "Discord" "Teams"})

# Application category
any(http.application_type[*] in {"File Sharing"})
```

**Common applications:** `Slack`, `Discord`, `Teams`, `Zoom`, `Google Drive`, `Dropbox`, `OneDrive`, `Box`, `GitHub`, `GitLab`, `AWS`, `Azure`, `GCP`

### File Type Detection (Uploads/Downloads)

```
# Block executable uploads
any(http.upload.file_type[*] in {"exe" "dll" "bat" "ps1" "msi"})

# Block executable downloads
any(http.download.file_type[*] in {"exe" "dll" "bat"})

# Block by MIME type
http.request.content_type contains "application/x-executable"
```

### Content Categories

```
# Block by category
any(http.content_category[*] in {"Gambling" "Adult Content"})

# Security categories
any(http.security_category[*] in {"Malware" "Phishing"})
```

### Identity Conditions

```
# By user email
identity.email == "user@example.com"

# By email domain
identity.email matches ".*@contractor\\.com$"

# By group
any(identity.groups[*] in {"Engineering"})

# By device posture
device_posture.checks.passed[*] == "uuid"
```

### Request Headers

```
# Check user agent
http.request.headers["user-agent"] contains "curl"

# Check authorization
not http.request.headers["authorization"]
```

## Common Policies

### Block File Uploads to Personal Cloud Storage

```json
{
  "name": "Block Personal Cloud Uploads",
  "enabled": true,
  "action": "block",
  "filters": ["http"],
  "traffic": "http.request.method in {\"PUT\" \"POST\"} and any(http.application[*] in {\"Google Drive\" \"Dropbox\" \"OneDrive\" \"iCloud\" \"Box\"})",
  "precedence": 50
}
```

### Block Executable Downloads

```json
{
  "name": "Block Executable Downloads",
  "enabled": true,
  "action": "block",
  "filters": ["http"],
  "traffic": "any(http.download.file_type[*] in {\"exe\" \"dll\" \"msi\" \"bat\" \"ps1\" \"vbs\" \"jar\"})",
  "rule_settings": {
    "block_page_enabled": true,
    "block_reason": "Downloading executables is blocked. Contact IT for approved software."
  },
  "precedence": 20
}
```

### Isolate Social Media

```json
{
  "name": "Isolate Social Media",
  "enabled": true,
  "action": "isolate",
  "filters": ["http"],
  "traffic": "any(http.content_category[*] in {\"Social Networks\"})",
  "rule_settings": {
    "isolate_settings": {
      "disable_copy_paste": true,
      "disable_download": true,
      "disable_upload": true
    }
  },
  "precedence": 30
}
```

### Do Not Inspect Banking Sites

```json
{
  "name": "Bypass Inspection - Banking",
  "enabled": true,
  "action": "do_not_inspect",
  "filters": ["http"],
  "traffic": "any(http.content_category[*] in {\"Financial Services\"})",
  "precedence": 5
}
```

### Block Shadow IT

```json
{
  "name": "Block Unapproved SaaS",
  "enabled": true,
  "action": "block",
  "filters": ["http"],
  "traffic": "any(http.application_type[*] in {\"File Sharing\" \"Cloud Storage\"}) and not any(http.application[*] in {\"SharePoint\" \"OneDrive\"})",
  "precedence": 40
}
```

### Allow Uploads Only to Corporate Domain

```json
{
  "name": "Block External Uploads",
  "enabled": true,
  "action": "block",
  "filters": ["http"],
  "traffic": "http.request.method == \"POST\" and not http.host matches \".*\\.company\\.com$\"",
  "precedence": 60
}
```

## TLS Inspection

### Enable TLS Inspection

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/configuration" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "settings": {
      "tls_decrypt": {
        "enabled": true
      }
    }
  }'
```

### Download Root CA

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/certificates" | \
  jq -r '.result[] | select(.active == true) | .certificate'
```

### Bypass Inspection for Specific Hosts

```json
{
  "name": "Do Not Inspect - Banking",
  "enabled": true,
  "action": "do_not_inspect",
  "filters": ["http"],
  "traffic": "http.host in $banking_domains",
  "precedence": 1
}
```

## Browser Isolation

### Full Isolation

```json
{
  "action": "isolate",
  "rule_settings": {
    "isolate_settings": {
      "disable_copy_paste": true,
      "disable_download": true,
      "disable_upload": true,
      "disable_printing": true,
      "disable_keyboard": false
    }
  }
}
```

### Partial Isolation (Read-only)

```json
{
  "action": "isolate",
  "rule_settings": {
    "isolate_settings": {
      "disable_copy_paste": false,
      "disable_download": true,
      "disable_upload": true
    }
  }
}
```

## Antivirus Scanning

### Enable AV Scanning

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
      }
    }
  }'
```

**Note:** `fail_closed: true` blocks files if scanning fails.

## See Also

- [DNS Policies](./dns-policies.md) - DNS-level filtering
- [DLP](../dlp/README.md) - Data loss prevention in HTTP traffic
- [Gotchas](./gotchas.md) - Common HTTP policy issues
