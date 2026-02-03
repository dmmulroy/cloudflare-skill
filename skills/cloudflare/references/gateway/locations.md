# Gateway Locations

Configure DNS endpoints for offices and networks without WARP.

## Overview

Locations provide Gateway DNS filtering for:
- **Office networks** - Route office DNS through Gateway
- **IoT devices** - Filter devices that can't run WARP
- **Legacy systems** - Apply policies to non-WARP traffic

**How it works:**
1. Create location with source IP ranges
2. Configure network devices to use Gateway DNS IPs
3. Gateway applies DNS policies to matching traffic

## Quick Start

### 1. Create Location

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/locations" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "NYC Office",
    "client_default": false,
    "networks": [
      {"network": "203.0.113.0/24"}
    ]
  }'
```

### 2. Configure DNS on Network

Point DNS to Gateway endpoints:
- **IPv4:** `172.64.36.1` and `172.64.36.2`
- **IPv6:** `2606:4700:4700::1111` and `2606:4700:4700::1001`
- **DoH:** `https://<unique-id>.cloudflare-gateway.com/dns-query`
- **DoT:** `<unique-id>.cloudflare-gateway.com`

## Location Types

### Office Network

Filter all DNS from an office by source IP:

```json
{
  "name": "NYC Office",
  "networks": [
    {"network": "203.0.113.0/24"}
  ]
}
```

### Default Location

For traffic not matching any specific location:

```json
{
  "name": "Default",
  "client_default": true
}
```

### DoH Endpoint

For DNS-over-HTTPS clients:

```json
{
  "name": "Roaming Users (DoH)",
  "doh_subdomain": "abc123xyz"
}
```

Users configure: `https://abc123xyz.cloudflare-gateway.com/dns-query`

## API Operations

### List Locations

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/locations" | \
  jq '.result[] | {id, name, networks}'
```

### Get Location

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/locations/$LOCATION_ID"
```

### Create Location

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/locations" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "LA Office",
    "networks": [
      {"network": "198.51.100.0/24"}
    ],
    "ecs_support": false
  }'
```

### Update Location

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/locations/$LOCATION_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "LA Office (Updated)",
    "networks": [
      {"network": "198.51.100.0/24"},
      {"network": "198.51.101.0/24"}
    ]
  }'
```

### Delete Location

```bash
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/locations/$LOCATION_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Using Locations in Policies

### Policy for Specific Location

```json
{
  "name": "Block Social Media - NYC Office",
  "action": "block",
  "filters": ["dns"],
  "traffic": "any(dns.content_category[*] in {\"Social Networks\"}) and identity.location.id == \"location-uuid\""
}
```

### Policy for All Locations Except One

```json
{
  "traffic": "any(dns.content_category[*] in {\"Gambling\"}) and identity.location.id != \"exec-location-uuid\""
}
```

## Network Configuration

### Router/Firewall DNS Settings

Configure upstream DNS servers:
- Primary: `172.64.36.1`
- Secondary: `172.64.36.2`

### DHCP Settings

Distribute Gateway DNS to clients via DHCP.

### DNS-over-HTTPS (DoH)

For DoH-capable devices:
```
https://<location-doh-subdomain>.cloudflare-gateway.com/dns-query
```

### DNS-over-TLS (DoT)

For DoT-capable devices:
```
<location-doh-subdomain>.cloudflare-gateway.com:853
```

## IPv6 Support

Enable IPv6 DNS for the location:

```json
{
  "name": "IPv6 Enabled Office",
  "networks": [
    {"network": "203.0.113.0/24"},
    {"network": "2001:db8::/32"}
  ]
}
```

IPv6 Gateway DNS:
- `2606:4700:4700::1111`
- `2606:4700:4700::1001`

## ECS (EDNS Client Subnet)

Enable ECS for more accurate geolocation:

```json
{
  "name": "ECS Enabled Location",
  "networks": [{"network": "203.0.113.0/24"}],
  "ecs_support": true
}
```

**Note:** ECS may expose client subnet to authoritative DNS servers.

## Troubleshooting

### Verify Traffic Reaching Location

1. Check DNS query logs in Gateway dashboard
2. Filter by location name
3. Verify source IPs match configured networks

### Test DNS Resolution

```bash
# Test from office network
dig @172.64.36.1 example.com

# Verify Gateway is answering
dig @172.64.36.1 TXT debug.cloudflare.com
```

### Location Not Matching

- Verify source IP is in configured CIDR range
- Check for NAT/proxy changing source IP
- Ensure DNS traffic egresses from expected IP

## Limits

| Limit | Value |
|-------|-------|
| Locations per account | 100 |
| Networks per location | 10 |
| IP ranges overlap | Not allowed |

## See Also

- [DNS Policies](./dns-policies.md) - Using locations in policies
- [Devices](../devices/README.md) - WARP client alternative
