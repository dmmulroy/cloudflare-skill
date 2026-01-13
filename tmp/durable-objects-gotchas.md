# Durable Objects Gotchas

> Source: https://coey.dev/durable-objects-gotchas

The quiz you wish you had before your first surprise bill.

**Problem:** Durable Objects billing is deceptively nuanced. Most people discover the gotchas after getting an unexpected bill.

**Solution:** Self-diagnosis quiz. Learn before you deploy.

---

## Gotcha #1: Duration Billing Trap

**Scenario:** You have a DO that handles a WebSocket connection for a real-time dashboard. User connects at 9am, disconnects at 5pm. During that time, the DO processes maybe 50 small messages total.

**Question:** What's your biggest cost driver here?

- A) The 50 request invocations
- B) 8 hours of wall-clock duration
- C) CPU time processing the messages
- D) Storage for connection state

**Answer:** B - Wall-clock duration.

DOs bill for wall-clock time while active, not just CPU time. 8 hours of keeping a WebSocket open = 8 hours of duration billing, even if the DO did almost nothing.

**The fix:** Use Hibernatable WebSockets. The DO can sleep while maintaining the connection, only waking (and billing) when messages arrive.

---

## Gotcha #2: storage.list() on Every Request

**Scenario:** Your DO stores user preferences (20 keys). On each request, you need 3 of those keys.

**Question:** What's the cheapest read pattern?

- A) `storage.get(['key1', 'key2', 'key3'])`
- B) `storage.list()` once on wake, cache in memory
- C) Single `storage.get('preferences')` with all 20 keys as one object
- D) It depends—on what?

**Answer:** D - It depends on request patterns and data volatility.

- **Option A** is cheapest per-request if you only need those 3 keys and data changes frequently
- **Option B** is cheapest if you serve many requests per wake cycle (amortizes the list cost)
- **Option C** is cheapest if you often need multiple keys and can tolerate reading/writing all 20 together

The real answer: profile your actual usage. Storage reads are cheap, but not free.

---

## Gotcha #3: Alarm Recursion

**Scenario:** You're using `storage.setAlarm()` to wake a DO every 5 minutes to check for stale data. The check takes 2ms of CPU time.

**Question:** Over 24 hours, what's happening to your costs that you might not expect?

**Answer:** 288 wake-ups × (minimum billable duration) per day.

Even if the check takes 2ms, you're billed for the minimum duration each time. Multiply across thousands of DOs, and you're waking them all up 288 times/day whether they have work or not.

**The fix:** Only schedule alarms when there's actual work pending. Check if the alarm is needed before setting it. If 90% of your DOs have no stale data, don't wake them.

---

## Gotcha #4: WebSocket Never Closes

**Scenario:** Your DO handles WebSocket connections. Users sometimes close their browser tabs without properly disconnecting.

**Question:** What happens if you never call `webSocket.close()` on disconnect?

**Answer:** The connection stays "open" from the DO's perspective, preventing hibernation and keeping duration billing active.

WebSocket connections don't automatically clean up on the DO side. If you don't handle the close event and explicitly close the connection, the DO may stay awake indefinitely.

**The fix:**
1. Handle `webSocketClose` and `webSocketError` events
2. Implement heartbeat/ping-pong to detect dead connections
3. Use Hibernatable WebSockets API properly

---

## Gotcha #5: Singleton vs Sharding

**Scenario:** You're building a rate limiter. Design A uses one global DO. Design B creates one DO per user. Design C creates one DO per user-per-hour (ephemeral).

**Question:** For 10,000 users making 100 requests/day each, rank these by cost (cheapest to most expensive).

**Answer:** Typically: B < A < C (but it depends on patterns).

- **Design A (global singleton):** One DO, but it never hibernates due to constant traffic. Continuous duration billing. Also a performance bottleneck.
- **Design B (per-user):** 10,000 DOs, but each only wakes for their user's 100 requests. Most hibernate most of the time.
- **Design C (per-user-per-hour):** Creates 240,000 DO instances per day. Each instance is cheap, but you're paying for a lot of cold starts and minimum durations.

**The nuance:** If users are bursty (all requests in 1 hour, then nothing), Design B might wake/sleep a lot. Design C might actually be cheaper in that case. Profile your actual traffic patterns.

---

## Gotcha #6: storage.get() vs Batching

**Scenario:** You need to read 5 keys from storage on each request.

**Question:** Is it cheaper to call `storage.get(['key1', 'key2', 'key3', 'key4', 'key5'])` or make 5 separate `storage.get()` calls?

**Answer:** Batched `storage.get(['key1', ...])` is cheaper.

Each storage operation has overhead. Batching reads into a single call reduces the number of billable operations. Same applies to writes—batch them when possible.

**Bonus:** Writes without intervening `await` calls are automatically coalesced into a single atomic transaction.

---

## Gotcha #7: Hibernation Confusion

**Scenario:** Your DO needs to respond to requests but also run background work every 10 minutes.

**Question:** How do you structure this so the DO can hibernate between alarm intervals, but still handle incoming requests without "waking up" a new instance that doesn't have warm state?

**Answer:** You can't rely on in-memory state across hibernation.

When a DO hibernates, in-memory state is lost. When it wakes (via request or alarm), it reconstructs state from storage.

**The fix:**
1. Store all important state in `storage` (SQLite or KV)
2. Use `blockConcurrencyWhile()` in constructor to load state on wake
3. Cache in memory for the current wake cycle only
4. Accept that every wake is potentially a "cold" wake from state perspective

---

## Gotcha #8: Fan-Out Tax

**Scenario:** You have an event that needs to notify 1,000 DOs. Each notification is tiny (100 bytes).

**Question:** What's the cost difference between a Worker making 1,000 `stub.fetch()` calls in parallel vs using a queue/alarm pattern?

**Answer:** Direct fan-out: 1,000 DO invocations billed immediately. Queue pattern: 1,000 queue messages + 1,000 DO invocations (eventually).

The queue pattern isn't cheaper in DO invocations, but it:
1. Avoids timing out the original Worker request
2. Provides retry/dead-letter handling
3. Can batch notifications if DOs are already awake

**The real cost:** 1,000 cold-start invocations can have latency spikes. If this is time-sensitive, direct fan-out is faster but more expensive in aggregate.

---

## Gotcha #9: Idempotency Key Explosion

**Scenario:** You're using DOs for idempotency. Each request gets a unique idempotency key. You create one DO per key.

**Question:** What happens if you create a new DO instance for each idempotency key that's used once?

**Answer:** You create millions of tiny DOs that each handle one request and never get cleaned up.

Each DO instance persists until explicitly deleted. Storage isn't free. You end up with a growing pile of single-use DOs.

**The fix:**
1. Use a sharded approach: hash idempotency keys into N buckets
2. Store idempotency records as rows in a single DO's SQLite table
3. Implement TTL/cleanup via alarms to delete old records
4. Consider if you actually need DO-level consistency for idempotency (maybe KV is enough?)

---

## Gotcha #10: Storage Compaction

**Scenario:** You're writing events to DO storage. Each event is 100 bytes. You write 1 event per request.

**Question:** What's the cost difference between writing 1 item per event vs batching 100 events per write?

**Answer:** Individual writes: 100x the write operations. Batched writes: 1x the write operations, but you need buffering logic.

Storage writes are billed per operation. Batching reduces operations but adds complexity (buffer in memory, flush on threshold or timer).

**The nuance:** If you're using SQLite storage, transactions are your friend. Multiple `INSERT` statements without intervening `await` are coalesced into a single transaction.

---

## Gotcha #11: waitUntil() in DOs

**Scenario:** You want to do background work after returning a response. You use `waitUntil()` like in Workers.

**Question:** Does `waitUntil()` work in DOs the same way it works in Workers?

**Answer:** No. `ctx.waitUntil()` in DOs keeps the DO alive but doesn't extend the request's lifetime. Use it for fire-and-forget work, but be aware:

1. The DO stays active (billed) until `waitUntil` promises resolve
2. If you're waiting for slow external calls, you're paying for that wait time
3. Errors in `waitUntil` code don't propagate to the original request

**The fix:** For true background work, consider using alarms or queues instead of `waitUntil`.

---

## Gotcha #12: KV vs DO Storage

**Scenario:** You need to store user session data. It's read-heavy (20 reads per write), write-rare.

**Question:** Should you use KV or DO storage?

**Answer:** For read-heavy, write-rare, eventually-consistent-OK data: **KV is cheaper**.

KV:
- Global edge caching
- Cheap reads from cache
- Writes propagate eventually (~60 seconds)
- No compute cost for reads

DO Storage:
- Strong consistency
- Every read hits the DO (compute cost)
- Co-located with compute
- Better for read-modify-write patterns

**The rule:** If you don't need strong consistency and writes are rare, KV is almost always cheaper for read-heavy workloads.

