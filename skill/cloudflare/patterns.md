# Common Cloudflare Patterns

Reusable architecture patterns combining multiple products.

## Full-Stack Web App
```
Pages (frontend) + Pages Functions (API) + D1 (database)
```

## Real-Time Collaboration
```
Workers (entry) + Durable Objects (state) + WebSockets
```

## RAG / AI Search
```
Workers AI (embeddings + LLM) + Vectorize (vector store) + D1/KV (metadata)
```

## Background Processing
```
Workers (producer) + Queues (buffer) + Workers (consumer) + R2 (results)
```

## Multi-Region Database Access
```
Hyperdrive (pooling/caching) → Existing Postgres/MySQL
```
