# Durable Objects Gotchas

## Limits

**SQLite**: 10GB/DO, 5GB total (Free) / unlimited (Paid), 2MB key+value, 30s CPU (300s max), 500 classes (Paid) / 100 (Free), 100 SQL cols, 100KB statement, 32MiB WebSocket msg

**General**: ~1000 req/s/DO, unlimited DOs, unlimited WebSockets (within 128MB memory)

## Common Issues

**DO overloaded**: 503 errors → shard across DOs with random IDs
**Storage quota**: Write failures → upgrade or cleanup alarms
**CPU exceeded**: Terminated → increase `limits.cpu_ms` or chunk work
**WebSockets disconnect**: Eviction → use hibernation or reconnection
**Migration failed**: Deploy error → check tag uniqueness/class names
**RPC not found**: compatibility_date < 2024-04-03 → update or use fetch
**One alarm limit**: Need multiple → use event queue pattern
**Constructor on wake**: Expensive init → lazy methods or cache

## Debugging

```bash
npx wrangler dev              # Local
npx wrangler dev --remote     # Prod DOs
npx wrangler tail             # Logs
npx wrangler durable-objects list
```

```typescript
this.ctx.storage.sql.databaseSize                           // Storage size
cursor.rowsRead, cursor.rowsWritten                         // Query stats
```

## RPC vs Fetch

**RPC** (recommended): Type-safe, simpler, faster, needs compatibility_date >= 2024-04-03
**Fetch**: HTTP semantics, legacy, proxying

```typescript
stub.myMethod(arg)                              // RPC
stub.fetch(new Request("http://do/endpoint"))   // Fetch
```

## Migration Gotchas

Tags unique/sequential, no rollback, test with `--dry-run`, `deleted_classes` destroys data, transfers need coordination, renames preserve data/IDs
