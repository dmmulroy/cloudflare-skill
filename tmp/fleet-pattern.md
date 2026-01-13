# Fleet Pattern: Hierarchical Durable Objects

> Source: https://coey.dev/fleet-pattern

Infinite nesting of manager/agent relationships through URL paths.

## The Hierarchical Coordination Challenge

**Problem:** Building hierarchical systems with manager/agent relationships requires complex coordination, state management, and real-time communication.

**Solution:** Fleet Pattern uses URL-based routing to create infinite nesting of Durable Objects, each capable of managing child agents with real-time WebSocket communication.

```
Client → ManagerDO → AgentDO (N)
                ↑          ↓
             heartbeats  results
                ↓          ↓
            WebSocket   Cascading
            Updates     Operations
```

## Architecture Overview

### Core Features

- **URL-based hierarchy** - Each path creates a unique DO
- **Infinite nesting** - Unlimited depth of manager/agent relationships
- **Real-time communication** - WebSocket-based state synchronization
- **Cascading operations** - Delete operations propagate down the tree
- **Message routing** - Direct and broadcast messaging between agents

### Tech Stack

- **Hono** - Edge-first web framework
- **Durable Objects** - Persistent state and WebSocket handling
- **TypeScript** - End-to-end type safety
- **WebSocket API** - Real-time bidirectional communication
- **Cloudflare Workers** - Global edge deployment

## Key Innovations

- **URL-based hierarchy** - `/team1/project1/task1` creates nested DO structure
- **Unified DO class** - Single class handles both manager and agent roles
- **Dynamic DO creation** - Agents created on-demand based on URL paths
- **Real-time state sync** - WebSocket connections maintain live state updates
- **Cascading deletion** - Removing a manager deletes all child agents recursively
- **Message routing** - Direct messages to specific agents or broadcast to all children

## Quick Start

```bash
# Clone and setup
git clone https://github.com/acoyfellow/fleet-pattern
cd fleet-pattern
bun install

# Start development server
bun run dev

# Test hierarchy
# http://localhost:8787/ (root manager)
# http://localhost:8787/team1 (team manager)
# http://localhost:8787/team1/project1 (project manager)

# Deploy to Cloudflare
bun run deploy
```

Creates a complete hierarchical system with real-time communication and cascading operations.

## Main Worker Routing

### URL-based Durable Object Creation

```ts
// src/index.ts - Main Worker with URL-based routing
import { Hono } from 'hono'
import { DurableObjectNamespace, DurableObjectState } from '@cloudflare/workers-types'
import type { Request as CFRequest } from '@cloudflare/workers-types'

interface Env {
  FLEET_DO: DurableObjectNamespace,
}

// The Worker: routes all requests through Hono
const app = new Hono<{ Bindings: Env }>()

// Route everything else to DOs based on URL path
app.all('*', async (c) => {
  const path = new URL(c.req.url).pathname
  const parts = path.split('/').filter(Boolean)
  const doName = parts.length === 0 ? '/' : `/${parts.join('/')}`

  const id = c.env.FLEET_DO.idFromName(doName)
  const stub = c.env.FLEET_DO.get(id)
  return stub.fetch(c.req.raw as CFRequest)
})

export default {
  fetch: app.fetch
}
```

## Durable Object Implementation

### Unified Manager/Agent Class

```ts
// FleetDO class - Unified Manager/Agent implementation
export class FleetDO {
  private app = new Hono()
  private connections = new Set<WebSocket>()

  constructor(private durableState: DurableObjectState, private env: Env) {
    this.app.get('*', c => {
      const upgradeHeader = c.req.header('Upgrade')
      if (upgradeHeader?.toLowerCase() === 'websocket') {
        return this.handleWebSocket(c)
      }
      return this.handleView(c)
    })

    // Handle cascading deletion
    this.app.delete('*', async () => {
      const data = await this.getState()

      if (data?.agents) {
        const path = new URL(this.durableState.id.toString()).pathname
        for (const agent of data.agents) {
          const childPath = path === '/' ? `/${agent}` : `${path}/${agent}`
          const childId = this.env.FLEET_DO.idFromName(childPath)
          const childStub = this.env.FLEET_DO.get(childId)
          await childStub.fetch(new Request(childPath, { method: 'DELETE' }))
        }
      }

      for (const ws of this.connections) {
        ws.close(1000, 'Agent deleted')
      }

      await this.durableState.storage.deleteAll()
      return new Response('OK')
    })
  }

  private async getState(): Promise<AgentState> {
    return await this.durableState.storage.get<AgentState>('data') || {
      data: { count: 0 },
      agents: []
    }
  }

  private async setState(data: AgentState): Promise<void> {
    await this.durableState.storage.put('data', data)
  }
}
```

## WebSocket Message Handling

### Real-time Communication Protocol

```ts
// WebSocket message handling for real-time communication
server.addEventListener('message', async event => {
  try {
    const msg = JSON.parse(event.data as string) as WSMessage
    const data = await this.getState()
    const path = new URL(c.req.url).pathname
    const senderName = path.split('/').filter(Boolean).pop() || 'root'

    switch (msg.type) {
      case 'increment':
        data.data.count++
        await this.setState(data)
        break

      case 'createAgent':
        if (!msg.payload?.name) throw new Error('Agent name required')
        if (!this.validateAgentName(msg.payload.name)) {
          throw new Error('Invalid agent name')
        }
        if (data.agents.includes(msg.payload.name)) {
          throw new Error('Agent already exists')
        }
        data.agents.push(msg.payload.name)
        await this.setState(data)
        break

      case 'deleteAgent':
        if (!msg.payload?.name) throw new Error('Agent name required')
        const index = data.agents.indexOf(msg.payload.name)
        if (index === -1) throw new Error('Agent not found')
        
        // Cascading deletion
        const childPath = path === '/' ? `/${msg.payload.name}` : `${path}/${msg.payload.name}`
        const childId = this.env.FLEET_DO.idFromName(childPath)
        const childStub = this.env.FLEET_DO.get(childId)
        await childStub.fetch(new Request(`https://internal${childPath}`, { method: 'DELETE' }))

        data.agents.splice(index, 1)
        await this.setState(data)
        break

      case 'sendMessage':
        // Direct message to specific agent
        const recipientPath = path === '/' ? `/${msg.payload.recipient}` : `${path}/${msg.payload.recipient}`
        const recipientId = this.env.FLEET_DO.idFromName(recipientPath)
        const recipientStub = this.env.FLEET_DO.get(recipientId)

        await recipientStub.fetch(new Request(`https://internal${recipientPath}/_message`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            type: 'message',
            payload: {
              message: msg.payload.message,
              sender: senderName
            }
          })
        }))
        break

      case 'broadcast':
        // Broadcast to all child agents
        for (const agent of data.agents) {
          const childPath = path === '/' ? `/${agent}` : `${path}/${agent}`
          const childId = this.env.FLEET_DO.idFromName(childPath)
          const childStub = this.env.FLEET_DO.get(childId)

          await childStub.fetch(new Request(`https://internal${childPath}/_message`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              type: 'message',
              payload: {
                message: msg.payload.message,
                sender: senderName
              }
            })
          }))
        }
        break
    }

    this.broadcast({ type: 'state', payload: data })

  } catch (err) {
    server.send(JSON.stringify({
      type: 'error',
      payload: { error: err.message }
    }))
  }
})
```

## Message Protocol

### WebSocket Communication Types

```ts
// WebSocket message protocol
interface WSMessage {
  type: 'increment' | 'createAgent' | 'deleteAgent' | 'sendMessage' | 'broadcast';
  payload?: {
    name?: string;
    message?: string;
    recipient?: string;
  };
}

interface WSResponse {
  type: 'state' | 'error' | 'message' | 'broadcast';
  payload: {
    data?: { count: number };
    agents?: string[];
    error?: string;
    message?: string;
    sender?: string;
  };
}

// Message flow examples
const messages = {
  // Local state change
  increment: { type: 'increment' },
  
  // Hierarchy management
  createAgent: { type: 'createAgent', payload: { name: 'newAgent' } },
  deleteAgent: { type: 'deleteAgent', payload: { name: 'oldAgent' } },
  
  // Communication
  sendMessage: { type: 'sendMessage', payload: { recipient: 'agent1', message: 'Hello!' } },
  broadcast: { type: 'broadcast', payload: { message: 'System update' } }
};
```

## Client-Side Integration

### Real-time UI Updates

```js
// Real-time UI updates with WebSocket
function updateUI(state) {
  // Update counter
  document.getElementById('count').textContent = state.data.count;

  // Update agents list
  const agentsList = document.getElementById('agents');
  agentsList.innerHTML = '';

  if (state.agents.length === 0) {
    agentsList.innerHTML = '<li class="no-agents">No agents</li>';
    return;
  }

  state.agents.forEach(name => {
    const li = document.createElement('li');
    li.innerHTML = `
      <div class="agent-row">
        <a href="${window.location.pathname === '/' ? '' : window.location.pathname}/${name}">${name}</a>
        <div class="agent-controls">
          <input type="text" placeholder="Message" class="message-input">
          <button onclick="sendMessageTo('${name}')" class="send-btn">Send</button>
          <button onclick="deleteAgent('${name}')" class="delete-btn">Delete</button>
        </div>
      </div>
    `;
    agentsList.appendChild(li);
  });
}

function sendMessageTo(recipient) {
  const row = document.querySelector(`[onclick="sendMessageTo('${recipient}')"]`).closest('.agent-row');
  const message = row.querySelector('.message-input').value.trim();

  if (message) {
    sendMessage({
      type: 'sendMessage',
      payload: { recipient, message }
    });
    row.querySelector('.message-input').value = '';
  }
}
```

## Configuration

### Wrangler Configuration

```toml
# wrangler.toml - Cloudflare Workers configuration
name = "fleet"
main = "src/index.ts"
compatibility_date = "2024-01-01"

assets = { directory = "public" }

[build.upload]
format = "modules"

[durable_objects]
bindings = [
  { name = "FLEET_DO", class_name = "FleetDO" }
]

[[migrations]]
tag = "v1"
new_classes = ["FleetDO"]

[observability.logs]
enabled = true
```

## Architecture Patterns

### URL-Based Hierarchy

- Each path segment creates a unique DO
- Infinite nesting depth supported
- Clean separation of concerns
- Natural resource organization

### Real-Time Communication

- WebSocket connections per DO
- State synchronization across clients
- Direct and broadcast messaging
- Automatic reconnection handling

### Cascading Operations

- Delete operations propagate down tree
- Automatic cleanup of child resources
- Safe resource management
- Consistent state maintenance

### Unified Implementation

- Single class for all roles
- Role determined by context
- Simplified codebase
- Consistent behavior patterns

## Production Use Cases

| Use Case | Description |
|----------|-------------|
| **Real-Time Collaborative IDE** | Each file is a DO with operational transform engine. Real-time cursors, editing, and file-specific permissions. |
| **Distributed Task Runner** | Each stage manages its own tasks with status communication up the chain. Automatic retry and failure management. |
| **IoT Device Management** | Each level manages device fleet with real-time sensor data aggregation and hierarchical monitoring. |
| **Game Server Infrastructure** | Each instance is a game server with real-time player state management and instance-to-instance communication. |
| **Content Management System** | Each page manages its own content and cache with real-time preview and hierarchical permissions. |
| **Distributed Chat System** | Each thread manages its own messages with real-time presence indicators and cross-thread notifications. |

## Scaling Patterns

### Multi-tenant and Geographic Distribution

```ts
// Scaling patterns for different use cases

// 1. Geographic Distribution
const region = getClosestRegion(clientIP);
const obj = env.FLEET_DO.getByName(`${region}-team1-project1`);

// 2. Tenant Isolation
const obj = env.FLEET_DO.getByName(`tenant-${tenantId}-team1`);

// 3. Feature-based Sharding
const obj = env.FLEET_DO.getByName(`feature-${featureId}-instance-${instanceId}`);

// 4. Time-based Partitioning
const date = new Date().toISOString().split('T')[0];
const obj = env.FLEET_DO.getByName(`${date}-analytics`);

// 5. Load-based Distribution
const shardId = hash(userId) % numShards;
const obj = env.FLEET_DO.getByName(`shard-${shardId}-user-${userId}`);
```

## Performance Characteristics

| Metric | Value |
|--------|-------|
| **DO Creation** | On-demand based on URL access patterns |
| **State Management** | 128MB storage per Durable Object instance |
| **WebSocket Connections** | 1,000 concurrent per DO instance |
| **Message Latency** | Sub-100ms for direct agent communication |
| **Hierarchy Depth** | Unlimited nesting with URL path limits |
| **Global Distribution** | 200+ edge locations worldwide |

## Development Workflow

1. **Clone Repository:** `git clone https://github.com/acoyfellow/fleet-pattern`
2. **Install Dependencies:** `bun install` sets up everything
3. **Local Development:** `bun run dev` starts with hot reloading
4. **Test Hierarchy:** Navigate to different URL paths to test nesting
5. **Deploy:** `bun run deploy` deploys to Cloudflare Workers

## Operational Considerations

- **Durable Object limits:** 1,000 concurrent instances per account
- **Storage limits:** 128MB per Durable Object instance
- **WebSocket limits:** 1,000 concurrent connections per DO
- **URL path limits:** 8,192 characters maximum path length
- **Cold starts:** ~10-50ms for new Durable Object instances
- **Message validation:** Input sanitization and error handling

## Security Features

- **Input validation:** Agent names restricted to alphanumeric, dash, underscore (1-32 chars)
- **WebSocket security:** Proper connection lifecycle management
- **Cascading deletion safety:** Hierarchical cleanup with error handling
- **Message sanitization:** JSON parsing with error boundaries
- **Resource isolation:** Each DO instance is completely isolated
- **Rate limiting:** Built-in protection against abuse
