# UserDO: Per-User Durable Objects

> Source: https://coey.dev/userdo

User data pods with live updates and boring ops.

Each user gets a Durable Object—ordered state, simple auth, instant SSE.

## The Quick Version

Give every user their own Durable Object. Read/write their profile and stream updates over SSE. It's simple, ordered, and fast.

Great for settings, presence, inboxes, and profile-driven UIs.

## Architecture

```
Client <-> Worker -> UserDO(name = user.sub)
             |             |
           JWT auth       SSE updates
```

## Infrastructure

Install the package.

```bash
# Install
bun install userdo
```

## Your Durable Object

Extend `UserDO` to add your tables and methods.

```ts
// 1) Extend UserDO (your data + logic)
import { UserDO, type Env } from "userdo/server";
import { z } from "zod";

const PostSchema = z.object({ title: z.string(), content: z.string() });

export class BlogDO extends UserDO {
  posts: any;
  constructor(state: DurableObjectState, env: Env) {
    super(state, env);
    this.posts = this.table('posts', PostSchema, { userScoped: true });
  }
  async createPost(title: string, content: string) {
    return await this.posts.create({ title, content });
  }
  async getPosts() {
    return await this.posts.orderBy('createdAt', 'desc').get();
  }
}
```

## Worker

Auth endpoints come built-in. Add your own routes.

```ts
// 2) Create Worker (auth built-in; add your endpoints)
import { createUserDOWorker, createWebSocketHandler, getUserDOFromContext } from 'userdo/server';
import type { BlogDO } from './blog-do';

const app = createUserDOWorker('BLOG_DO');
const wsHandler = createWebSocketHandler('BLOG_DO');

app.post('/api/posts', async (c) => {
  const user = c.get('user');
  if (!user) return c.json({ error: 'Unauthorized' }, 401);
  const { title, content } = await c.req.json();
  const blog = getUserDOFromContext(c, user.email, 'BLOG_DO') as unknown as BlogDO;
  const post = await blog.createPost(title, content);
  return c.json({ post });
});

export default {
  async fetch(request: Request, env: any, ctx: any) {
    if (request.headers.get('upgrade') === 'websocket') return wsHandler.fetch(request, env, ctx);
    return app.fetch(request, env, ctx);
  }
};
```

## Client

Use the browser client. It handles auth and real-time.

```ts
// 4) Browser client
import { UserDOClient } from 'userdo/client';

const client = new UserDOClient('/api');
await client.signup('user@example.com', 'password');
await client.login('user@example.com', 'password');
client.onChange('table:posts', (evt) => console.log('post changed', evt));
```

## Wrangler Config

Enable SQLite for your DO via migration.

```jsonc
{
  "main": "src/index.ts",
  "compatibility_flags": ["nodejs_compat"],
  "vars": { "JWT_SECRET": "your-jwt-secret-here" },
  "durable_objects": {
    "bindings": [ { "name": "BLOG_DO", "class_name": "BlogDO" } ]
  },
  "migrations": [ { "tag": "v1", "new_sqlite_classes": ["BlogDO"] } ]
}
```

## Built-In Endpoints

```
Built-in endpoints:
- POST /api/signup
- POST /api/login
- POST /api/logout
- GET  /api/me
- GET  /api/ws (WebSocket)
- GET  /data
- POST /data
- Organizations: /api/organizations...
```

## Organizations

Org-scoped tables and membership management.

```ts
// Organization-scoped example
import { UserDO, type Env } from 'userdo/server';
import { z } from 'zod';

const Project = z.object({ name: z.string() });
const Task = z.object({ title: z.string() });

export class TeamDO extends UserDO {
  projects: any; tasks: any;
  constructor(state: DurableObjectState, env: Env) {
    super(state, env);
    this.projects = this.table('projects', Project, { organizationScoped: true });
    this.tasks = this.table('tasks', Task, { organizationScoped: true });
  }
}
```
