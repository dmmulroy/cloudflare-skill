# Gateway Gotchas

Common issues, limitations, and troubleshooting for Cloudflare Gateway.

## Common Errors

### "DNS Query Not Being Filtered"

**Symptoms:** Traffic bypasses Gateway DNS policies

**Causes & Solutions:**

1. **WARP not connected** - Check WARP client is connected and Gateway enabled
   ```bash
   warp-cli status
   warp-cli settings
   ```

2. **DNS over HTTPS (DoH) bypass** - Browser using its own DoH provider
   - Disable DoH in browser settings
   - Create DNS policy to block DoH providers

3. **Hardcoded DNS servers** - Application using its own DNS
   - Block UDP/TCP 53 to non-Gateway IPs

4. **Split tunnel excluding traffic** - Check device policy split tunnel config

### "HTTPS Sites Not Being Filtered"

**Cause:** TLS inspection not enabled or certificate not installed

**Solutions:**

1. **Enable TLS inspection:**
   ```bash
   curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/configuration" \
     -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"settings": {"tls_decrypt": {"enabled": true}}}'
   ```

2. **Install root CA on devices:**
   - Download CA from Gateway dashboard
   - Install in system trust store
   - For managed devices, deploy via MDM

3. **Check do_not_inspect policies** - Verify the site isn't being bypassed

### "Certificate Errors After Enabling TLS Inspection"

**Cause:** Root CA not trusted or certificate pinning

**Solutions:**

1. **Install root CA** - Deploy to all devices via MDM
2. **Bypass pinned applications:**
   ```json
   {
     "name": "Bypass Certificate Pinning",
     "action": "do_not_inspect",
     "filters": ["http"],
     "traffic": "any(http.application[*] in {\"Banking App\" \"Healthcare App\"})"
   }
   ```

3. **Bypass specific domains:**
   ```json
   {
     "traffic": "http.host in $pinned_domains"
   }
   ```

### "Policy Not Taking Effect"

**Causes:**

1. **Precedence conflict** - Lower precedence (higher number) policy being matched first
   ```bash
   curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
     "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/rules" | \
     jq '.result | sort_by(.precedence) | .[] | {name, precedence, action, enabled}'
   ```

2. **Policy disabled** - Check `enabled: true`

3. **Traffic expression wrong** - Test with simpler expression first

4. **Propagation delay** - Wait 60 seconds for changes to propagate

### "Block Page Not Showing"

**Causes:**

1. **Block page not enabled:**
   ```json
   "rule_settings": {
     "block_page_enabled": true
   }
   ```

2. **HTTPS traffic without TLS inspection** - Only DNS block page works without TLS inspection

3. **Non-browser application** - Block pages only work for browsers

### "Application Detection Not Working"

**Cause:** Application signatures require TLS inspection to identify traffic

**Solution:** Enable TLS inspection for accurate application detection

### "False Positives - Legitimate Site Blocked"

**Solutions:**

1. **Check which rule blocked it:**
   ```bash
   # Check Gateway activity logs in dashboard
   # or via Logpush to see matched rule
   ```

2. **Add to allow list:**
   ```json
   {
     "name": "Allow False Positive",
     "action": "allow",
     "filters": ["dns"],
     "traffic": "dns.fqdn == \"legitimate-site.com\"",
     "precedence": 1
   }
   ```

3. **Request recategorization** via Cloudflare Radar

### "Slow DNS Resolution"

**Causes:**

1. **Many complex policies** - Simplify or reduce policy count
2. **Large lists** - Break into smaller, targeted lists
3. **Logging overhead** - Reduce logging verbosity for non-critical traffic

### "Gateway Logs Missing"

**Solutions:**

1. **Enable logging:**
   ```bash
   curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/logging" \
     -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
       "settings_by_rule_type": {
         "dns": {"log_all": true},
         "http": {"log_all": true}
       }
     }'
   ```

2. **Set up Logpush** for real-time log export

3. **Check retention** - Gateway logs have limited retention

## Limits

| Limit | Value |
|-------|-------|
| Gateway rules | 500 |
| Lists per account | 100 |
| Items per list | 5,000 (20,000 Enterprise) |
| Locations | 100 |
| DNS query size | 512 bytes (UDP), 65535 (TCP/DoH) |
| HTTP body inspection | 100 KB |
| File scan size | 15 MB |

## TLS Inspection Limitations

### Applications That May Break

- Apps with certificate pinning (banking, healthcare)
- VPN clients
- Some desktop applications
- IoT devices

### Bypass Categories to Consider

- Financial Services
- Health
- Government (some)

### Common Bypass List

```json
{
  "name": "TLS Bypass Domains",
  "type": "DOMAIN",
  "items": [
    {"value": "*.apple.com"},
    {"value": "*.icloud.com"},
    {"value": "*.microsoft.com"},
    {"value": "*.windowsupdate.com"},
    {"value": "*.digicert.com"},
    {"value": "*.letsencrypt.org"},
    {"value": "*.banking.com"},
    {"value": "*.healthcare.com"}
  ]
}
```

## Performance Considerations

### Policy Order

- Put allow rules before block rules when possible
- Use specific matchers over broad ones
- Avoid regex when exact match works

### List Optimization

- Use multiple smaller lists vs one large list
- Keep frequently matched items at the top
- Remove stale entries regularly

## Debugging

### Test DNS Resolution

```bash
# Through WARP
dig @1.1.1.1 example.com

# Check Gateway is being used
dig +short TXT debug.cloudflare.com
```

### Test HTTP Filtering

```bash
# This should be blocked if malware category blocked
curl -v https://malware.testcategory.com

# Check WARP is proxying
curl -I https://cloudflare.com/cdn-cgi/trace
```

### Check Gateway Configuration

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/gateway/configuration" | jq .
```

## See Also

- [DNS Policies](./dns-policies.md) - DNS filtering
- [HTTP Policies](./http-policies.md) - HTTP filtering
- [Devices Gotchas](../devices/gotchas.md) - WARP client issues
