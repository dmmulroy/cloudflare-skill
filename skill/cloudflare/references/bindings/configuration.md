## Wrangler Configuration Patterns

### JSON Configuration

**wrangler.jsonc:**
```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2025-01-01", // Use current date for new projects
  "vars": {
    "API_URL": "https://api.example.com"
  }
}
```

### Environment-Specific Configuration

```jsonc
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2025-01-01", // Use current date for new projects
  
  // Production bindings
  "vars": {
    "ENV": "production"
  },
  "kv_namespaces": [
    {
      "binding": "CACHE",
      "id": "prod-kv-id"
    }
  ],
  
  // Environment overrides
  "env": {
    "staging": {
      "vars": {
        "ENV": "staging"
      },
      "kv_namespaces": [
        {
          "binding": "CACHE",
          "id": "staging-kv-id