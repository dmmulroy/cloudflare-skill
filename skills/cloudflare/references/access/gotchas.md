# Access Gotchas

Common issues, limitations, and troubleshooting for Cloudflare Access.

## Common Errors

### "Access Denied" Despite Correct Policy

**Symptoms:** User matches policy rules but still gets denied

**Causes & Solutions:**

1. **Policy precedence** - Lower number = higher priority. A `deny` policy with precedence 1 blocks before `allow` with precedence 2.
   ```bash
   # Check policy order
   curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
     "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps/$APP_ID/policies" | \
     jq '.result | sort_by(.precedence) | .[] | {name, decision, precedence}'
   ```

2. **`require` rules not met** - User matches `include` but fails `require` (e.g., device posture)
   ```bash
   # Check user's failed logins for details
   curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
     "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/users/$USER_ID/failed_logins"
   ```

3. **Group membership not synced** - IdP group changes take time to propagate
   - Wait 5-15 minutes for SCIM sync
   - User can force refresh by logging out and back in

4. **Browser cached old session** - Clear cookies for `*.cloudflareaccess.com`

### "Application Not Found"

**Causes:**
- Domain DNS not proxied through Cloudflare (orange cloud)
- Application domain doesn't match exactly (www vs non-www)
- Tunnel not connected for self-hosted apps

**Solution:**
```bash
# Verify DNS is proxied
dig +short app.example.com  # Should return Cloudflare IPs

# Check tunnel status
cloudflared tunnel info my-tunnel
```

### "Invalid Service Token"

**Causes:**
- Token expired (default: 1 year)
- Token revoked or deleted
- Wrong `CF-Access-Client-Id` or `CF-Access-Client-Secret` headers

**Using service tokens:**
```bash
curl -H "CF-Access-Client-Id: $CLIENT_ID" \
     -H "CF-Access-Client-Secret: $CLIENT_SECRET" \
     https://app.example.com/api
```

### "Session Expired Frequently"

**Cause:** `session_duration` set too low

**Solution:** Increase session duration (max 720h = 30 days)
```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps/$APP_ID" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"session_duration": "168h"}'  # 1 week
```

### "CORS Errors"

**Cause:** Access login page triggers CORS issues for API requests

**Solutions:**

1. **For browser apps** - Configure CORS headers on the Access app:
   ```json
   {
     "cors_headers": {
       "allowed_methods": ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
       "allowed_origins": ["https://your-frontend.com"],
       "allow_credentials": true,
       "max_age": 86400
     }
   }
   ```

2. **For API clients** - Use service tokens instead of browser auth

3. **For SPA with API** - Use `same_site_cookie_attribute: "none"` with `secure: true`

### "IdP Login Loop"

**Symptoms:** Redirects between IdP and Access endlessly

**Causes:**
- Mismatched redirect URIs in IdP config
- Multiple IdPs with same email domain
- Cookie issues in browser

**Solutions:**
1. Verify callback URL in IdP matches exactly: `https://<team-name>.cloudflareaccess.com/cdn-cgi/access/callback`
2. If using multiple IdPs, set `auto_redirect_to_identity: false` on apps
3. Clear all cookies and try incognito mode

### "Device Posture Check Failed"

**Cause:** User's device doesn't meet posture requirements

**Debug:**
```bash
# Check posture rules
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/posture"

# Check user's device status
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices?search=user@example.com"
```

**Common posture issues:**
- WARP client not connected
- Antivirus not running
- Disk encryption disabled
- OS version too old

## Limits

| Limit | Value |
|-------|-------|
| Applications per account | 500 |
| Policies per application | 50 |
| Groups per account | 300 |
| Rules per policy/group | 100 |
| Service tokens per account | 500 |
| Identity providers | 50 |
| Session duration max | 720h (30 days) |
| mTLS certificates | 50 |

## Performance Considerations

### Cold Start Latency

First request after policy change may be slower (< 500ms typically).

### IdP Roundtrip

Initial login adds IdP latency (varies by provider). Subsequent requests use cached session.

### Global Propagation

Policy changes propagate globally within 60 seconds.

## Debugging

### Check User's Identity

```bash
# What Access sees for a user
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/users/$USER_ID/last_seen_identity" | jq .
```

### Access Request Logs

```bash
# Recent access attempts
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/logs/access_requests?limit=100" | \
  jq '.result[] | {user_email, app_domain, action, created_at}'
```

### Test Policy Evaluation

```bash
# Test which policy matches for a user
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps/$APP_ID/user_policy_checks"
```

## See Also

- [Policies](./policies.md) - Policy configuration details
- [Groups](./groups.md) - Access Groups
- [Devices Gotchas](../devices/gotchas.md) - Device posture issues
