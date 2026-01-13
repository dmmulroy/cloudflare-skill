# Rules of Durable Objects

> Source: https://developers.cloudflare.com/durable-objects/best-practices/rules-of-durable-objects/

Durable Objects provide a powerful primitive for building stateful, coordinated applications. Each Durable Object is a single-threaded, globally-unique instance with its own persistent storage. Understanding how to design around these properties is essential for building effective applications.

This is a guidebook on how to build more effective and correct Durable Object applications.

## When to use Durable Objects

### Use Durable Objects for stateful coordination, not stateless request handling

Workers are stateless functions: each request may run on a different instance, in a different location, with no shared memory between requests. Durable Objects are stateful compute: each instance has a unique identity, runs in a single location, and maintains state across requests.

Use Durable Objects when you need:

* **Coordination** — Multiple clients need to interact with shared state (chat rooms, multiplayer games, collaborative documents)
* **Strong consistency** — Operations must be serialized to avoid race conditions (inventory management, booking systems, turn-based games)
* **Per-entity storage** — Each user, tenant, or resource needs its own isolated database (multi-tenant SaaS, per-user data)
* **Persistent connections** — Long-lived WebSocket connections that survive across requests (real-time notifications, live updates)
* **Scheduled work per entity** — Each entity needs its own timer or scheduled task (subscription renewals, game timeouts)

Use plain Workers when you need:

* **Stateless request handling** — API endpoints, proxies, or transformations with no shared state
* **Maximum global distribution** — Requests should be handled at the nearest edge location
* **High fan-out** — Each request is independent and can be processed in parallel

#### JavaScript

```js
import { DurableObject } from "cloudflare:workers";

// Good use of Durable Objects: Seat booking requires coordination
// All booking requests for a venue must be serialized to prevent double-booking
export class SeatBooking extends DurableObject {
  async bookSeat(seatId, userId) {
    // Check if seat is already booked
    const existing = this.ctx.storage.sql
      .exec("SELECT user_id FROM bookings WHERE seat_id = ?", seatId)
      .toArray();

    if (existing.length > 0) {
      return { success: false, message: "Seat already booked" };
    }

    // Book the seat - this is safe because Durable Objects are single-threaded
    this.ctx.storage.sql.exec(
      "INSERT INTO bookings (seat_id, user_id, booked_at) VALUES (?, ?, ?)",
      seatId,
      userId,
      Date.now(),
    );

    return { success: true, message: "Seat booked successfully" };
  }
}

export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const eventId = url.searchParams.get("event") ?? "default";

    // Route to a Durable Object by event ID
    // All bookings for the same event go to the same instance
    const id = env.BOOKING.idFromName(eventId);
    const booking = env.BOOKING.get(id);

    const { seatId, userId } = await request.json();
    const result = await booking.bookSeat(seatId, userId);

    return Response.json(result, {
      status: result.success ? 200 : 409,
    });
  },
};
```

#### TypeScript

```ts
import { DurableObject } from "cloudflare:workers";

export interface Env {
  BOOKING: DurableObjectNamespace<SeatBooking>;
}

// Good use of Durable Objects: Seat booking requires coordination
// All booking requests for a venue must be serialized to prevent double-booking
export class SeatBooking extends DurableObject<Env> {
  async bookSeat(
    seatId: string,
    userId: string
  ): Promise<{ success: boolean; message: string }> {
    // Check if seat is already booked
    const existing = this.ctx.storage.sql
      .exec<{ user_id: string }>(
        "SELECT user_id FROM bookings WHERE seat_id = ?",
        seatId
      )
      .toArray();

    if (existing.length > 0) {
      return { success: false, message: "Seat already booked" };
    }

    // Book the seat - this is safe because Durable Objects are single-threaded
    this.ctx.storage.sql.exec(
      "INSERT INTO bookings (seat_id, user_id, booked_at) VALUES (?, ?, ?)",
      seatId,
      userId,
      Date.now()
    );

    return { success: true, message: "Seat booked successfully" };
  }
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const eventId = url.searchParams.get("event") ?? "default";

    // Route to a Durable Object by event ID
    // All bookings for the same event go to the same instance
    const id = env.BOOKING.idFromName(eventId);
    const booking = env.BOOKING.get(id);

    const { seatId, userId } = await request.json<{
      seatId: string;
      userId: string;
    }>();
    const result = await booking.bookSeat(seatId, userId);

    return Response.json(result, {
      status: result.success ? 200 : 409,
    });
  },
};
```

A common pattern is to use Workers as the stateless entry point that routes requests to Durable Objects when coordination is needed. The Worker handles authentication, validation, and response formatting, while the Durable Object handles the stateful logic.

## Design and sharding

### Model your Durable Objects around your "atom" of coordination

The most important design decision is choosing what each Durable Object represents. Create one Durable Object per logical unit that needs coordination: a chat room, a game session, a document, a user's data, or a tenant's workspace.

This is the key insight that makes Durable Objects powerful. Instead of a shared database with locks, each "atom" of your application gets its own single-threaded execution environment with private storage.

#### JavaScript

```js
import { DurableObject } from "cloudflare:workers";

// Each chat room is its own Durable Object instance
export class ChatRoom extends DurableObject {
  async sendMessage(userId, message) {
    // All messages to this room are processed sequentially by this single instance.
    // No race conditions, no distributed locks needed.
    this.ctx.storage.sql.exec(
      "INSERT INTO messages (user_id, content, created_at) VALUES (?, ?, ?)",
      userId,
      message,
      Date.now(),
    );
  }
}

export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const roomId = url.searchParams.get("room") ?? "lobby";

    // Each room ID maps to exactly one Durable Object instance globally
    const id = env.CHAT_ROOM.idFromName(roomId);
    const stub = env.CHAT_ROOM.get(id);

    await stub.sendMessage("user-123", "Hello, room!");
    return new Response("Message sent");
  },
};
```

#### TypeScript

```ts
import { DurableObject } from "cloudflare:workers";

export interface Env {
  CHAT_ROOM: DurableObjectNamespace<ChatRoom>;
}

// Each chat room is its own Durable Object instance
export class ChatRoom extends DurableObject<Env> {
  async sendMessage(userId: string, message: string) {
    // All messages to this room are processed sequentially by this single instance.
    // No race conditions, no distributed locks needed.
    this.ctx.storage.sql.exec(
      "INSERT INTO messages (user_id, content, created_at) VALUES (?, ?, ?)",
      userId,
      message,
      Date.now()
    );
  }
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const roomId = url.searchParams.get("room") ?? "lobby";

    // Each room ID maps to exactly one Durable Object instance globally
    const id = env.CHAT_ROOM.idFromName(roomId);
    const stub = env.CHAT_ROOM.get(id);

    await stub.sendMessage("user-123", "Hello, room!");
    return new Response("Message sent");
  },
};
```

> **Note:** If you have global application or user configuration that you need to access frequently (on every request), consider using [Workers KV](https://developers.cloudflare.com/kv/) instead.

Do not create a single "global" Durable Object that handles all requests:

```js
// Bad: A single Durable Object handling ALL chat rooms
export class ChatRoom extends DurableObject {
  async sendMessage(roomId, userId, message) {
    // All messages for ALL rooms go through this single instance.
    // This becomes a bottleneck as traffic grows.
    this.ctx.storage.sql.exec(
      "INSERT INTO messages (room_id, user_id, content) VALUES (?, ?, ?)",
      roomId,
      userId,
      message,
    );
  }
}

export default {
  async fetch(request, env) {
    // Bad: Always using the same ID means one global instance
    const id = env.CHAT_ROOM.idFromName("global");
    const stub = env.CHAT_ROOM.get(id);

    await stub.sendMessage("room-123", "user-456", "Hello!");
    return new Response("Sent");
  },
};
```

### Use deterministic IDs for predictable routing

Use `getByName()` with meaningful, deterministic strings for consistent routing. The same input always produces the same Durable Object ID, ensuring requests for the same logical entity always reach the same instance.

```js
// Good: Deterministic ID from a meaningful string
// All requests for "game-abc123" go to the same Durable Object
const stub = env.GAME_SESSION.getByName(gameId);
```

Creating a stub does not instantiate or wake up the Durable Object. The Durable Object is only activated when you call a method on the stub.

Use `newUniqueId()` only when you need a new, random instance and will store the mapping externally:

```js
// newUniqueId() creates a random ID - useful when creating new instances
// You must store this ID somewhere (e.g., D1) to find it again later
const id = env.GAME_SESSION.newUniqueId();
const stub = env.GAME_SESSION.get(id);

// Store the mapping: gameCode -> id.toString()
// await env.DB.prepare("INSERT INTO games (code, do_id) VALUES (?, ?)").bind(gameCode, id.toString()).run();
```

### Use parent-child relationships for related entities

Do not put all your data in a single Durable Object. When you have hierarchical data (workspaces containing projects, game servers managing matches), create separate child Durable Objects for each entity. The parent coordinates and tracks children, while children handle their own state independently.

This enables parallelism: operations on different children can happen concurrently, while each child maintains its own single-threaded consistency.

With this pattern:

* Listing matches only queries the parent (children stay hibernated)
* Different matches process player actions in parallel
* Each match has its own SQLite database for player data

### Consider location hints for latency-sensitive applications

By default, a Durable Object is created near the location of the first request it receives. For most applications, this works well. However, you can provide a location hint to influence where the Durable Object is created.

```js
// Provide a location hint for where this Durable Object should be created
const id = env.GAME_SESSION.idFromName(gameId, {
  locationHint: region,
});
```

Location hints are suggestions, not guarantees. Refer to [Data location](https://developers.cloudflare.com/durable-objects/reference/data-location/) for available regions and details.

## Storage and state

### Use SQLite-backed Durable Objects

[SQLite storage](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/) is the recommended storage backend for new Durable Objects. It provides a familiar SQL API for relational queries, indexes, transactions, and better performance than the legacy key-value storage backed Durable Objects. SQLite Durable Objects also support the KV API in synchronous and asynchronous versions.

Configure your Durable Object class to use SQLite storage in your Wrangler configuration:

**wrangler.jsonc:**
```jsonc
{
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["ChatRoom"] }
  ]
}
```

**wrangler.toml:**
```toml
[[migrations]]
tag = "v1"
new_sqlite_classes = [ "ChatRoom" ]
```

### Initialize storage and run migrations in the constructor

Use `blockConcurrencyWhile()` in the constructor to run migrations and initialize state before any requests are processed. This ensures your schema is ready and prevents race conditions during initialization.

```ts
export class ChatRoom extends DurableObject<Env> {
  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);

    // blockConcurrencyWhile() ensures no requests are processed until this completes
    ctx.blockConcurrencyWhile(async () => {
      await this.migrate();
    });
  }

  private async migrate() {
    // Check current schema version
    const version =
      this.ctx.storage.sql
        .exec<{ version: number }>("PRAGMA user_version")
        .one()?.version ?? 0;

    if (version < 1) {
      this.ctx.storage.sql.exec(`
        CREATE TABLE IF NOT EXISTS messages (
          id INTEGER PRIMARY KEY AUTOINCREMENT,
          user_id TEXT NOT NULL,
          content TEXT NOT NULL,
          created_at INTEGER NOT NULL
        );
        CREATE INDEX IF NOT EXISTS idx_messages_created_at ON messages(created_at);
        PRAGMA user_version = 1;
      `);
    }

    if (version < 2) {
      // Future migration: add a new column
      this.ctx.storage.sql.exec(`
        ALTER TABLE messages ADD COLUMN edited_at INTEGER;
        PRAGMA user_version = 2;
      `);
    }
  }
}
```

### Understand the difference between in-memory state and persistent storage

Durable Objects provide multiple state management layers, each with different characteristics:

| Type | Speed | Persistence | Use Case |
| - | - | - | - |
| In-memory (class properties) | Fastest | Lost on eviction or crash | Caching, active connections |
| SQLite storage | Fast | Durable across restarts | Primary data storage |
| External (R2, D1) | Variable | Durable, cross-DO accessible | Large files, shared data |

In-memory state is **not preserved** if the Durable Object is evicted from memory due to inactivity, or if it crashes from an uncaught exception. Always persist important state to SQLite storage.

> **Warning:** If an uncaught exception occurs in your Durable Object, the runtime may terminate the instance. Any in-memory state will be lost, but SQLite storage remains intact. Always persist critical state to storage before performing operations that might fail.

### Create indexes for frequently-queried columns

Just like any database, indexes dramatically improve read performance for frequently-filtered columns. The cost is slightly more storage and marginally slower writes.

```sql
-- Index for queries filtering by user
CREATE INDEX IF NOT EXISTS idx_messages_user_id ON messages(user_id);

-- Index for time-based queries (recent messages)
CREATE INDEX IF NOT EXISTS idx_messages_created_at ON messages(created_at);

-- Composite index for user + time queries
CREATE INDEX IF NOT EXISTS idx_messages_user_time ON messages(user_id, created_at);
```

### Understand how input and output gates work

While Durable Objects are single-threaded, JavaScript's `async`/`await` can allow multiple requests to interleave execution while a request waits for the result of an asynchronous operation. Cloudflare's runtime uses **input gates** and **output gates** to prevent data races and ensure correctness by default.

**Input gates** block new events (incoming requests, fetch responses) while synchronous JavaScript execution is in progress. Awaiting async operations like `fetch()` or KV storage methods opens the input gate, allowing other requests to interleave. However, storage operations provide special protection.

**Output gates** hold outgoing network messages (responses, fetch requests) until pending storage writes complete. This ensures clients never see confirmation of data that has not been persisted.

**Write coalescing:** Multiple storage writes without intervening `await` calls are automatically batched into a single atomic implicit transaction:

```ts
async transfer(fromId: string, toId: string, amount: number) {
  // Good: These writes are coalesced into one atomic transaction
  this.ctx.storage.sql.exec(
    "UPDATE accounts SET balance = balance - ? WHERE id = ?",
    amount,
    fromId
  );
  this.ctx.storage.sql.exec(
    "UPDATE accounts SET balance = balance + ? WHERE id = ?",
    amount,
    toId
  );
  this.ctx.storage.sql.exec(
    "INSERT INTO transfers (from_id, to_id, amount, created_at) VALUES (?, ?, ?, ?)",
    fromId,
    toId,
    amount,
    Date.now()
  );
  // All three writes commit together atomically
}
```

For more details, see [Durable Objects: Easy, Fast, Correct — Choose three](https://blog.cloudflare.com/durable-objects-easy-fast-correct-choose-three/) and the [glossary](https://developers.cloudflare.com/durable-objects/reference/glossary/).

### Avoid race conditions with non-storage I/O

Input gates only protect during storage operations. Non-storage I/O like `fetch()` or writing to R2 allows other requests to interleave, which can cause race conditions.

To handle this, use optimistic locking (check-and-set) patterns: read a version number before the external call, then verify it has not changed before writing.

> **Note:** With the legacy KV storage backend, use the [`transaction()`](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/#transaction) method for atomic read-modify-write operations across async boundaries.

### Use `blockConcurrencyWhile()` sparingly

The [`blockConcurrencyWhile()`](https://developers.cloudflare.com/durable-objects/api/state/#blockconcurrencywhile) method guarantees that no other events are processed until the provided callback completes, even if the callback performs asynchronous I/O. This is useful for operations that must be atomic, such as state initialization from storage in the constructor.

Because `blockConcurrencyWhile()` blocks *all* concurrency unconditionally, it significantly reduces throughput. If each call takes ~5ms, that individual Durable Object is limited to approximately 200 requests/second. Reserve it for initialization and migrations, not regular request handling. For normal operations, rely on input/output gates and write coalescing instead.

For atomic read-modify-write operations during request handling, prefer [`transaction()`](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/#transaction) over `blockConcurrencyWhile()`. Transactions provide atomicity for storage operations without blocking unrelated concurrent requests.

> **Warning:** Using `blockConcurrencyWhile()` across I/O operations (such as `fetch()`, KV, R2, or other external API calls) is an anti-pattern. This is equivalent to holding a lock across I/O in other languages or concurrency frameworks — it blocks all other requests while waiting for slow external operations, severely degrading throughput. Keep `blockConcurrencyWhile()` callbacks fast and limited to local storage operations.

## Communication and API design

### Use RPC methods instead of the `fetch()` handler

Projects with a [compatibility date](https://developers.cloudflare.com/workers/configuration/compatibility-flags/) of `2024-04-03` or later should use RPC methods. RPC is more ergonomic, provides better type safety, and eliminates manual request/response parsing.

Define public methods on your Durable Object class, and call them directly from stubs with full TypeScript support.

Refer to [Invoke methods](https://developers.cloudflare.com/durable-objects/best-practices/create-durable-object-stubs-and-send-requests/) for more details on RPC and the legacy `fetch()` handler.

### Initialize Durable Objects explicitly with an `init()` method

Durable Objects do not know their own name or ID from within. If your Durable Object needs to know its identity (for example, to store a reference to itself or to communicate with related objects), you must explicitly initialize it.

### Always `await` RPC calls

When calling methods on a Durable Object stub, always use `await`. Unawaited calls create dangling promises, causing errors to be swallowed and return values to be lost.

```ts
// Bad: Not awaiting the call
// The message ID is lost, and any errors are swallowed
stub.sendMessage("user-123", "Hello");

// Good: Properly awaited
const messageId = await stub.sendMessage("user-123", "Hello");
```

## Error handling

### Handle errors and use exception boundaries

Uncaught exceptions in a Durable Object can leave it in an unknown state and may cause the runtime to terminate the instance. Wrap risky operations in try/catch blocks, and handle errors appropriately.

When calling Durable Objects from a Worker, errors may include `.retryable` and `.overloaded` properties indicating whether the operation can be retried. For transient failures, implement exponential backoff to avoid overwhelming the system.

Refer to [Error handling](https://developers.cloudflare.com/durable-objects/best-practices/error-handling/) for details on error properties, retry strategies, and exponential backoff patterns.

## WebSockets and real-time

### Use the Hibernatable WebSockets API for cost efficiency

The Hibernatable WebSockets API allows Durable Objects to sleep while maintaining WebSocket connections. This significantly reduces costs for applications with many idle connections.

```ts
export class ChatRoom extends DurableObject<Env> {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === "/websocket") {
      // Check for WebSocket upgrade
      if (request.headers.get("Upgrade") !== "websocket") {
        return new Response("Expected WebSocket", { status: 400 });
      }

      const pair = new WebSocketPair();
      const [client, server] = Object.values(pair);

      // Accept the WebSocket with Hibernation API
      this.ctx.acceptWebSocket(server);

      return new Response(null, { status: 101, webSocket: client });
    }

    return new Response("Not found", { status: 404 });
  }

  // Called when a message is received (even after hibernation)
  async webSocketMessage(ws: WebSocket, message: string | ArrayBuffer) {
    const data = typeof message === "string" ? message : "binary data";

    // Broadcast to all connected clients
    for (const client of this.ctx.getWebSockets()) {
      if (client !== ws && client.readyState === WebSocket.OPEN) {
        client.send(data);
      }
    }
  }

  // Called when a WebSocket is closed
  async webSocketClose(
    ws: WebSocket,
    code: number,
    reason: string,
    wasClean: boolean
  ) {
    console.log(`WebSocket closed: ${code} ${reason}`);
  }

  // Called when a WebSocket error occurs
  async webSocketError(ws: WebSocket, error: unknown) {
    console.error("WebSocket error:", error);
  }
}
```

With the Hibernation API, your Durable Object can go to sleep when there is no active JavaScript execution, but WebSocket connections remain open. When a message arrives, the Durable Object wakes up automatically.

Refer to [WebSockets](https://developers.cloudflare.com/durable-objects/best-practices/websockets/) for more details.

### Use `serializeAttachment()` to persist per-connection state

WebSocket attachments let you store metadata for each connection that survives hibernation. Use this for user IDs, session tokens, or other per-connection data.

## Scheduling and lifecycle

### Use alarms for per-entity scheduled tasks

Each Durable Object can schedule its own future work using the [Alarms API](https://developers.cloudflare.com/durable-objects/api/alarms/), allowing a Durable Object to execute background tasks on any interval without an incoming request, RPC call, or WebSocket message.

Key points about alarms:

* **`setAlarm(timestamp)`** schedules the `alarm()` handler to run at any time in the future (millisecond precision)
* **Alarms do not repeat automatically** — you must call `setAlarm()` again to schedule the next execution
* **Only schedule alarms when there is work to do** — avoid waking up every Durable Object on short intervals (seconds), as each alarm invocation incurs costs

### Make alarm handlers idempotent

In rare cases, alarms may fire more than once. Your `alarm()` handler should be safe to run multiple times without causing issues.

### Clean up storage with `deleteAll()`

To fully clear a Durable Object's storage, call `deleteAll()`. Simply deleting individual keys or dropping tables is not sufficient, as some internal metadata may remain. If you have alarms set, delete those first.

```ts
async clearStorage() {
  // If you have an alarm set, delete it first
  await this.ctx.storage.deleteAlarm();

  // Delete all storage
  await this.ctx.storage.deleteAll();

  // The Durable Object instance still exists, but with empty storage
  // A subsequent request will find no data
}
```

## Anti-patterns to avoid

### Do not use a single Durable Object as a global singleton

A single Durable Object handling all traffic becomes a bottleneck. While async operations allow request interleaving, all synchronous JavaScript execution is single-threaded, and storage operations provide serialization guarantees that limit throughput.

A common mistake is using a Durable Object for global rate limiting or global counters. This funnels all traffic through a single instance.

This pattern does not scale. As traffic increases, the single Durable Object becomes a chokepoint. Instead, identify natural coordination boundaries in your application (per user, per room, per document) and create separate Durable Objects for each.

## Testing and migrations

### Test with Vitest and plan for class migrations

Use `@cloudflare/vitest-pool-workers` for testing Durable Objects. The integration provides isolated storage per test and utilities for direct instance access.

```ts
import {
  env,
  runInDurableObject,
  runDurableObjectAlarm,
} from "cloudflare:test";
import { describe, it, expect } from "vitest";

describe("ChatRoom", () => {
  // Each test gets isolated storage automatically
  it("should send and retrieve messages", async () => {
    const id = env.CHAT_ROOM.idFromName("test-room");
    const stub = env.CHAT_ROOM.get(id);

    // Call RPC methods directly on the stub
    await stub.sendMessage("user-1", "Hello!");
    await stub.sendMessage("user-2", "Hi there!");

    const messages = await stub.getMessages(10);
    expect(messages).toHaveLength(2);
  });

  it("can access instance internals and trigger alarms", async () => {
    const id = env.CHAT_ROOM.idFromName("test-room");
    const stub = env.CHAT_ROOM.get(id);

    // Access storage directly for verification
    await runInDurableObject(stub, async (instance, state) => {
      const count = state.storage.sql
        .exec<{ count: number }>("SELECT COUNT(*) as count FROM messages")
        .one();
      expect(count.count).toBe(0); // Fresh instance due to test isolation
    });

    // Trigger alarms immediately without waiting
    const alarmRan = await runDurableObjectAlarm(stub);
    expect(alarmRan).toBe(false); // No alarm was scheduled
  });
});
```

Configure Vitest in your `vitest.config.ts`:

```ts
import { defineWorkersConfig } from "@cloudflare/vitest-pool-workers/config";

export default defineWorkersConfig({
  test: {
    poolOptions: {
      workers: {
        wrangler: { configPath: "./wrangler.toml" },
      },
    },
  },
});
```

For schema changes, run migrations in the constructor using `blockConcurrencyWhile()`. For class renames or deletions, use Wrangler migrations:

**wrangler.jsonc:**
```jsonc
{
  "migrations": [
    // Rename a class
    { "tag": "v2", "renamed_classes": [{ "from": "OldChatRoom", "to": "ChatRoom" }] },
    // Delete a class (removes all data!)
    { "tag": "v3", "deleted_classes": ["DeprecatedRoom"] }
  ]
}
```

**wrangler.toml:**
```toml
[[migrations]]
tag = "v2"

  [[migrations.renamed_classes]]
  from = "OldChatRoom"
  to = "ChatRoom"

[[migrations]]
tag = "v3"
deleted_classes = [ "DeprecatedRoom" ]
```

Refer to [Durable Objects migrations](https://developers.cloudflare.com/durable-objects/reference/durable-objects-migrations/) for more details on class migrations, and [Testing with Durable Objects](https://developers.cloudflare.com/durable-objects/examples/testing-with-durable-objects/) for comprehensive testing patterns including SQLite queries and alarm testing.
