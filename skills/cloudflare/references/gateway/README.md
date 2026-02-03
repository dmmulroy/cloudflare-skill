# Cloudflare Gateway

Secure Web Gateway for DNS and HTTP filtering, threat protection, and data loss prevention.

## Overview

Cloudflare Gateway provides:
- **DNS filtering** - Block domains by category, threat, or custom lists
- **HTTP filtering** - Inspect and filter HTTP/HTTPS traffic
- **Network filtering** - Control Layer 4 traffic
- **Threat protection** - Block malware, phishing, C2 domains
- **Data loss prevention** - Detect sensitive data in traffic

**Architecture:**
- DNS policies: WARP client or DNS endpoint → Gateway DNS resolver
- HTTP policies: WARP client (with TLS inspection) → Gateway proxy

## Quick Start

### 1. Create DNS Policy

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/rules" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Block Malware Domains",
    "enabled": true,
    "action": "block",
    "traffic": "dns",
    "filters": ["dns"],
    "rule_settings": {
      "block_page_enabled": true,
      "block_reason": "This domain is blocked due to security threats"
    },
    "conditions": [
      {
        "type": "traffic",
        "expression": {
          "any": {
            "security_category": ["malware", "phishing", "spyware"]
          }
        }
      }
    ]
  }'
```

### 2. Create HTTP Policy

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/rules" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Block File Uploads to Cloud Storage",
    "enabled": true,
    "action": "block",
    "traffic": "http",
    "filters": ["http"],
    "conditions": [
      {
        "type": "traffic",
        "expression": {
          "and": [
            {"http.request.method": "PUT"},
            {"any": {"application": ["Google Drive", "Dropbox", "OneDrive"]}}
          ]
        }
      }
    ]
  }'
```

## Policy Types

| Type | Traffic | Use Case |
|------|---------|----------|
| DNS | `dns` | Block domains, categories, threats |
| HTTP | `http` | Inspect web traffic, block uploads, DLP |
| Network | `l4` | Control TCP/UDP connections |

## Actions

| Action | Effect |
|--------|--------|
| `allow` | Permit traffic |
| `block` | Block with optional block page |
| `isolate` | Open in remote browser (Browser Isolation) |
| `do_not_inspect` | Skip TLS inspection |
| `do_not_scan` | Skip antivirus scanning |
| `egress` | Route through specific egress IP |
| `audit_ssh` | Log SSH commands |

## Traffic Selectors (Conditions)

### DNS Policies

```json
// Block specific domain
{"dns.fqdn": "malware.example.com"}

// Block domain pattern
{"dns.fqdn_regex": ".*\\.torrent\\..*"}

// Block by category
{"any": {"content_category": ["gambling", "adult_content"]}}

// Block by security category
{"any": {"security_category": ["malware", "phishing"]}}

// Match resolved IP
{"dns.resolved_ip": "192.168.1.0/24"}
```

### HTTP Policies

```json
// Match URL
{"http.request.uri": "https://example.com/sensitive/*"}

// Match host
{"http.host": "dropbox.com"}

// Match by application
{"any": {"application": ["Slack", "Discord"]}}

// Match by file type (uploads)
{"http.upload.file_type": ["exe", "dll", "bat"]}

// Match HTTP method
{"http.request.method": "POST"}

// Match by user identity
{"identity.email": "*@company.com"}

// Match by device posture
{"device_posture.check_passed": "disk-encryption-uuid"}
```

### Network Policies

```json
// Match destination port
{"net.dst_port": 22}

// Match destination IP
{"net.dst_ip": "10.0.0.0/8"}

// Match protocol
{"net.protocol": "TCP"}
```

## Decision Flow

1. Evaluate rules in precedence order (lowest first)
2. First matching rule wins
3. If no rules match, traffic is allowed by default

## Block Page

Configure a custom block page:

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/configuration" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "settings": {
      "block_page": {
        "enabled": true,
        "name": "Company Security Block",
        "mailto_subject": "Request Access",
        "mailto_address": "security@company.com",
        "header_text": "Access Blocked",
        "footer_text": "Contact IT if you need access"
      }
    }
  }'
```

## In This Reference

- [DNS Policies](./dns-policies.md) - DNS filtering details
- [HTTP Policies](./http-policies.md) - HTTP/HTTPS filtering
- [Lists](./lists.md) - Custom allow/block lists
- [Locations](./locations.md) - DNS endpoint configuration
- [API](./api.md) - Full API reference
- [Gotchas](./gotchas.md) - Common issues

## See Also

- [Devices](../devices/README.md) - WARP client deployment
- [DLP](../dlp/README.md) - Data loss prevention
- [Access](../access/README.md) - Application access control
