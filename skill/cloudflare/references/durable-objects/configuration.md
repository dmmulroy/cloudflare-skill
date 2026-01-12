# Durable Objects Configuration

## Basic Setup

```jsonc
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2024-04-03",
  "durable_objects": {
    "bindings": [
      { "name": "MY_DO", "class_name": "MyDO" },
      { "name": "EXTERNAL", "class_name": "ExternalDO", "script_name": "other-worker" }
    ]
  },
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["MyDO"] }
  ]
}
```

## Migrations

```jsonc
{
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["MyDO"] },            // Create SQLite (recommended)
    // { "tag": "v1", "new_classes": ["MyDO"] },                // Create KV (paid only)
    { "tag": "v2", "renamed_classes": [{ "from": "Old", "to": "New" }] },
    { "tag": "v3", "transferred_classes": [{ "from": "Src", "from_script": "old", "to": "Dest" }] },
    { "tag": "v4", "deleted_classes": ["Obsolete"] }           // Destroys ALL data!
  ]
}
```

Tags unique/sequential, no rollback, auto-applied on deploy, test with `--dry-run`

## Advanced

```jsonc
{
  "limits": { "cpu_ms": 300000 },  // Default 30s, max 300s
  "env": {
    "production": {
      "durable_objects": {
        "bindings": [{ "name": "MY_DO", "class_name": "MyDO", "environment": "production" }]
      }
    }
  }
}
```

## Types

```typescript
import { DurableObject } from "cloudflare:workers";

interface Env {
  MY_DO: DurableObjectNamespace<MyDO>;
}

export class MyDO extends DurableObject<Env> {}

type DurableObjectNamespace<T> = {
  newUniqueId(options?: { jurisdiction?: string }): DurableObjectId;
  idFromName(name: string): DurableObjectId;
  idFromString(id: string): DurableObjectId;
  get(id: DurableObjectId): DurableObjectStub<T>;
};
```

## Commands

```bash
npx wrangler dev                                       # Local dev
npx wrangler dev --remote                              # Test prod DOs
npx wrangler deploy                                    # Deploy + migrations
npx wrangler deploy --dry-run                          # Validate only
npx wrangler durable-objects list                      # List namespaces
npx wrangler durable-objects info <namespace> <id>     # Object info
npx wrangler durable-objects delete <namespace> <id>   # Delete object
```
