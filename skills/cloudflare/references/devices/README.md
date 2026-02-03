# Cloudflare Devices (WARP)

Endpoint security with the WARP client for DNS filtering, secure web gateway, and private network access.

## Overview

WARP client provides:
- **DNS filtering** - Gateway DNS policies on device
- **Secure Web Gateway** - HTTP/HTTPS inspection and filtering
- **Private network access** - Access resources via Tunnel
- **Device posture** - Verify device security state
- **Split tunneling** - Control what traffic goes through WARP

## WARP Modes

| Mode | DNS | HTTP | Private Networks | Use Case |
|------|-----|------|------------------|----------|
| Gateway with WARP | Yes | Yes | Yes | Full protection |
| Gateway with DoH | Yes | No | No | DNS filtering only |
| WARP (Consumer) | No | No | No | Consumer privacy VPN |

## Quick Start

### 1. Deploy WARP Client

**macOS:**
```bash
brew install --cask cloudflare-warp
```

**Windows:** Download from https://developers.cloudflare.com/warp-client/get-started/windows/

**Linux:**
```bash
curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg | sudo gpg --yes --dearmor -o /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] https://pkg.cloudflareclient.com/ jammy main" | sudo tee /etc/apt/sources.list.d/cloudflare-client.list
sudo apt update && sudo apt install cloudflare-warp
```

### 2. Enroll Device

**User-initiated (recommended):**
1. Open WARP client
2. Click Settings → Account → Login with Cloudflare for Teams
3. Enter organization name: `your-org.cloudflareaccess.com`

**MDM deployment:**
Use `mdm.xml` configuration file with organization settings.

### 3. Verify Connection

```bash
warp-cli status
# Expected: Status: Connected, Mode: Gateway with WARP
```

## Device Enrollment

### Authentication Options

1. **Identity Provider** - User logs in via Okta, Azure AD, etc.
2. **One-Time PIN** - Email-based verification
3. **Service Token** - Automated deployment
4. **Device Certificate** - mTLS enrollment

### Enrollment Policy

Configure in: Zero Trust → Settings → WARP Client → Device Enrollment

```json
{
  "enrollment": {
    "auth_enabled": true,
    "allowed_identity_providers": ["okta-idp-uuid"],
    "rule": {
      "include": [
        {"email_domain": {"domain": "company.com"}}
      ]
    }
  }
}
```

## Split Tunneling

### Include Mode (Recommended)

Only specified traffic goes through WARP:

```json
{
  "include": [
    {"address": "10.0.0.0/8"},
    {"address": "172.16.0.0/12"},
    {"host": "internal.company.com"}
  ]
}
```

### Exclude Mode

All traffic goes through WARP except specified:

```json
{
  "exclude": [
    {"address": "192.168.1.0/24"},
    {"host": "local-only.com"}
  ]
}
```

## CLI Commands

```bash
# Status
warp-cli status

# Connect/Disconnect
warp-cli connect
warp-cli disconnect

# Registration
warp-cli registration show
warp-cli registration delete  # For re-enrollment

# Settings
warp-cli settings

# Switch modes
warp-cli mode warp+doh        # Gateway with WARP
warp-cli mode doh             # Gateway with DoH only
warp-cli mode warp            # Consumer WARP

# Diagnostics
warp-cli debug qlog           # Enable QUIC logging
warp-cli debug connectivity   # Test connectivity
```

## Device Management API

### List Devices

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices" | \
  jq '.result[] | {id, name, user_email, last_seen}'
```

### Get Device

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/$DEVICE_ID"
```

### Revoke Device

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/$DEVICE_ID/revoke" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## In This Reference

- [WARP](./warp.md) - WARP client configuration
- [Posture](./posture.md) - Device posture checks
- [Policies](./policies.md) - Device settings policies
- [API](./api.md) - Full API reference
- [Gotchas](./gotchas.md) - Common issues

## See Also

- [Gateway](../gateway/README.md) - DNS and HTTP filtering
- [Tunnel](../tunnel/README.md) - Private network access
- [Access](../access/README.md) - Device posture in access policies
