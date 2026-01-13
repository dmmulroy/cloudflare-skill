# The Perfect Durable Object?

> Source: https://coey.dev/perfect-do

Trying to figure out what "good" looks like.

Keep building things with DOs and something feels off. Not broken.. just wrong. What would a well-designed DO actually look like?

## Metaphors That Help Me

### The Night Watchman

A DO is like a night watchman who:

- Owns **ONE building** (not the whole city)
- Sleeps until something happens
- Wakes instantly, handles it with full authority
- Goes back to sleep
- Can coordinate with other watchmen via RPC when needed

With Workers RPC, DO-to-DO communication chains like 1->2->3 are treated as a single end-to-end call. However, this is still a network hop and can be slow, especially between colos. The question isn't just performance, but whether the coordination complexity is necessary.

### The Safety Deposit Box

Your data, your box, your rules.

- Bank handles infrastructure
- But the box is **YOURS**
- Opens when needed, closes when done
- Not a vault for the whole bank's operations

### Database Row That Can Think

Database row that woke up, realized it could run code, handles its own business logic. Single-threaded. Authoritative over its slice of state.

Not a database. Not a general-purpose compute node.

## Rough Numbers

Updated based on [official Cloudflare docs](https://developers.cloudflare.com/durable-objects/platform/limits/) and employee feedback. These are design heuristics, not hard limits. DOs routinely scale into "red" territory with proper sharding patterns.

| Characteristic | Feels Right | Yellow Flag | Red Flag |
|----------------|-------------|-------------|----------|
| Requests/sec (sustained) | < 100 | 100-500 | > 500 |
| Storage keys | < 100 | 100-1000 | > 1000 (consider access patterns) |
| Total state size | < 10MB | 10MB-100MB | > 1GB |
| Alarm frequency | Minutes-hours (only when work exists) | Every 30s | Every few seconds (waking idle DOs) |
| WebSocket duration | Short bursts | Hours (hibernating) | Days always-on |
| Fan-out from this DO | Never/rarely | To < 10 DOs | To 100+ DOs |

Aim for green. Yellow = question it. Red = consider sharding patterns.

## What a "DO" Seems To Be

- **Single-threaded authority** over a piece of state. No locks, no races, no coordination.
- **Stateful actor that hibernates.** Sleep is default, not optimization.
- **Colocated compute + storage.** Code runs where data lives.
- **Natural ownership boundary.** One user. One room. One session. One document. The "atom" of coordination.
- **Parent-child relationships.** Hierarchical data uses separate child DOs for parallelism while maintaining single-threaded consistency per child.

## What a "DO" Probably Isn't

- **A global singleton.** One DO handling everything = bottleneck with extra steps.
- **A message broker.** High-frequency pub/sub? Use Queues.
- **A long-running process.** Needs to run continuously? Use Workers + alarms, not always-on DO.
- **A chatty microservice.** Calling it on every request? Reconsider.
- **A general database.** Consistency doesn't matter or ownership unclear? Use KV or D1.
- **A place for blockConcurrencyWhile() on every request.** Use sparingly.. only for initialization/migrations. Regular requests should rely on input/output gates.

## Questions Before Creating One

### 1. Does this data have natural ownership?

User/room/session-owned? If shared across boundaries, DO might be wrong.

### 2. Do I actually need strong consistency here?

DOs give strong consistency. If eventual is fine, KV is cheaper.

### 3. Will instances communicate infrequently?

RPC chains are network hops (can be slow between colos). Constant coordination suggests wrong boundaries. Use parent-child relationships for hierarchical data instead of constant cross-DO calls.

### 4. Can this DO hibernate between meaningful work?

If it needs to stay awake, you're paying wall-clock time. Adds up.

### 5. Am I going to create thousands of these?

Thousands = fine. Millions = think about sharding. Billions = wrong tool.

## The "Perfect" DO

```ts
// - Owns ONE thing (user, room, document, session) - the "atom" of coordination
// - Uses deterministic IDs (getByName) for predictable routing
// - Uses SQLite storage as source of truth (not KV)
// - Uses RPC methods, not fetch() handler
// - Wakes up infrequently
// - Does meaningful work when awake
// - Stores < 100 keys, < 10MB total (heuristics)
// - Hibernates ASAP
// - Can coordinate via RPC when needed (zero latency chains)
// - Could disappear and reconstruct from storage
// - Is the authority, not a cache
// - Uses parent-child relationships for hierarchical data
```

## Worth Deeper Exploration

These areas have solutions and patterns, but warrant deeper thought and exploration based on your specific use case.

### WebSockets

Use Hibernatable WebSockets API.. DO owns the connection. Allows DO to sleep while maintaining connections, significantly reducing costs for idle connections.

### Sharding

One DO per user vs hash into N buckets? For high-throughput systems, use `request.cf.colo` for geographic awareness. Rate limit binding is per-colo and enables dynamic sharding patterns.

`request.cf.colo` gives you the IATA airport code of the datacenter (e.g., "SFO", "LHR"). Rate limiting bindings are per-colo (counters not shared across datacenters), so combining them gives automatic geographic distribution.

```ts
// Worker: Colo-aware sharding with rate limiting
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const userId = new URL(request.url).searchParams.get("userId") || "unknown";
    const colo = request.cf?.colo || "unknown";
    const shardKey = `${colo}:${userId}`;
    
    // Rate limit per-colo before hitting DO
    const { success } = await env.RATE_LIMITER.limit({ key: shardKey });
    if (!success) return new Response("Rate limited", { status: 429 });
    
    // Route to colo-aware DO shard
    const stub = env.MY_DO.getByName(shardKey);
    return await stub.fetch(request);
  }
}
```

### DO-to-DO Communication

RPC chains (1->2->3) are treated as a single end-to-end call, but this is still a network hop and can be slow, especially between colos.

**Use when:** Parent-child hierarchies (fleet pattern), infrequent coordination, clear ownership boundaries.

**Avoid when:** Every request needs multiple DOs, constant cross-DO calls, or when a database would be simpler.

### Storage as Cache vs Source of Truth

Use SQLite-backed storage as source of truth. In-memory state is NOT preserved on eviction or crash.. always persist critical state. Use SQLite for relational queries, indexes, transactions.
