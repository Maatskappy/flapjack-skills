---
name: flapjack-integration
version: 1.4.2
author: Flapjack
description: Use when implementing Flapjack AI agents into an existing or new application — installing the SDK, setting up FlapjackClient, creating threads, streaming messages, embedding chat UIs with React hooks, or connecting to the Flapjack API. Triggers on mentions of flapjack, @maats/flapjack, FlapjackClient, useChat with flapjack, or AI agent embedding.
tags:
  - ai-agent
  - chatbot
  - sdk
  - react
  - streaming
homepage: https://flapjack.chat
---

# Flapjack Integration

## Overview

Flapjack is a hosted platform for deploying AI agents. You create agents via the dashboard at flapjack.chat, then embed them into any product using `@maats/flapjack` (TypeScript client + React hooks + pre-built components). The SDK communicates with the Flapjack API over SSE for real-time streaming.

**Architecture:** `Client (SDK) → Flapjack API (Next.js + Supabase) → Tensorlake Runtime (Python)`

## When to Use

- Adding an AI agent/chatbot to an existing app
- Building a new app with Flapjack-powered AI
- Integrating Flapjack streaming into a custom UI
- Setting up FlapjackClient for server-side or client-side use

**When NOT to use:** Building/modifying the Flapjack platform itself (that's the flapjack repo, not SDK usage).

## Quick Reference

| What | How |
|------|-----|
| Install SDK | `npm install @maats/flapjack` (v0.4.1+) |
| API key format | `fj_live_...` (from flapjack.chat dashboard → Keys) |
| Base URL (production) | `https://api.flapjack.dev` |
| Base URL (local dev) | `http://localhost:3000` |
| Auth (SDK) | `FLAPJACK_API_KEY=fj_live_...` — the SDK handles auth internally |
| Auth (direct REST) | `Authorization: Bearer fj_live_...` or Supabase JWT |
| Streaming protocol | Server-Sent Events (SSE) |

## Environment Setup

```bash
# .env or .env.local in consumer app
FLAPJACK_API_KEY=fj_live_...              # from flapjack.chat → Keys
FLAPJACK_BASE_URL=https://api.flapjack.dev  # or localhost:3000 for local dev

# For React apps (client-side) — DEV ONLY
NEXT_PUBLIC_FLAPJACK_API_KEY=fj_live_...
NEXT_PUBLIC_FLAPJACK_BASE_URL=https://api.flapjack.dev
```

**Security:** Never expose `fj_live_` keys in client-side code in production. Use a server-side proxy route that adds the API key, then call that route from the client.

## Core Integration: TypeScript Client

```typescript
import { FlapjackClient } from '@maats/flapjack';

const client = new FlapjackClient({
  apiKey: process.env.FLAPJACK_API_KEY!,
  baseUrl: process.env.FLAPJACK_BASE_URL ?? 'https://api.flapjack.dev',
});

// List available agents
const agents = await client.listAgents();

// Create a conversation thread
const thread = await client.createThread(agents[0].id);

// Send a message and stream the response
for await (const event of client.sendMessage(thread.id, 'Hello!')) {
  switch (event.type) {
    case 'meta':
      console.log('Stream started at:', event.startedAt);
      break;
    case 'token':
      process.stdout.write(event.delta); // streaming text chunk
      break;
    case 'tool_call':
      console.log('Tool called:', event.tool.name);
      break;
    case 'tool_executing':
      console.log('Executing:', event.tool_name);
      break;
    case 'tool_result':
      console.log('Tool result:', event.tool_name, event.result);
      break;
    case 'requires_action':
      console.log('Client tools requested:', event.toolCalls.map(t => t.name));
      break;
    case 'done':
      console.log('\nFinal:', event.content);
      break;
    case 'error':
      console.error('Error:', event.code, event.detail);
      break;
  }
}
```

### Custom Tools (Client-Side Execution)

```typescript
// Define tools the agent can call — executed in your code, not server-side
const tools = [{
  name: 'lookup_order',
  description: 'Look up an order by ID',
  parameters: {
    type: 'object',
    properties: { orderId: { type: 'string' } },
    required: ['orderId'],
  },
}];

for await (const event of client.sendMessage(thread.id, 'Check order #123', {
  tools,
  onToolCall: async (call) => {
    const order = await db.orders.find(JSON.parse(call.arguments).orderId);
    return JSON.stringify(order);
  },
})) {
  if (event.type === 'token') process.stdout.write(event.delta);
}
```

### Stopping a Response

```typescript
await client.stopThread(thread.id);
```

### Knowledge (RAG) Documents

The SDK exposes the knowledge endpoints directly — useful for one-off
seed scripts that upload your docs, or admin tooling that lists/deletes
documents.

```typescript
// Upload a document for the agent's RAG store. File | Blob accepted.
const md = await readFile('docs/quickstart.md', 'utf8');
const blob = new Blob([md], { type: 'text/markdown' });
const doc = await client.uploadDocument(agentId, blob, 'Quickstart');

// List existing documents on the agent
const docs = await client.listDocuments(agentId);

// Delete a document
await client.deleteDocument(doc.id);
```

**Important — built-in knowledge retrieval is silent.** When the agent
processes a turn, the engine pre-fetches the top-N matching chunks via
pgvector and injects them into the prompt as context. **No `tool_call`
SSE event fires for the built-in RAG path.** If your UI is listening
for `tool_call` events to render a "Searched: …" badge, the badge will
never fire from built-in knowledge — it only fires for *configured*
tools (custom webhook tools, MCP, web search, computer use). To
surface knowledge usage today, fetch the underlying chunks separately
(e.g. via your own per-message `/sources` route on the server side).
Pattern: store the message ID from the `done` event, then look up
which chunks the engine retrieved for that turn.

For very large doc sets, prefer to seed with a Node script outside
your Next.js dev server — `npm run seed:knowledge` style.

## React Integration

### Provider + useChat Hook

```tsx
import { FlapjackProvider, useChat } from '@maats/flapjack/react';

// Wrap your app (or chat section) with the provider
function App() {
  return (
    <FlapjackProvider config={{
      apiKey: process.env.NEXT_PUBLIC_FLAPJACK_API_KEY!,
      baseUrl: process.env.NEXT_PUBLIC_FLAPJACK_BASE_URL ?? 'https://api.flapjack.dev',
    }}>
      <ChatWidget agentId="your-agent-uuid" />
    </FlapjackProvider>
  );
}

// Use the hook in any child component
function ChatWidget({ agentId }: { agentId: string }) {
  const { messages, sendMessage, addSystemMessage, isStreaming, stop, threadId } = useChat(agentId);

  const handleSubmit = (text: string) => {
    sendMessage(text);
  };

  return (
    <div>
      {messages.map((msg, i) => (
        <div key={i} className={
          msg.systemMessage ? 'system-activity' :
          msg.role === 'user' ? 'user-msg' : 'assistant-msg'
        }>
          {msg.systemMessage
            ? `${msg.systemMessage.icon} ${msg.systemMessage.label}`
            : msg.content}
        </div>
      ))}
      {isStreaming && <div>Thinking...</div>}
      <input
        onKeyDown={(e) => {
          if (e.key === 'Enter') {
            handleSubmit(e.currentTarget.value);
            e.currentTarget.value = '';
          }
        }}
        disabled={isStreaming}
        placeholder="Type a message..."
      />
      {isStreaming && <button onClick={stop}>Stop</button>}
    </div>
  );
}
```

### useChat Options

```typescript
const { messages, sendMessage, isStreaming, stop, threadId } = useChat(agentId, {
  tools,                   // Custom tool definitions
  onToolCall,              // Handler for client-side tool calls
  onPlanActivity,          // Called on plan/todo operations
  resourceBindings,        // Attach resource context
  modelOverride,           // Override agent's default model
  webOverrides,            // Override web tool settings
  memoryOverrides,         // Override memory settings
  planOverrides,           // Override plan settings
  computerOverrides,       // Override computer settings
});
```

### usePlan Hook

```tsx
import { usePlan } from '@maats/flapjack/react';

function PlanView({ threadId }: { threadId: string | null }) {
  const { plan, planOpen, setPlanOpen, fetchPlan } = usePlan(threadId);
  if (!plan) return null;
  return (
    <div>
      <h3>{plan.title}</h3>
      {plan.todos?.map((todo) => (
        <div key={todo.id}>{todo.status === 'done' ? '✓' : '○'} {todo.content}</div>
      ))}
    </div>
  );
}
```

### Pre-Built Components

```tsx
import { ChatPanel, FloatingChat, PlanPanel } from '@maats/flapjack/components';
```

Available: `ChatPanel`, `ChatMessages`, `ChatMessage`, `ChatInput`, `ChatEmpty`, `ChatLoading`, `ChatToolCall`, `FloatingChat`, `PlanPanel`.

### useFlapjack: drop down to the underlying client

`useChat` owns the message-list state — `messages` is a flat list of
role + content pairs and you can't change its shape. When you need
assistant turns to carry extra fields (sources, citations, embedded
actions, per-turn metadata, anything you'd render in a side panel) or
want to render tool activity from custom tools / MCP / computer-use as
inline badges, drop down to the underlying client and drive the SSE
iterator yourself.

```tsx
import { useFlapjack } from '@maats/flapjack/react';

type ChatMessage =
  | { id: string; role: 'user'; content: string }
  // your own assistant shape — extend as needed
  | { id: string; role: 'assistant'; content: string; sources?: Source[]; streaming: boolean };

function CustomChat({ agentId }: { agentId: string }) {
  const { client } = useFlapjack();
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const threadIdRef = useRef<string | null>(null);

  async function send(text: string) {
    const userId = crypto.randomUUID();
    const assistantId = crypto.randomUUID();
    setMessages((m) => [
      ...m,
      { id: userId, role: 'user', content: text },
      { id: assistantId, role: 'assistant', content: '', streaming: true },
    ]);

    if (!threadIdRef.current) {
      const thread = await client.createThread({ agentId });
      threadIdRef.current = thread.id;
    }

    for await (const ev of client.sendMessage(threadIdRef.current, text)) {
      setMessages((m) => m.map((msg) => {
        if (msg.id !== assistantId || msg.role !== 'assistant') return msg;
        if (ev.type === 'token') return { ...msg, content: msg.content + ev.delta };
        if (ev.type === 'done')  return { ...msg, content: msg.content || ev.content, streaming: false };
        // ev.type === 'tool_call' / 'tool_result' — fan out into your own UI
        return msg;
      }));
    }
  }
  // …render messages with whatever shape you like
}
```

This is the right reach when `useChat` doesn't expose enough — but
reach for `useChat` first; only drop down when you actually need to
extend the message shape.

### Server-Side Proxy Pattern (Recommended for Production)

Don't ship `fj_live_…` keys to the browser. Instead, mount **path-mirrored**
proxy routes in your app — `/api/threads/…`, `/api/agents/…`, etc., the
same paths Flapjack itself serves — and configure the client SDK with
`baseUrl: ''` so it hits same-origin URLs that you control. The browser
never sees the real key.

The proxy has two shapes: **non-streaming routes** (calls the SDK
server-side and forwards the JSON result), and the **streaming SSE
route** for `POST /api/threads/:id/messages` (pipes upstream bytes
straight through so client aborts propagate to upstream).

```typescript
// app/api/agents/route.ts (Next.js) — non-streaming
import { FlapjackClient } from '@maats/flapjack';

const client = new FlapjackClient({
  apiKey: process.env.FLAPJACK_API_KEY!,
  baseUrl: process.env.FLAPJACK_BASE_URL!,
});

export async function GET() {
  // …add your auth check here — throw 401 if the caller isn't signed in.
  const agents = await client.listAgents();
  return Response.json(agents);
}
```

```typescript
// app/api/threads/[threadId]/messages/route.ts — streaming SSE
//
// Recommended pattern: pipe the upstream body straight through. No
// re-encoding, no iterator, and the upstream fetch is bound to the
// browser's signal so closing the tab aborts the upstream LLM call
// instead of letting it run to completion on Flapjack's bill.
export const runtime = 'nodejs';

const FLAPJACK_BASE_URL = process.env.FLAPJACK_BASE_URL ?? 'https://api.flapjack.dev';
const FLAPJACK_API_KEY = process.env.FLAPJACK_API_KEY!;

export async function POST(
  req: Request,
  { params }: { params: Promise<{ threadId: string }> },
) {
  // …auth check first; 401 if needed.
  const { threadId } = await params;

  const upstream = await fetch(
    `${FLAPJACK_BASE_URL}/api/threads/${encodeURIComponent(threadId)}/messages`,
    {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${FLAPJACK_API_KEY}`,
        'Content-Type': req.headers.get('content-type') ?? 'application/json',
      },
      body: req.body,
      // @ts-expect-error — Node's fetch requires duplex: 'half' for stream bodies.
      duplex: 'half',
      signal: req.signal,
    },
  );

  if (!upstream.ok || !upstream.body) {
    const text = await upstream.text().catch(() => '');
    return new Response(text || JSON.stringify({ error: `HTTP ${upstream.status}` }), {
      status: upstream.status,
      headers: { 'Content-Type': upstream.headers.get('content-type') ?? 'application/json' },
    });
  }

  return new Response(upstream.body, {
    headers: {
      'Content-Type': 'text/event-stream; charset=utf-8',
      'Cache-Control': 'no-cache, no-transform',
      Connection: 'keep-alive',
      'X-Accel-Buffering': 'no',
    },
  });
}
```

**Why pass-through and not the SDK iterator?** An older pattern wraps
`for await (const event of client.sendMessage(...))` in a
`ReadableStream` + `controller.enqueue` and re-encodes each event back
to SSE. It works, but: (1) it decodes and re-encodes every byte for no
reason, (2) it doesn't propagate `req.signal` upstream by default, so
the upstream LLM call keeps running after the browser disconnects, and
(3) it adds another error surface. Pipe the body, forward the signal,
return `upstream.body` — done.

**Pair it with the matching client config.** Because the proxy is
path-mirrored, the client SDK runs unchanged with `baseUrl: ''`:

```tsx
'use client';
import { FlapjackProvider } from '@maats/flapjack/react';

// `apiKey` is required by the SDK but is a sentinel here — the real
// fj_live_… key lives only on the server. The proxy ignores whatever
// the client sends and uses its own.
export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <FlapjackProvider config={{ apiKey: 'proxy', baseUrl: '' }}>
      {children}
    </FlapjackProvider>
  );
}
```

**Path-mirrored vs. prefixed.** Mirroring (`/api/threads/...`) +
`baseUrl: ''` is the simplest setup — the SDK builds URLs as
`/api/threads/:id/...` and they resolve to your routes directly. If you
have to namespace under a prefix (e.g. `/api/flapjack/threads/...`),
that works too — set `baseUrl: '/api/flapjack'`. Stick with one. Path
order matters: nesting non-Flapjack routes under the same prefix can
shadow the SDK's expected paths.

**The full proxy surface for a chat-only integration** is four
non-streaming routes and one streaming route:

| Method | Path | Forwards to | Notes |
|--------|------|-------------|-------|
| `GET`  | `/api/agents`                            | `client.listAgents()` | Used by the SDK to populate agent pickers; skip if you hard-code an agent ID. |
| `POST` | `/api/threads`                           | `client.createThread({ agentId })` | |
| `POST` | `/api/threads/:threadId/messages`        | streaming SSE — see above | The only streaming route. |
| `POST` | `/api/threads/:threadId/stop`            | `client.stopThread(threadId)` | |

Add knowledge / plan / runner routes as needed — same proxy shape, gated
by your auth.

## SSE Event Types

All streaming responses use Server-Sent Events with these event types:

| Event | Fields | Description |
|-------|--------|-------------|
| `meta` | `startedAt: string` | Stream metadata (sent first) |
| `token` | `delta: string` | Incremental text chunk from the LLM |
| `tool_call` | `tool: { id, name, arguments }` | Agent is calling a server-side tool |
| `tool_executing` | `tool_name: string` | Tool execution started |
| `tool_result` | `tool_name, tool_call_id, result` | Tool execution completed |
| `requires_action` | `toolCalls: ToolCall[]` | Agent requests client-side tool execution. Handle via `onToolCall` or `submitToolResults()`. |
| `done` | `ok: boolean, messageId?: string, content: string` | Full final response with persisted message ID |
| `error` | `code: string, detail?: string` | Error occurred |

> **Note:** Built-in knowledge retrieval is **silent** — chunks are
> pre-fetched + injected into the prompt without firing a `tool_call`
> event. `tool_call` only fires for *configured* tools (custom
> webhooks, MCP, web search, computer use). Don't wire a "Searched: …"
> retrieval badge to `tool_call` for built-in knowledge — it won't
> ever fire. See the Knowledge section above for how to surface RAG
> activity.

## API Endpoints Reference

All endpoints accept `Authorization: Bearer fj_live_...` (API key) or Supabase JWT.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/agents` | List agents for your org |
| `POST` | `/api/agents` | Create a new agent |
| `GET` | `/api/agents/:id` | Get agent details |
| `PATCH` | `/api/agents/:id` | Update agent config |
| `DELETE` | `/api/agents/:id` | Delete agent |
| `POST` | `/api/threads` | Create conversation thread (`{ agentId, title?, kind? }`) |
| `POST` | `/api/threads/:id/messages` | Send message, streams SSE response (`{ content, tools?, toolResults? }`) |
| `POST` | `/api/threads/:id/stop` | Stop active streaming response |
| `GET/POST/DELETE` | `/api/threads/:id/participants` | Multiplayer participant management |
| `POST` | `/api/knowledge/upload` | Upload document for RAG |
| `GET` | `/api/knowledge` | List knowledge documents |
| `DELETE` | `/api/knowledge/:id` | Delete knowledge document |
| `GET/POST` | `/api/mcps` | List/add MCP server connections |
| `POST` | `/api/mcps/:id/test` | Test MCP server connection |
| `GET/POST` | `/api/integrations` | List/create database integrations |
| `GET/POST` | `/api/tools` | List/create custom tool definitions |
| `GET/POST` | `/api/plans` | List/create plans (by threadId) |
| `POST` | `/api/plans/:id/todos` | Add todos to a plan |
| `GET` | `/api/analytics` | Usage analytics (period, agentId) |
| `GET/PUT` | `/api/agents/:id/web` | Web tool configuration |
| `GET/PUT` | `/api/agents/:id/memory` | Memory configuration |
| `GET/PUT` | `/api/agents/:id/plan` | Plan configuration |
| `GET/PUT` | `/api/agents/:id/computer` | Computer use configuration |
| `GET/PUT` | `/api/agents/:id/compaction` | Compaction (context management) configuration |
| `GET/PUT` | `/api/agents/:id/credentials` | Credential resolver (BYOK) configuration |
| `GET/PUT` | `/api/agents/:id/multiplayer` | Multiplayer configuration |
| `GET/PUT/DELETE` | `/api/agents/:id/marketplace` | Marketplace profile |
| `GET/POST` | `/api/agents/:id/skills` | List/install skills on an agent |
| `GET/POST` | `/api/runners` | List/create runners (headless AI pipelines) |
| `GET/POST` | `/api/runners/:id/steps` | List/add runner steps |
| `GET/POST` | `/api/runners/:id/triggers` | List/add runner triggers |
| `POST` | `/api/runners/:id/runs` | Trigger a run |
| `GET` | `/api/runners/:id/analytics` | Runner usage + per-model cost breakdown |
| `GET/POST` | `/api/projects` | List/create projects (aesthetic grouping for agents + runners) |
| `GET/PATCH/DELETE` | `/api/projects/:id` | Get/update/delete a project |
| `GET` | `/api/projects/:id/members` | List runners + agents assigned to a project |
| `GET` | `/api/skills` | List installable skills (registry) |
| `GET` | `/api/skills/browse` | Browse public skills marketplace |
| `GET` | `/api/logs` | List trace summaries (agent turns + runner steps) |
| `GET` | `/api/logs/:traceId` | Get full span tree for a trace |
| `POST` | `/api/agents/from-template` | **v0.4.1+** Create agent with persistent Linux computer (one call: agent + config + bootstrap). Idempotent on `externalAppId`. |
| `GET` | `/api/agents/:id/computer/status` | **v0.4.1+** Aggregate computer status (cached 10s): lifecycle, dev server, disk, last test. |
| `GET` | `/api/agents/:id/computer/bootstrap/stream` | **v0.4.1+** Stream the latest bootstrap run's log via **SSE** (`log` / `status` / `done`). |
| `POST` | `/api/agents/:id/computer/exec` | **v0.4.1+** Exec shell command on the agent's computer. Returns **SSE** (`exec_started` / `stdout` / `stderr` / `exit`). Rate-limited 60/min/agent. |
| `GET` | `/api/agents/:id/computer-instances` | List provider records (Heyo VMs etc.) backing the agent's computer. |
| `GET/PUT` | `/api/threads/:id/computer` | Thread-scoped computer state. |

## Persistent Computer (Remote Control) — v0.4.1+

For platforms that want to provision a Flapjack agent per app with a
preloaded Linux computer. One `createAgentFromTemplate` call creates the
agent + boots a persistent Heyo VM with your chosen template + optional
GitHub repo cloned + dependencies installed.

Five templates: `node-playwright`, `python-jupyter`, `nextjs-fullstack`,
`rust-cargo`, `blank`.

> **Naming note (v0.4.1):** the user-facing surface is now "computer"
> throughout — SDK methods, route paths, response shapes, and webhook
> events. The pre-0.4.1 `sandbox` symbols (`getSandboxStatus`,
> `execSandbox`, `dassieAppId`, `sandbox.idled`, etc.) have been removed.

```typescript
import { FlapjackClient, verifyWebhookSignature } from '@maats/flapjack';

const client = new FlapjackClient({ apiKey: process.env.FLAPJACK_API_KEY! });

// Provision on app-create — idempotent on externalAppId
const res = await client.createAgentFromTemplate({
  name: 'Grocery Tracker',
  template: 'nextjs-fullstack',
  repo: { url: 'https://github.com/acme/grocery', installCmd: 'pnpm install' },
  envVars: [{ key: 'NODE_ENV', value: 'development' }],
  webhookUrl: 'https://you.example.com/api/flapjack/webhook',
  externalAppId: app.id,
});
// Store res.agent.id; bootstrap runs in background (~1–3 min).
// res.computer.status is the initial lifecycle status.

// Optional: stream the bootstrap log live (closes on done)
for await (const ev of client.streamBootstrap(res.agent.id)) {
  if (ev.type === 'log')    process.stdout.write(ev.chunk);
  if (ev.type === 'status') console.log('→', ev.status);
  if (ev.type === 'done')   break;
}

// Watch status (cached server-side, safe to poll every 10s)
const status = await client.getComputerStatus(res.agent.id);
// status.computer.status: 'bootstrapping' | 'ready' | 'idle' | 'stopped' | 'destroyed' | 'error'
// status.signals.devServer?.{ port, listening }

// Run commands — async iterator streams stdout/stderr
for await (const ev of client.execComputer(res.agent.id, { command: 'pnpm test' })) {
  if (ev.type === 'stdout') process.stdout.write(ev.chunk);
  if (ev.type === 'exit')   console.log('exit', ev.exitCode);
}

// Verify inbound webhooks (HMAC-SHA256, hex signature header)
const ok = await verifyWebhookSignature(rawBody, sigHeader, process.env.WEBHOOK_SECRET!);
```

**Lifecycle webhooks** (POSTed to `webhookUrl`, signed with `X-Flapjack-Signature`):
`bootstrap.succeeded` · `bootstrap.failed` · `computer.idled` · `computer.destroyed` · `agent.deleted`.

**Cost control:** computers auto-stop after 2h idle; next `exec` resumes them transparently.

**Rate limits:** 60 exec/min per agent, 3 concurrent per agent, 600/min org ceiling. 429s carry `Retry-After`.

**SDK type renames (0.4.0 → 0.4.1):** `SandboxStatus` → `ComputerStatus`, `SandboxLifecycleStatus` → `ComputerLifecycleStatus`, `SandboxExecEvent` → `ComputerExecEvent`, `ExecSandboxOptions` → `ExecComputerOptions`, `SandboxWebhookEvent`/`SandboxWebhookPayload` → `ComputerWebhookEvent`/`ComputerWebhookPayload`.

**When to reach for this:** you're building an app-platform layer where each app needs its own agent + persistent environment to run tests, host a dev server, or perform CI tasks. For in-conversation chat tooling, stick with the normal thread + message path — the agent gets computer tools automatically via its chat-turn config.

## Supported Models

Agents can be configured with these LLM models (set via dashboard or API):

| Model ID | Vendor | Notes |
|----------|--------|-------|
| `gpt-5.4` | OpenAI | Default. Frontier reasoning model |
| `gpt-5.4-mini` | OpenAI | Cheaper, faster |
| `gpt-5.4-nano` | OpenAI | High-volume, lowest cost |
| `claude-opus-4-7` | Anthropic | Deepest reasoning, 1M context |
| `claude-opus-4-6` | Anthropic | Previous-generation Opus, 1M context |
| `claude-sonnet-4-6` | Anthropic | Near-Opus quality, 1M context |
| `claude-haiku-4-5` | Anthropic | Fastest, lowest cost |

## Integration Checklist

When adding Flapjack to an app:

1. **Install:** `npm install @maats/flapjack`
2. **Get API key:** Sign up at flapjack.chat, create org, generate key (`fj_live_...`)
3. **Create agent:** Via dashboard — set name, system prompt, model
4. **Set env vars:** `FLAPJACK_API_KEY` and `FLAPJACK_BASE_URL`
5. **Choose pattern:**
   - Server-side only → Use `FlapjackClient` directly
   - React app → Use `FlapjackProvider` + `useChat` hook
   - React (drop-in) → Use `ChatPanel` or `FloatingChat` from `@maats/flapjack/components`
   - Production React → Server proxy route + client hook
6. **Handle streaming:** Process all SSE event types (especially `token`, `done`, `error`, `requires_action`)
7. **Add error handling:** Network failures, auth errors (401), rate limits

## System User Messages

The `useChat` hook provides `addSystemMessage` for injecting system activity indicators — compact, centered pills (icon + label) that replace standard user bubbles. Used for plan approvals, tool confirmations, MCP events, etc.

```tsx
const { addSystemMessage } = useChat(agentId);

// Sends to agent (default) — agent will respond
addSystemMessage({ icon: 'check', label: 'Plan approved' }, {
  content: 'Approved. Please proceed with the plan.',
});

// Visual-only — no agent turn
addSystemMessage({ icon: 'info', label: 'Memory saved' }, { sendToAgent: false });
```

Built-in icons: `check` (✓), `tool` (🔧), `info` (ℹ), `error` (✕). Pass any emoji or character for custom icons.

Messages with `systemMessage` metadata render via the `SystemUserMessage` component (SDK) or can be detected in custom `renderMessage` callbacks via `msg.systemMessage`.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Exposing `fj_live_` key in client JS | Use server-side proxy route |
| Not handling `error` events in stream | Always add error case in event switch |
| Ignoring `tool_call`/`tool_result` events | Show tool activity in UI for transparency |
| Ignoring `requires_action` events | Use `onToolCall` or `submitToolResults()` to resume stream |
| Hardcoding agent ID | Fetch from `/api/agents` or use env var |
| Not showing streaming state | Use `isStreaming` from `useChat` to disable input |
| Missing `done` event handling | `done` carries the full final text — persist or display it |
| Using wrong base URL | Production: `https://api.flapjack.dev`, local: `http://localhost:3000` |
| Using `sendMessage` for system activity | Use `addSystemMessage` — renders as a pill, not a user bubble |
| Wiring a retrieval badge to `tool_call` for built-in knowledge | Built-in RAG is silent — no `tool_call` event fires. Fetch sources separately keyed by message ID. |
| Re-encoding the streaming proxy with `controller.enqueue` | Pipe upstream `body` through; pass `signal: req.signal` so client aborts cancel the upstream call. |
| Mounting proxy at `/api/flapjack/threads/...` with `baseUrl: ''` | Either path-mirror at `/api/threads/...` (preferred) or use `baseUrl: '/api/flapjack'`. The two have to agree. |
| Shipping `apiKey: process.env.NEXT_PUBLIC_FLAPJACK_API_KEY` in production | That ships your `fj_live_…` to every browser. Use the proxy + `apiKey: 'proxy', baseUrl: ''` pattern instead. |

## Agent Configuration

Agents are configured with these key fields:

```typescript
// API request body (camelCase) — used in POST/PATCH /api/agents
{
  name: string;              // Display name
  description?: string;      // What this agent does
  stablePreamble: string;    // System prompt — defines agent behavior
  defaultModel: string;      // LLM model ID (see supported models)
}

// DB/response fields are snake_case: stable_preamble, default_model
```

The `stable_preamble` is the system prompt that shapes agent behavior. Write it like you'd write any LLM system prompt — clear role, constraints, and personality.

### Anthropic-Specific Features (Claude Models)

When an agent uses a Claude model (`claude-*`), the runtime automatically enables:

- **Prompt caching:** Enabled by default with up to four cache breakpoints per
  request (tools, system, prior compaction summary, and last user message).
  Tool definitions are cached as one prefix so adding/swapping tools mid-thread
  invalidates them — keep tools stable for a thread to maximise hit rate.

  Per-model minimum cacheable prompt size is enforced — `claude-opus-4-7`,
  `claude-opus-4-6`, and `claude-haiku-4-5` need at least **4096 input tokens**;
  `claude-sonnet-4-6` needs **2048**; older models accept down to **1024**.
  Below the threshold the request runs uncached (no error). Check the `done`
  event's `usage.cache_read_tokens` / `usage.cache_write_tokens` — both 0 means
  the prompt was too small.

  Optional 1-hour TTL for low-frequency traffic (cron-driven runners, infrequent
  re-engagement) — opt in with `anthropicOverrides.cache.ttl: '1h'`. Default
  `5m` is best for chat. 5m writes cost 1.25x base; 1h writes cost 2x base;
  reads cost 0.1x base for both.

  Override surface (all optional):

  ```ts
  for await (const event of client.sendMessage(threadId, content, {
    anthropicOverrides: {
      cache: {
        disabled: false,         // turn caching off entirely (debugging)
        ttl: '5m',               // or '1h' for low-frequency traffic
        cacheTools: true,        // disable when tool defs vary per request
      },
    },
  })) {
    // handle event
  }
  ```

- **Context management betas:** Beta headers for extended output and interleaved thinking are sent automatically.

Additional Anthropic features can be enabled per-message via `anthropicOverrides`:

```typescript
// Extended thinking — chain-of-thought reasoning with a token budget
for await (const event of client.sendMessage(thread.id, 'Analyze this complex problem', {
  anthropicOverrides: {
    thinking: { enabled: true, budgetTokens: 10000 },
  },
})) {
  // ... handle events
}

// Model fallback chain — try primary model, fall back on 400/403/404
for await (const event of client.sendMessage(thread.id, 'Hello', {
  modelOverride: 'claude-sonnet-4-6',
  anthropicOverrides: {
    fallbackModels: ['claude-haiku-4-5'],
  },
})) {
  // ... handle events
}
```

`AnthropicOverrides` type:
```typescript
type AnthropicOverrides = {
  thinking?: { enabled: boolean; budgetTokens?: number };  // default budget: 10000
  fallbackModels?: string[];  // tried in order on compatible errors
  cache?: {
    disabled?: boolean;
    ttl?: '5m' | '1h';
    cacheTools?: boolean;
  };
};
```

### OpenAI-Specific Features (GPT Models)

OpenAI's prompt cache is **fully automatic** for prompts ≥1024 tokens on
`gpt-4o` and newer. Cache reads are discounted: gpt-5.4 family caches input
tokens at roughly 0.1x base (a ~90% discount); older models like gpt-4o
cache at 0.5x base (~50%). Check OpenAI's pricing page for the exact rate
for your model. Cache writes are free — there is no separate write tier.

Flapjack passes two routing parameters to maximise hit rate:

- **`prompt_cache_key`** — defaults to the `thread_id`, so requests on the
  same conversation stay on the same backend shard.
- **`safety_identifier`** — defaults to your `org_id`. OpenAI uses it for
  safety reporting and tenant isolation. We pass it on every request so
  all of an org's traffic carries the same identifier.

> SDK options use camelCase (`promptCacheKey`, `safetyIdentifier`); Flapjack
> translates them to OpenAI's snake_case wire fields (`prompt_cache_key`,
> `safety_identifier`) before forwarding the request.

Override surface (all optional):

```ts
for await (const event of client.sendMessage(threadId, content, {
  openaiOverrides: {
    disabled: false,                          // skip both cache hints
    promptCacheKey: 'agent-template:nps',     // share a shard across many threads
    safetyIdentifier: hashedUserId,           // per-end-user routing
  },
})) {
  // handle event
}
```

Use `promptCacheKey` to share a cache shard across many threads of the same
template (e.g. all "support" agent conversations route to the same shard so
the system preamble stays warm). Per-conversation defaults are best for
chat workloads where each thread is its own long-lived context.

## Features Available to Agents

Via the Flapjack dashboard, agents can be configured with:

- **Webhook Tools:** Custom API endpoints the agent can call during conversations
- **Custom Tools (Client-Side):** Tool definitions passed at runtime via SDK; executed in your code via `onToolCall`
- **Database Integrations:** Direct Postgres/Supabase query access for the agent
- **MCP Servers:** Connect to any MCP-compatible tool server (Supabase, GitHub, PostHog, etc.)
- **Knowledge/RAG:** Upload documents for retrieval-augmented generation (pgvector similarity search)
- **Memory:** Store and recall facts/preferences across conversations (agent, thread, or resource scoped)
- **Web Tools:** Search, research, read, and crawl the web (Perplexity + Firecrawl)
- **Plans & Todos:** Structured planning with real-time progress tracking (see below)
- **Computer Use:** Sandboxed code execution. Choose **ephemeral Tensorlake** (fast, runs alongside agent) or **persistent Heyo VM** (Firecracker microVM, reliable persistence). Provider is configurable per-agent; persistent mode is required for the Remote Control / `createAgentFromTemplate` flow.
- **Compaction:** Automatic conversation context management. Anthropic native compaction (Claude models) or OpenAI summary-based fallback keeps long conversations coherent without exceeding context limits. Configure via `/api/agents/:id/compaction`.
- **Multiplayer Chat:** Multi-user conversations with @mention routing
- **Marketplace Profile:** Public agent listing with handle, avatar, capabilities, and readiness score
- **Credential Resolution (BYOK):** Per-user API key resolution via webhook — embedding apps provide their own keys for LLM billing
- **Skills:** Install reusable skill packages on an agent via `/api/agents/:id/skills` (browse the registry at `/api/skills/browse`)
- **Projects:** Aesthetic grouping for runners + agents. `project_id` is nullable on both; deleting a project demotes members rather than cascading.
- **Logs / Tracing:** Span-tree traces for every agent turn and runner step (LLM + tool sub-spans, with rolled-up tokens + cost). Read via `/api/logs` and `/api/logs/:traceId`.
- **Analytics:** Per-agent usage tracking (tokens, cost, tool calls, compaction costs). Runners have their own analytics endpoint with per-model cost breakdown.

## Plans & Todos (Progress Tracking)

When **Plans & Todos** is enabled for an agent (in agent settings), the agent can create structured plans with todo items, and track progress through them in real time.

### How It Works

1. **Plan mode:** User clicks the plan button (ListTodo icon) in `ChatInput` to enable plan mode
2. **Plan creation:** The agent creates a plan with todos via the `plan_create` tool; the plan panel auto-opens
3. **Approval:** User clicks "Approve" in the plan panel (or types feedback to request changes)
4. **Execution with progress:** After approval, the agent works through the plan and updates each todo's status (`in_progress` → `done`) using the `todo_update` tool. The plan panel updates in real time.
5. **Visibility:** The plan remains accessible via the plan button throughout execution and after completion

### SDK Integration

The `usePlan` hook manages plan state. `ChatPanel` handles everything automatically, but you can also use it standalone:

```tsx
import { useChat, usePlan } from '@maats/flapjack/react';

const { messages, sendMessage, isStreaming, threadId } = useChat(agentId);
const { plan, planOpen, setPlanOpen, planMode, setPlanMode, debouncedFetchPlan } = usePlan(threadId);
```

When composing your own UI, pass plan props to `ChatInput`:

```tsx
<ChatInput
  onSend={handleSend}
  isStreaming={isStreaming}
  plan={plan}
  planMode={planMode}
  planOpen={planOpen}
  onPlanToggle={() => plan ? setPlanOpen(p => !p) : setPlanMode(p => !p)}
  onPlanApprove={handleApprove}
/>
```

### Plan Panel

The `PlanPanel` component displays the plan title, a progress counter (done/total), nested todos with status icons (pending, in-progress, done, skipped), and an approve button. It's automatically rendered inside `ChatInput` when `planOpen` is `true` and a `plan` exists.

### SSE Events for Plans

Plan tool calls (`plan_create`, `plan_update`, `todo_add`, `todo_update`) arrive as `tool_result` SSE events. Call `debouncedFetchPlan()` on these events to refresh the plan panel.

## Runners (Headless AI Pipelines)

Runners are headless, schedulable AI pipelines that execute multi-step workflows without a chat UI:

```typescript
// Create a runner
const runner = await client.createRunner({ name: 'Daily Report' });

// Add steps
await client.addRunnerStep(runner.id, { kind: 'agent', name: 'Analyze', agentId: '...' });

// Add a trigger
await client.addRunnerTrigger(runner.id, { kind: 'cron', cronExpression: '0 9 * * MON' });

// Trigger a run manually
const run = await client.triggerRun(runner.id, { input: { topic: 'AI news' } });
```

Step kinds: `agent`, `webhook`, `condition`, `computer`. Trigger kinds: `manual`, `api`, `cron`, `webhook`, `poll`, `bulk_import`, `button`.

Per-runner analytics (success rate, total cost, daily series, per-model
cost breakdown) are exposed at `GET /api/runners/:id/analytics` and via
`client.getRunnerAnalytics(runnerId, { period })`.

These are configured through the dashboard and automatically available to the agent during conversations — no SDK-side configuration needed (except for custom tools, which use the `tools` option).

## Projects (Aesthetic Grouping)

Projects are an organizational container for runners and agents. Membership
is purely cosmetic — `runner-engine` and `runner-config` ignore `project_id`
entirely, so attaching/detaching a project never changes execution behavior.
Deleting a project sets `project_id = null` on its members rather than
cascading deletes.

```typescript
const project = await client.createProject({
  name: 'Customer Onboarding',
  slug: 'customer-onboarding',     // optional, normalized + nullable
  description: 'Agents + runners that handle new-account flows',
});

// Move an agent or runner into the project
await client.updateAgent(agentId, { projectId: project.id });
await client.updateRunner(runnerId, { projectId: project.id });

// Listing membership returns column subsets (not full Runner/Agent rows)
const { runners, agents } = await client.getProjectMembers(project.id);
```

## BYOK Credential Resolution

Agents can be configured with a credential resolver webhook for Bring Your Own Key (BYOK) support. When enabled, Flapjack calls your webhook before each LLM request to resolve per-user API keys, allowing users to pay for their own API usage.

### Setup

```typescript
// Configure a credential resolver webhook for an agent
await client.updateCredentialConfig(agentId, {
  enabled: true,
  resolverUrl: 'https://app.example.com/api/resolve-credentials',
  timeoutMs: 5000, // optional, default 5s
});

// Check current config
const config = await client.getCredentialConfig(agentId);
console.log(config.enabled, config.resolver_url);
```

### Webhook Contract

When a message is sent, Flapjack POSTs to your resolver URL (HMAC-signed with `TOOL_HMAC_SECRET`):

```typescript
// Your webhook receives:
// POST https://app.example.com/api/resolve-credentials
// Headers: X-Flapjack-Timestamp, X-Flapjack-Nonce, X-Flapjack-Signature
// Body:
{
  userId: "user_abc",    // senderId from SDK, or auth userId
  orgId: "org_xyz",
  provider: "anthropic", // or "openai"
  model: "claude-sonnet-4-6"
}

// Your webhook should respond with:
{
  apiKey: "sk-ant-...",  // the user's API key
  source: "byok"         // or "platform" for analytics
}
// Return { apiKey: null } or non-200 to fall back to platform key
```

### Example Webhook (Next.js)

```typescript
// app/api/resolve-credentials/route.ts
export async function POST(req: Request) {
  const { userId, orgId, provider } = await req.json();

  // Look up the user's custom API key from your database
  const user = await db.users.findById(userId);
  const customKey = user?.apiKeys?.[provider];

  if (customKey) {
    return Response.json({ apiKey: decrypt(customKey), source: 'byok' });
  }

  // Fall back to platform key
  return Response.json({ apiKey: null, source: 'platform' });
}
```

### Billing Attribution

Usage events in Flapjack analytics include a `credential_source` field (`'byok'` or `'platform'`) so you can differentiate billing between BYOK users and platform users.
