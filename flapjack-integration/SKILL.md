---
name: flapjack-integration
version: 1.2.0
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
| Install SDK | `npm install @maats/flapjack` (v0.3.0+) |
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

### Server-Side Proxy Pattern (Recommended for Production)

Instead of exposing API keys client-side, create a proxy route:

```typescript
// app/api/flapjack/agents/route.ts (Next.js)
import { FlapjackClient } from '@maats/flapjack';

const client = new FlapjackClient({
  apiKey: process.env.FLAPJACK_API_KEY!,
  baseUrl: process.env.FLAPJACK_BASE_URL!,
});

export async function GET() {
  const agents = await client.listAgents();
  return Response.json(agents);
}
```

```typescript
// app/api/flapjack/threads/[threadId]/messages/route.ts
import { FlapjackClient } from '@maats/flapjack';

const client = new FlapjackClient({
  apiKey: process.env.FLAPJACK_API_KEY!,
  baseUrl: process.env.FLAPJACK_BASE_URL!,
});

export async function POST(req: Request, { params }: { params: Promise<{ threadId: string }> }) {
  const { threadId } = await params;
  const { content } = await req.json();

  const stream = new ReadableStream({
    async start(controller) {
      const encoder = new TextEncoder();
      const write = (type: string, data: unknown) =>
        controller.enqueue(encoder.encode(`event: ${type}\ndata: ${JSON.stringify(data)}\n\n`));
      try {
        for await (const event of client.sendMessage(threadId, content)) {
          write(event.type, event);
        }
      } catch (err) {
        write('error', { code: 'STREAM_FAILURE', detail: err instanceof Error ? err.message : 'UNKNOWN' });
      } finally {
        controller.close();
      }
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream; charset=utf-8',
      'Cache-Control': 'no-cache, no-transform',
      Connection: 'keep-alive',
    },
  });
}
```

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
| `GET/PUT` | `/api/agents/:id/credentials` | Credential resolver (BYOK) configuration |
| `GET/PUT` | `/api/agents/:id/multiplayer` | Multiplayer configuration |
| `GET/PUT/DELETE` | `/api/agents/:id/marketplace` | Marketplace profile |
| `GET/POST` | `/api/runners` | List/create runners (headless AI pipelines) |
| `GET/POST` | `/api/runners/:id/steps` | List/add runner steps |
| `GET/POST` | `/api/runners/:id/triggers` | List/add runner triggers |
| `POST` | `/api/runners/:id/runs` | Trigger a run |
| `POST` | `/api/agents/from-template` | **v0.4** Create agent with persistent Linux sandbox (one call: agent + config + bootstrap). Idempotent on `dassieAppId`. |
| `GET` | `/api/agents/:id/sandbox/status` | **v0.4** Aggregate sandbox status (cached 10s): lifecycle, dev server, disk, last test. |
| `POST` | `/api/agents/:id/sandbox/exec` | **v0.4** Exec shell command on sandbox. Returns **SSE** (`exec_started` / `stdout` / `stderr` / `exit`). Rate-limited 60/min/agent. |

## Persistent Computer (Remote Control) — v0.4+

For platforms that want to provision a Flapjack agent per app with a
preloaded Linux sandbox (Dassie-style integration). One `createAgentFromTemplate`
call creates the agent + boots a persistent Heyo VM with your chosen
template + optional GitHub repo cloned + dependencies installed.

Five templates: `node-playwright`, `python-jupyter`, `nextjs-fullstack`,
`rust-cargo`, `blank`.

```typescript
import { FlapjackClient, verifyWebhookSignature } from '@maats/flapjack';

const client = new FlapjackClient({ apiKey: process.env.FLAPJACK_API_KEY! });

// Provision on app-create — idempotent on dassieAppId
const res = await client.createAgentFromTemplate({
  name: 'Grocery Tracker',
  template: 'nextjs-fullstack',
  repo: { url: 'https://github.com/acme/grocery', installCmd: 'pnpm install' },
  envVars: [{ key: 'NODE_ENV', value: 'development' }],
  webhookUrl: 'https://you.example.com/api/flapjack/webhook',
  dassieAppId: app.id,
});
// Store res.agent.id; bootstrap runs in background (~1–3 min).

// Watch status (cached server-side, safe to poll every 10s)
const status = await client.getSandboxStatus(res.agent.id);
// status.sandbox.status: 'bootstrapping' | 'ready' | 'idle' | 'stopped' | 'destroyed' | 'error'
// status.signals.devServer?.{ port, listening }

// Run commands — async iterator streams stdout/stderr
for await (const ev of client.execSandbox(res.agent.id, { command: 'pnpm test' })) {
  if (ev.type === 'stdout') process.stdout.write(ev.chunk);
  if (ev.type === 'exit')   console.log('exit', ev.exitCode);
}

// Verify inbound webhooks (HMAC-SHA256, hex signature header)
const ok = await verifyWebhookSignature(rawBody, sigHeader, process.env.WEBHOOK_SECRET!);
```

**Lifecycle webhooks** (POSTed to `webhookUrl`, signed with `X-Flapjack-Signature`):
`bootstrap.succeeded` · `bootstrap.failed` · `sandbox.idled` · `sandbox.destroyed` · `agent.deleted`.

**Cost control:** sandboxes auto-stop after 2h idle; next `exec` resumes them transparently.

**Rate limits:** 60 exec/min per agent, 3 concurrent per agent, 600/min org ceiling. 429s carry `Retry-After`.

**When to reach for this:** you're building an app-platform layer (like Dassie) where each app needs its own agent + persistent environment to run tests, host a dev server, or perform CI tasks. For in-conversation chat tooling, stick with the normal thread + message path — the agent gets computer tools automatically via its chat-turn config.

## Supported Models

Agents can be configured with these LLM models (set via dashboard or API):

| Model ID | Vendor | Notes |
|----------|--------|-------|
| `gpt-5.4` | OpenAI | Default. Frontier reasoning model |
| `gpt-5.4-mini` | OpenAI | Cheaper, faster |
| `gpt-5.4-nano` | OpenAI | High-volume, lowest cost |
| `claude-opus-4-6` | Anthropic | Deepest reasoning, 1M context |
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

- **Prompt caching:** `cache_control: { type: 'ephemeral' }` on the system prompt and the last user message. This saves ~90% on input tokens for multi-turn conversations with large system prompts. No configuration needed — it's on by default.
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
};
```

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
- **Computer Use:** Sandboxed code execution — choose Tensorlake (fast, runs alongside agent) or Vercel Sandbox (Firecracker microVMs, reliable persistence). Configure provider per-agent.
- **Multiplayer Chat:** Multi-user conversations with @mention routing
- **Marketplace Profile:** Public agent listing with handle, avatar, capabilities, and readiness score
- **Credential Resolution (BYOK):** Per-user API key resolution via webhook — embedding apps provide their own keys for LLM billing
- **Analytics:** Per-agent usage tracking (tokens, cost, tool calls)

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

These are configured through the dashboard and automatically available to the agent during conversations — no SDK-side configuration needed (except for custom tools, which use the `tools` option).

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
