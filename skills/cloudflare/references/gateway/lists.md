# Gateway Lists

Custom allow/block lists for domains, URLs, IPs, and more.

## Overview

Lists provide reusable collections for Gateway policies:
- **Domain lists** - FQDNs for DNS policies
- **URL lists** - Full URLs for HTTP policies
- **IP lists** - IPs/CIDRs for network policies
- **Serial lists** - Device serial numbers
- **Email lists** - Email addresses

## List Types

| Type | Use Case | Example |
|------|----------|---------|
| `DOMAIN` | DNS filtering | `malware.com` |
| `URL` | HTTP filtering | `https://site.com/path` |
| `IP` | Network filtering | `10.0.0.0/8` |
| `SERIAL` | Device serial numbers | `ABC123DEF` |
| `EMAIL` | User emails | `user@example.com` |

## Creating Lists

### Domain List

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "blocked_domains",
    "description": "Domains to block",
    "type": "DOMAIN",
    "items": [
      {"value": "malware.example.com"},
      {"value": "phishing.example.com"},
      {"value": "*.gambling.com"}
    ]
  }'
```

### URL List

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "blocked_urls",
    "description": "Specific URLs to block",
    "type": "URL",
    "items": [
      {"value": "https://example.com/malware.exe"},
      {"value": "https://badsite.com/phishing/*"}
    ]
  }'
```

### IP List

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "internal_networks",
    "description": "Internal IP ranges",
    "type": "IP",
    "items": [
      {"value": "10.0.0.0/8"},
      {"value": "172.16.0.0/12"},
      {"value": "192.168.0.0/16"}
    ]
  }'
```

## Managing List Items

### Add Items

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

### Remove Items

```bash
curl -X PATCH "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists/$LIST_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "remove": ["domain-to-remove.com"]
  }'
```

### Replace All Items

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists/$LIST_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "blocked_domains",
    "items": [
      {"value": "only-these.com"},
      {"value": "domains-remain.com"}
    ]
  }'
```

### Get List Items

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists/$LIST_ID/items"
```

## Using Lists in Policies

### DNS Policy with Domain List

```json
{
  "name": "Block Custom Domains",
  "action": "block",
  "filters": ["dns"],
  "traffic": "dns.fqdn in $blocked_domains"
}
```

### HTTP Policy with URL List

```json
{
  "name": "Block Specific URLs",
  "action": "block",
  "filters": ["http"],
  "traffic": "http.request.uri in $blocked_urls"
}
```

### Network Policy with IP List

```json
{
  "name": "Allow Internal Traffic",
  "action": "allow",
  "filters": ["l4"],
  "traffic": "net.dst.ip in $internal_networks"
}
```

### Combine List with Other Conditions

```json
{
  "name": "Block Except for Engineering",
  "action": "block",
  "filters": ["dns"],
  "traffic": "dns.fqdn in $risky_domains and not any(identity.groups[*] in {\"Engineering\"})"
}
```

## Bulk Import

### From File (CSV)

```bash
# Prepare CSV (one item per line)
cat > domains.csv << EOF
malware1.com
malware2.com
phishing1.com
EOF

# Read and create list
ITEMS=$(cat domains.csv | jq -R -s 'split("\n") | map(select(length > 0)) | map({"value": .})')

curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"imported_domains\",
    \"type\": \"DOMAIN\",
    \"items\": $ITEMS
  }"
```

### Sync from External Source

```bash
#!/bin/bash
# Sync list from threat intel feed
DOMAINS=$(curl -s "https://threatfeed.example.com/domains.txt" | head -1000 | jq -R -s 'split("\n") | map(select(length > 0)) | map({"value": .})')

curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists/$LIST_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"threat_intel_domains\",
    \"items\": $DOMAINS
  }"
```

## Common List Patterns

### Allow List (Safe Sites)

```json
{
  "name": "allowed_domains",
  "type": "DOMAIN",
  "items": [
    {"value": "company.com"},
    {"value": "*.microsoft.com"},
    {"value": "*.google.com"}
  ]
}
```

**Policy:**
```json
{
  "name": "Allow Safe Domains",
  "action": "allow",
  "filters": ["dns"],
  "traffic": "dns.fqdn in $allowed_domains",
  "precedence": 1
}
```

### Block List (Risky Sites)

```json
{
  "name": "blocked_domains",
  "type": "DOMAIN",
  "items": [
    {"value": "*.torrent.com"},
    {"value": "proxy*.com"},
    {"value": "vpn-free.com"}
  ]
}
```

### Corporate Assets

```json
{
  "name": "corporate_ips",
  "type": "IP",
  "items": [
    {"value": "203.0.113.0/24", "description": "NYC Office"},
    {"value": "198.51.100.0/24", "description": "LA Office"}
  ]
}
```

## Wildcard Support

- Domain lists support wildcards: `*.example.com`
- URL lists support wildcards: `https://example.com/*`
- IP lists support CIDR notation: `10.0.0.0/8`

## Limits

| Limit | Value |
|-------|-------|
| Lists per account | 100 |
| Items per list | 5,000 (Standard), 20,000 (Enterprise) |
| Item value length | 255 characters |

## See Also

- [DNS Policies](./dns-policies.md) - Using lists in DNS rules
- [HTTP Policies](./http-policies.md) - Using lists in HTTP rules
- [API](./api.md) - Full list API reference
