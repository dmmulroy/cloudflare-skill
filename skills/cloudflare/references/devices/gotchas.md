# Devices Gotchas

Common issues, limitations, and troubleshooting for WARP and device management.

## Common Errors

### "WARP Not Connecting"

**Symptoms:** WARP shows "Connecting..." or "Unable to Connect"

**Causes & Solutions:**

1. **Network blocking UDP 443/QUIC:**
   ```bash
   # Test connectivity
   warp-cli debug connectivity
   ```
   - Switch to TCP mode if QUIC blocked

2. **Firewall blocking Cloudflare IPs:**
   - Allow outbound to Cloudflare ranges: https://www.cloudflare.com/ips/

3. **VPN/proxy conflict:**
   - Disable other VPN clients
   - Check for conflicting proxy settings

4. **DNS resolution issues:**
   ```bash
   # Test DNS
   dig @1.1.1.1 cloudflare.com
   ```

5. **Certificate issues:**
   ```bash
   # Reset WARP
   warp-cli registration delete
   # Re-enroll
   ```

### "Device Posture Check Failed"

**Symptoms:** User blocked from accessing resources

**Debug:**
```bash
# Check posture status on device
warp-cli debug posture

# Check posture via API
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/$DEVICE_ID" | \
  jq '.result.device_posture'
```

**Common causes:**

1. **Disk encryption not enabled:**
   - macOS: Enable FileVault
   - Windows: Enable BitLocker

2. **Firewall disabled:**
   - macOS: System Preferences → Security → Firewall → Turn On
   - Windows: Windows Security → Firewall → Turn On

3. **OS version outdated:**
   - Update to minimum required version

4. **EDR agent not running:**
   - Verify CrowdStrike/SentinelOne service is active

5. **Check hasn't run yet:**
   - Wait for next scheduled check (5-15 min)
   - Or disconnect/reconnect WARP

### "Cannot Enroll Device"

**Causes:**

1. **Organization not configured:**
   - Verify org name: `your-org.cloudflareaccess.com`

2. **Enrollment rules blocking:**
   ```bash
   # Check enrollment settings via API
   curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
     "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy"
   ```

3. **User not in allowed groups:**
   - Check Identity Provider groups

4. **Device limit reached:**
   - Check account device limits

### "Split Tunnel Not Working"

**Symptoms:** Traffic going through WARP that should be excluded (or vice versa)

**Debug:**
```bash
# Check current routes
warp-cli settings
```

**Solutions:**

1. **Verify split tunnel config:**
   ```bash
   curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
     "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/devices/policy/exclude"
   ```

2. **Check for overlapping routes:**
   - More specific routes take precedence

3. **Verify correct mode:**
   - Include mode: Only listed routes go through WARP
   - Exclude mode: All traffic except listed goes through WARP

4. **Restart WARP after config change:**
   ```bash
   warp-cli disconnect
   warp-cli connect
   ```

### "Private Network Access Not Working"

**Symptoms:** Cannot reach IPs routed through Tunnel

**Debug:**
```bash
# Check if WARP connected
warp-cli status

# Check routes
warp-cli debug routes

# Test connectivity
ping 10.0.0.1  # Internal IP
```

**Solutions:**

1. **Verify tunnel has IP route:**
   ```bash
   cloudflared tunnel route ip show
   ```

2. **Check virtual network assignment:**
   - WARP device and tunnel must use same virtual network

3. **Verify Gateway mode enabled:**
   ```bash
   warp-cli settings
   # Should show: Mode = Gateway with WARP
   ```

4. **Check split tunnel includes the IP range:**
   - If using include mode, internal IPs must be listed

### "TLS Certificate Errors"

**Cause:** Gateway TLS inspection without root CA installed

**Solutions:**

1. **Install Cloudflare root CA:**
   - Download from Gateway dashboard
   - Install in system trust store

2. **Or bypass TLS inspection:**
   - Create "do_not_inspect" rule for affected sites

### "WARP Disconnects Frequently"

**Causes:**

1. **Captive portal detection:**
   - Connect to captive portal first
   - Adjust captive portal timeout:
     ```json
     {"captive_portal": 300}
     ```

2. **Network instability:**
   - Check network connection
   - Try different network

3. **Sleep/wake issues:**
   - Disable auto-disconnect on sleep

### "Gateway DNS Not Filtering"

**Causes:**

1. **DoH/DoT bypass:**
   - Browser using its own DNS provider
   - Disable DoH in browser

2. **WARP in wrong mode:**
   ```bash
   warp-cli mode warp+doh  # Ensure Gateway mode
   ```

3. **Application hardcoded DNS:**
   - Some apps ignore system DNS

## Limits

| Limit | Value |
|-------|-------|
| Devices per user | 5 (configurable) |
| Device policies | 100 |
| Posture checks | 100 |
| Split tunnel entries | 1,000 |
| Fallback domains | 100 |

## Platform-Specific Issues

### macOS

1. **System Extension approval required:**
   - System Preferences → Security → Allow "Cloudflare"

2. **Network Extension not loading:**
   ```bash
   systemextensionsctl list
   ```

3. **FileVault check false negative:**
   - Ensure FileVault fully enabled (not just enabled for current user)

### Windows

1. **WinTun driver issues:**
   - Reinstall WARP client

2. **BitLocker check false negative:**
   - Ensure BitLocker enabled on all drives

3. **Domain join check:**
   - Verify AD domain membership status

### Linux

1. **Limited features:**
   - No GUI, CLI only
   - Fewer posture checks available

2. **systemd service:**
   ```bash
   sudo systemctl status warp-svc
   ```

## Debugging Commands

```bash
# Full status
warp-cli status

# Current settings
warp-cli settings

# Account info
warp-cli registration show

# Debug posture
warp-cli debug posture

# Debug connectivity
warp-cli debug connectivity

# Debug logs (macOS)
log stream --predicate 'subsystem == "com.cloudflare.1dot1dot1dot1.macos"' --level debug

# Debug logs (Linux)
journalctl -u warp-svc -f
```

## See Also

- [WARP](./warp.md) - WARP configuration
- [Posture](./posture.md) - Device posture setup
- [Gateway Gotchas](../gateway/gotchas.md) - DNS/HTTP filtering issues
