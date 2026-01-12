# Durable Objects Patterns

## Rate Limiting

```typescript
async checkLimit(key: string, limit: number, windowMs: number): Promise<boolean> {
  const req = await this.ctx.storage.sql.exec(
    "SELECT COUNT(*) as count FROM requests WHERE key = ? AND timestamp > ?",
    key, Date.now() - windowMs
  ).one();
  if (req.count >= limit) return false;
  this.ctx.storage.sql.exec("INSERT INTO requests (key, timestamp) VALUES (?, ?)", key, Date.now());
  return true;
}
```

## Distributed Lock

```typescript
private held = false;
async acquire(timeoutMs = 5000): Promise<boolean> {
  if (this.held) return false;
  this.held = true;
  await this.ctx.storage.setAlarm(Date.now() + timeoutMs);
  return true;
}
async release() { this.held = false; await this.ctx.storage.deleteAlarm(); }
async alarm() { this.held = false; }
```

## Real-time Collaboration

```typescript
async fetch(req: Request): Promise<Response> {
  const [client, server] = Object.values(new WebSocketPair());
  this.ctx.acceptWebSocket(server);
  server.send(JSON.stringify({ type: "init", content: this.ctx.storage.kv.get("doc") || "" }));
  return new Response(null, { status: 101, webSocket: client });
}

async webSocketMessage(ws: WebSocket, msg: string) {
  const data = JSON.parse(msg);
  if (data.type === "edit") {
    this.ctx.storage.kv.put("doc", data.content);
    for (const c of this.ctx.getWebSockets()) if (c !== ws) c.send(msg);
  }
}
```

## Session Management

```typescript
async createSession(userId: string, data: object): Promise<string> {
  const id = crypto.randomUUID(), exp = Date.now() + 86400000;
  this.ctx.storage.sql.exec("INSERT INTO sessions VALUES (?, ?, ?, ?)", id, userId, JSON.stringify(data), exp);
  await this.ctx.storage.setAlarm(exp);
  return id;
}

async getSession(id: string): Promise<object | null> {
  const row = this.ctx.storage.sql.exec("SELECT data FROM sessions WHERE id = ? AND expires_at > ?", id, Date.now()).one();
  return row ? JSON.parse(row.data) : null;
}

async alarm() { this.ctx.storage.sql.exec("DELETE FROM sessions WHERE expires_at <= ?", Date.now()); }
```

## Deduplication

```typescript
private pending = new Map<string, Promise<Response>>();
async deduplicatedFetch(url: string): Promise<Response> {
  if (this.pending.has(url)) return this.pending.get(url)!;
  const p = fetch(url).finally(() => this.pending.delete(url));
  this.pending.set(url, p);
  return p;
}
```

## Multiple Events (Single Alarm)

```typescript
async scheduleEvent(id: string, runAt: number, repeatMs?: number) {
  await this.ctx.storage.put(`event:${id}`, { id, runAt, repeatMs });
  const curr = await this.ctx.storage.getAlarm();
  if (!curr || runAt < curr) await this.ctx.storage.setAlarm(runAt);
}

async alarm() {
  const now = Date.now(), events = await this.ctx.storage.list({ prefix: "event:" });
  let next: number | null = null;
  for (const [key, ev] of events) {
    if (ev.runAt <= now) {
      await this.processEvent(ev);
      ev.repeatMs ? await this.ctx.storage.put(key, { ...ev, runAt: now + ev.repeatMs }) : await this.ctx.storage.delete(key);
    }
    if (ev.runAt > now && (!next || ev.runAt < next)) next = ev.runAt;
  }
  if (next) await this.ctx.storage.setAlarm(next);
}
```

## Best Practices

**Design**: Keep objects focused, use `idFromName()` for coordination, `newUniqueId()` for sharding, minimize constructor work, leverage WebSocket hibernation

**Storage**: Prefer SQLite, create indexes judiciously, batch with transactions, set alarms for cleanup, use PITR before risky ops

**Performance**: One DO ~1000 req/s max - shard for more, cache in memory, avoid long ops, use alarms for deferred work

**Reliability**: Handle 503 with retry+backoff, design for cold starts, test migrations, monitor alarm retries

**Security**: Validate inputs in Workers, don't trust user names, rate limit DO creation, use jurisdiction tags, encrypt sensitive data
