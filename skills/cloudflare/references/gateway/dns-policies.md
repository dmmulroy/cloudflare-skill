# Gateway DNS Policies

Filter and control DNS resolution for your organization.

## Overview

DNS policies evaluate DNS queries and can:
- Block malicious domains
- Filter by content category
- Allow/block custom domain lists
- Override DNS responses
- Log DNS activity

## Policy Structure

```json
{
  "name": "Block Malware",
  "enabled": true,
  "action": "block",
  "filters": ["dns"],
  "traffic": "any(dns.security_category[*] in {\"Malware\" \"Phishing\" \"Spyware\"})",
  "rule_settings": {
    "block_page_enabled": true,
    "block_reason": "Blocked for security reasons"
  },
  "precedence": 10
}
```

## Traffic Expressions

### Domain Matching

```
# Exact domain
dns.fqdn == "example.com"

# Subdomain matching (includes *.example.com)
dns.fqdn matches ".*\\.example\\.com$"

# Domain in list
dns.fqdn in $blocked_domains

# Ends with
dns.fqdn matches ".*\\.ru$"
```

### Content Categories

```
# Single category
any(dns.content_category[*] in {"Gambling"})

# Multiple categories
any(dns.content_category[*] in {"Gambling" "Adult Content" "Dating"})

# Exclude category
not any(dns.content_category[*] in {"Business"})
```

**Common categories:**
- `Adult Content`, `Dating`, `Gambling`
- `Social Networks`, `Streaming Media`, `Gaming`
- `News`, `Shopping`, `Technology`
- `Education`, `Government`, `Health`

### Security Categories

```
# Block security threats
any(dns.security_category[*] in {"Malware" "Phishing" "Spyware" "Botnet"})

# Block command & control
any(dns.security_category[*] in {"Command and Control"})
```

**Security categories:**
- `Malware`, `Phishing`, `Spyware`
- `Command and Control`, `Botnet`
- `Cryptomining`, `DGA Domains`
- `Newly Seen Domains`, `Parked Domains`

### Resolved IP Matching

```
# Block if resolves to private IP
dns.resolved_ip in {10.0.0.0/8 172.16.0.0/12 192.168.0.0/16}

# Block specific IP
dns.resolved_ip == "93.184.216.34"
```

### DNS Record Type

```
# Only MX records
dns.query_rtype == "MX"

# Block TXT lookups (often used for data exfiltration)
dns.query_rtype == "TXT"
```

### Identity Conditions

```
# By user email
identity.email == "user@example.com"

# By user group
any(identity.groups[*] in {"Engineering"})

# By device posture
device_posture.checks.passed[*] == "disk-encryption-uuid"
```

### Location Conditions

```
# Specific office location
identity.location.id == "location-uuid"

# By source IP
identity.source_ip in {203.0.113.0/24}
```

## Common Policies

### Block All Security Threats

```json
{
  "name": "Block Security Threats",
  "enabled": true,
  "action": "block",
  "filters": ["dns"],
  "traffic": "any(dns.security_category[*] in {\"Malware\" \"Phishing\" \"Spyware\" \"Botnet\" \"Command and Control\" \"Cryptomining\"})",
  "precedence": 1
}
```

### Block Adult Content

```json
{
  "name": "Block Adult Content",
  "enabled": true,
  "action": "block",
  "filters": ["dns"],
  "traffic": "any(dns.content_category[*] in {\"Adult Content\" \"Adult Themes\"})",
  "precedence": 10
}
```

### Block Social Media (Except for Marketing Team)

```json
{
  "name": "Allow Marketing - Social Media",
  "enabled": true,
  "action": "allow",
  "filters": ["dns"],
  "traffic": "any(dns.content_category[*] in {\"Social Networks\"}) and any(identity.groups[*] in {\"Marketing\"})",
  "precedence": 5
},
{
  "name": "Block Social Media",
  "enabled": true,
  "action": "block",
  "filters": ["dns"],
  "traffic": "any(dns.content_category[*] in {\"Social Networks\"})",
  "precedence": 10
}
```

### Block Newly Seen Domains

```json
{
  "name": "Block New Domains",
  "enabled": true,
  "action": "block",
  "filters": ["dns"],
  "traffic": "any(dns.security_category[*] in {\"Newly Seen Domains\"})",
  "rule_settings": {
    "block_page_enabled": true,
    "block_reason": "New domain detected - please contact IT if this is needed"
  },
  "precedence": 5
}
```

### Allow Specific Domains (Bypass)

```json
{
  "name": "Allow Business Critical",
  "enabled": true,
  "action": "allow",
  "filters": ["dns"],
  "traffic": "dns.fqdn in $allowed_domains",
  "precedence": 1
}
```

### Block DNS over HTTPS/TLS (DoH/DoT) Providers

```json
{
  "name": "Block DoH Providers",
  "enabled": true,
  "action": "block",
  "filters": ["dns"],
  "traffic": "dns.fqdn in {\"dns.google\" \"cloudflare-dns.com\" \"dns.quad9.net\" \"doh.opendns.com\"}",
  "precedence": 2
}
```

## Using Lists

### Create a Domain List

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/lists" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "blocked_domains",
    "type": "DOMAIN",
    "items": [
      {"value": "badsite.com"},
      {"value": "malware.example.com"}
    ]
  }'
```

### Use List in Policy

```json
{
  "traffic": "dns.fqdn in $blocked_domains"
}
```

## DNS Override

Redirect DNS resolution to a different IP:

```json
{
  "name": "Override Internal DNS",
  "enabled": true,
  "action": "allow",
  "filters": ["dns"],
  "traffic": "dns.fqdn == \"internal.example.com\"",
  "rule_settings": {
    "override_ips": ["10.0.0.100"]
  }
}
```

## SafeSearch Enforcement

Force SafeSearch on search engines:

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/configuration" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "settings": {
      "safesearch": {
        "enabled": true
      },
      "youtube_restricted_mode": {
        "enabled": true
      }
    }
  }'
```

## Precedence

- Lower number = higher priority
- First matching rule wins
- Use gaps (10, 20, 30) to allow future insertions

**Recommended order:**
1. `1-9`: Critical allows (bypass for essential services)
2. `10-49`: Security blocks (malware, phishing)
3. `50-99`: Content filtering (categories)
4. `100+`: Default policies

## See Also

- [HTTP Policies](./http-policies.md) - HTTP traffic filtering
- [Lists](./lists.md) - Custom domain lists
- [Gotchas](./gotchas.md) - Common DNS policy issues
