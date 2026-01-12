# Durable Objects API

## Class Structure

```typescript
import { DurableObject } from "cloudflare:workers";

export class MyDO extends DurableObject<Env> {
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
  }
  async myMethod(arg: string): Promise<string> { return arg; }
  async alarm() { }
  async webSocketMessage(ws: WebSocket, msg: string | ArrayBuffer) { }
}
```

## SQLite Storage

```typescript
// Query
const cursor = this.ctx.storage.sql.exec("SELECT * FROM users WHERE age > ?", 18);
cursor.toArray()          // All rows
cursor.one()              // 1 row or throw
cursor.raw().toArray()    // [[1, 'Alice']]
cursor.rowsRead, cursor.rowsWritten, cursor.columnNames
this.ctx.storage.sql.databaseSize

// Schema
this.ctx.storage.sql.exec("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT)");
```

## KV Storage

```typescript
// Sync (SQLite only)
this.ctx.storage.kv.get/put/delete("key", value)
this.ctx.storage.kv.list({ prefix: "user:" })

// Async
await this.ctx.storage.get/put/delete("key", value)
await this.ctx.storage.put({ k1: "a", k2: "b" })  // Batch 128 max
await this.ctx.storage.list({ prefix: "user:", start: "user:100", limit: 50 })
await this.ctx.storage.deleteAll()
```

## Transactions

```typescript
// Sync
this.ctx.storage.transactionSync(() => { /* SQL ops */ });

// Async
await this.ctx.storage.transaction(async (txn) => {
  await txn.get/put/delete("key", value);  // Throw to rollback
});
```

## Point-in-Time Recovery

```typescript
await this.ctx.storage.getCurrentBookmark()
await this.ctx.storage.getBookmarkForTime(Date.now() - 86400000)
await this.ctx.storage.onNextSessionRestoreBookmark(bookmark)  // Call ctx.abort() after
```

## Alarms

```typescript
await this.ctx.storage.setAlarm(Date.now() + 3600000)
await this.ctx.storage.getAlarm()
await this.ctx.storage.deleteAlarm()
async alarm() { /* runs when fires */ }
```

## WebSockets

```typescript
async fetch(req: Request): Promise<Response> {
  const [client, server] = Object.values(new WebSocketPair());
  this.ctx.acceptWebSocket(server, ["room:123"]);
  server.serializeAttachment({ userId: "abc" });
  return new Response(null, { status: 101, webSocket: client });
}

async webSocketMessage(ws: WebSocket, msg: string | ArrayBuffer) {
  const data = ws.deserializeAttachment();
  for (const c of this.ctx.getWebSockets()) c.send(msg);
}

// Management: getWebSockets(), getTags(ws), ws.send/close()
```
