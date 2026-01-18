## Configuration Options

### Widget Configurations

```javascript
{
  sitekey: 'required',              // Your widget sitekey
  action: 'optional-string',        // Custom action identifier for analytics
  cData: 'optional-string',         // Custom data passed back in validation
  callback: (token) => {},          // Success callback with token
  'error-callback': (code) => {},   // Error callback
  execution: 'render',              // 'render' (default) or 'execute'
  'expired-callback': () => {},     // Token expiry callback
  'before-interactive-callback': () => {}, // Before showing checkbox
  'after-interactive-callback': () => {},  // After checkbox interaction
  'unsupported-callback': () => {}, // Client/browser not supported callback
  theme: 'auto',                    // 'light', 'dark', 'auto'
  language: 'auto',                 // 'auto' or ISO 639-1 code (e.g. 'en', 'en-US')
  tabindex: 0,                      // Tab index for accessibility
  'timeout-callback': () => {},     // Challenge timeout callback
  'response-field': true,           // Create hidden input (default: true)
  'response-field-name': 'cf-turnstile-response', // Name of the input element
  size: 'normal',                   // 'normal', 'flexible', 'compact'
  retry: 'auto',                    // 'auto' (default) or 'never'
  'retry-interval': 8000,           // Retry delay in ms (max 900000)
  'refresh-expired': 'auto',        // 'auto', 'manual', 'never'
  'refresh-timeout': 'auto',        // 'auto', 'manual', 'never'
  appearance: 'always',             // 'always', 'execute', 'interaction-only'
  'feedback-enabled': true          // Allow Cloudflare to gather feedback on failure
}
```
