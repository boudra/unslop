# Density Slop

Code that packs too much logic into too few lines. Dense code is hard to read, hard to debug, and hard to modify. AI agents optimize for "cleverness" and conciseness when they should optimize for clarity.

## Patterns

### 1. Nested Ternaries

```typescript
// Bad: nested ternaries — impossible to read
const label = isConnected
  ? hasError
    ? isRetrying
      ? "Retrying..."
      : "Error"
    : isStreaming
      ? "Streaming"
      : "Connected"
  : isConnecting
    ? "Connecting..."
    : "Disconnected";

// Bad: ternary with function calls
const provider = isPremium
  ? getCustomProvider(user.preferences)
  : isEnterprise
    ? getEnterpriseProvider(org.config)
    : getDefaultProvider();
```

```typescript
// Good: named steps or early returns in a function
function connectionLabel(state: ConnectionState): string {
  if (state.status === "disconnected") return "Disconnected";
  if (state.status === "connecting") return "Connecting...";
  if (state.status === "error") {
    return state.isRetrying ? "Retrying..." : "Error";
  }
  return state.isStreaming ? "Streaming" : "Connected";
}

// Good: compute before use
const provider = resolveProvider(user, org);

function resolveProvider(user: User, org: Org): Provider {
  if (isPremium(user)) return getCustomProvider(user.preferences);
  if (isEnterprise(org)) return getEnterpriseProvider(org.config);
  return getDefaultProvider();
}
```

**Rule:** **Nested ternaries are forbidden.** A single ternary is only acceptable when both branches are a single identifier, primitive, or trivial property access (`x ? a : b`, `on ? "yes" : "no"`). Any ternary whose branches contain another ternary, a function call, a JSX element, or non-trivial logic → extract to a function with `if` / early returns, or a `switch`.

### 2. Dense Object Literals

Object literals with inline logic, function calls, and conditional properties all packed together.

```typescript
// Bad: object literal as a logic dump
const agentPayload = {
  id: generateId(),
  name: config.name || agent.name || "Untitled",
  status: agent.isRunning ? (agent.hasError ? "error" : "running") : "stopped",
  provider: providers.find((p) => p.id === agent.providerId)?.name ?? "unknown",
  uptime: agent.startedAt ? Math.floor((Date.now() - agent.startedAt.getTime()) / 1000) : 0,
  capabilities: [
    ...(agent.canRead ? ["read"] : []),
    ...(agent.canWrite ? ["write"] : []),
    ...(agent.canExecute ? ["execute"] : []),
  ],
  metadata: {
    ...agent.metadata,
    ...(config.extraMetadata ?? {}),
    lastSeen: new Date().toISOString(),
  },
};
```

```typescript
// Good: precompute, then assemble
const status = resolveAgentStatus(agent);
const providerName = providers.find((p) => p.id === agent.providerId)?.name ?? "unknown";
const uptimeSeconds = agent.startedAt
  ? Math.floor((Date.now() - agent.startedAt.getTime()) / 1000)
  : 0;

const agentPayload = {
  id: generateId(),
  name: config.name || agent.name || "Untitled",
  status,
  provider: providerName,
  uptime: uptimeSeconds,
  capabilities: agentCapabilities(agent),
  metadata: {
    ...agent.metadata,
    ...config.extraMetadata,
    lastSeen: new Date().toISOString(),
  },
};
```

**Rule:** Object literals should assemble pre-computed values. If a property value requires branching logic, compute it before the object literal.

### 3. Complex Boolean Expressions

```typescript
// Bad: multi-clause boolean with mixed concerns
if (
  agent.status === "running" &&
  !agent.isTerminating &&
  (agent.provider === "claude" || agent.provider === "codex") &&
  agent.lastHeartbeat &&
  Date.now() - agent.lastHeartbeat.getTime() < 30000 &&
  !pendingCommands.has(agent.id)
) {
  // ...
}
```

```typescript
// Good: named conditions
const isAlive = agent.status === "running" && !agent.isTerminating;
const isSupportedProvider = agent.provider === "claude" || agent.provider === "codex";
const isResponsive = agent.lastHeartbeat != null &&
  Date.now() - agent.lastHeartbeat.getTime() < 30000;
const isIdle = !pendingCommands.has(agent.id);

if (isAlive && isSupportedProvider && isResponsive && isIdle) {
  // ...
}
```

**Rule:** If a boolean expression has more than 2 clauses or mixes different concerns, extract named conditions. Each name should describe what the condition *means*, not what it *checks*.

### 4. Chained Method Calls with Logic

```typescript
// Bad: long chain with inline logic
const result = agents
  .filter((a) => a.status === "running" && a.provider !== "legacy")
  .map((a) => ({
    ...a,
    displayName: a.name || `Agent ${a.id.slice(0, 8)}`,
    isHealthy: a.lastHeartbeat ? Date.now() - a.lastHeartbeat.getTime() < 30000 : false,
  }))
  .sort((a, b) => (a.isHealthy === b.isHealthy ? a.displayName.localeCompare(b.displayName) : a.isHealthy ? -1 : 1))
  .slice(0, limit);
```

```typescript
// Good: break the chain at logic boundaries
const activeAgents = agents.filter(
  (a) => a.status === "running" && a.provider !== "legacy",
);

const withDisplay = activeAgents.map((a) => ({
  ...a,
  displayName: a.name || `Agent ${a.id.slice(0, 8)}`,
  isHealthy: isAgentHealthy(a),
}));

const sorted = withDisplay.sort((a, b) => {
  if (a.isHealthy !== b.isHealthy) return a.isHealthy ? -1 : 1;
  return a.displayName.localeCompare(b.displayName);
});

const result = sorted.slice(0, limit);
```

**Rule:** Method chains are fine for simple transformations. When a chain step contains non-trivial logic (conditional mapping, multi-key sorting, inline computation), break the chain into named intermediate variables.

### 5. Callback Pyramids

```typescript
// Bad: deeply nested callbacks
ws.on("message", (raw) => {
  try {
    const msg = JSON.parse(raw);
    if (msg.type === "subscribe") {
      db.getAgent(msg.agentId, (err, agent) => {
        if (err) {
          ws.send(JSON.stringify({ error: err.message }));
        } else {
          subscriptions.add(msg.agentId, ws, (subErr) => {
            if (subErr) {
              ws.send(JSON.stringify({ error: subErr.message }));
            } else {
              ws.send(JSON.stringify({ ok: true, agent }));
            }
          });
        }
      });
    }
  } catch (e) {
    ws.send(JSON.stringify({ error: "Parse error" }));
  }
});
```

```typescript
// Good: flatten with async/await and early returns
ws.on("message", async (raw) => {
  const parsed = messageSchema.safeParse(JSON.parse(raw));
  if (!parsed.success) {
    ws.send(JSON.stringify({ error: "Invalid message" }));
    return;
  }

  if (parsed.data.type !== "subscribe") return;

  const agent = await db.getAgent(parsed.data.agentId);
  await subscriptions.add(parsed.data.agentId, ws);
  ws.send(JSON.stringify({ ok: true, agent }));
});
```

**Rule:** More than 3 levels of nesting is a smell. Use async/await, early returns, or extract inner logic into named functions.

### 6. Inline Type Computations

```typescript
// Bad: complex type logic inline
function getConfig(): {
  providers: Array<{
    name: string;
    enabled: boolean;
    config: Record<string, string | number | boolean>;
    capabilities: ("read" | "write" | "execute")[];
  }>;
  settings: {
    timeout: number;
    retries: number;
    fallback: { provider: string; delay: number } | null;
  };
} {
  // ...
}
```

```typescript
// Good: named types
interface ProviderEntry {
  name: string;
  enabled: boolean;
  config: Record<string, string | number | boolean>;
  capabilities: ProviderCapability[];
}

interface FallbackConfig {
  provider: string;
  delay: number;
}

interface AppConfig {
  providers: ProviderEntry[];
  settings: {
    timeout: number;
    retries: number;
    fallback: FallbackConfig | null;
  };
}

function getConfig(): AppConfig {
  // ...
}
```

**Rule:** If a type literal is more than 3 properties or has nested objects, promote it to a named type.

### 7. Nested and Fused Operations

The smell isn't chain length — linear pipelines of simple transforms are fine. The smell is operations **wrapped around** or **fused into** each other, forcing the reader to parse inside-out or unpack multiple non-trivial calls from a single expression position. A single clear step is fine even when expensive (`const a = isThing ? doExpensiveThing() : null`).

```typescript
// Bad: outer transform wrapping a chain — read inside-out
return Object.fromEntries(
  [...providers.entries()]
    .filter(([, p]) => p.enabled)
    .map(([name, p]) => [name, createClient(logger, name, p)]),
);

// Bad: call wrapping a spread wrapping a map
const lastActive = Math.max(...events.map(e => e.timestamp));

// Bad: non-trivial calls fused in one position
const connection = fetchClients()[index].connect();
```

```typescript
// Good: unwrap the outer transform, name the parts
const enabled = [...providers.entries()].filter(([, p]) => p.enabled);
const clients: Record<string, Client> = {};
for (const [name, config] of enabled) {
  clients[name] = createClient(logger, name, config);
}

// Good: pull the inner transform out
const timestamps = events.map(e => e.timestamp);
const lastActive = Math.max(...timestamps);

// Good: separate fetch, selection, and the call
const clients = fetchClients();
const client = clients[index];
const connection = client.connect();
```

**Rule:** Look for operations that wrap other operations (`fromEntries(chain)`, `max(...list.map(...))`) or fuse non-trivial calls into a single expression position. Break them apart with named intermediates so the reader sees one step at a time.
