# Structure Slop

Structural problems that emerge when AI generates code without understanding the project's architecture. This covers both extremes: too much repetition (copy-paste) and too much abstraction (DRY obsession).

## Patterns

### 1. Copy-Paste Duplication

The same logic implemented multiple times with minor variations.

```typescript
// Bad: duplicated agent status logic across three files

// in agent-list.tsx
function getStatusColor(agent: Agent) {
  if (agent.status === "running") return "green";
  if (agent.status === "stopped") return "gray";
  if (agent.status === "error") return "red";
  if (agent.status === "starting") return "yellow";
  return "gray";
}

// in agent-detail.tsx
function statusToColor(status: string) {
  switch (status) {
    case "running": return "#00ff00";
    case "stopped": return "#888888";
    case "error": return "#ff0000";
    case "starting": return "#ffff00";
    default: return "#888888";
  }
}

// in agent-card.tsx
const STATUS_COLORS: Record<string, string> = {
  running: "green",
  stopped: "gray",
  error: "red",
  starting: "yellow",
};
```

```typescript
// Good: one source of truth
// in agent-status.ts
const STATUS_COLOR = {
  running: "green",
  stopped: "gray",
  error: "red",
  starting: "yellow",
} as const satisfies Record<AgentStatus, string>;

function statusColor(status: AgentStatus): string {
  return STATUS_COLOR[status];
}
```

**Smell:** The same discriminator (`status`, `provider`, `type`) is checked in more than two places. Centralize.

### 2. Premature Abstraction

The opposite of duplication — extracting a "reusable" abstraction for code that's used exactly once.

```typescript
// Bad: single-use helper that adds indirection
function createAgentStatusChecker(agent: Agent) {
  return {
    isRunning: () => agent.status === "running",
    isStopped: () => agent.status === "stopped",
    isError: () => agent.status === "error",
  };
}

// Used exactly once:
const checker = createAgentStatusChecker(agent);
if (checker.isRunning()) { ... }

// Bad: single-use wrapper around a standard API
function fetchWithTimeout(url: string, timeout: number) {
  return fetch(url, { signal: AbortSignal.timeout(timeout) });
}

// Called once in the entire codebase
const response = await fetchWithTimeout(apiUrl, 5000);

// Bad: factory function for a simple object
function createNotificationPayload(title: string, body: string) {
  return { title, body };
}

// Bad: trivial branch extracted into a helper
function workspaceScriptType(serviceScript: boolean): "script" | "service" {
  if (serviceScript) {
    return "service";
  }

  return "script";
}

// Used once, immediately after computing the input:
const serviceScript = isServiceScript(config);
const scriptType = workspaceScriptType(serviceScript);
```

```typescript
// Good: inline the single-use code
if (agent.status === "running") { ... }

const response = await fetch(apiUrl, { signal: AbortSignal.timeout(5000) });

const payload = { title, body };

const scriptType = serviceScript ? "service" : "script";
```

**Rule:** Don't extract a function until it's called from at least two unrelated call sites. A function that's called once is indirection, not abstraction.

**Smell:** A helper whose body is just an `if`/ternary/switch around one local boolean or enum, especially when the name only restates the expression (`workspaceScriptType(serviceScript)`). Keep the expression local unless it is reused, hides genuinely messy domain logic, or gives a domain concept a name that multiple call sites depend on.

### 3. God Functions

Functions that do too many things — fetching, validating, transforming, persisting, and notifying all in one.

```typescript
// Bad: 80-line function doing everything
async function handleAgentMessage(ws: WebSocket, raw: string) {
  // Parse the message
  let message;
  try {
    message = JSON.parse(raw);
  } catch {
    ws.send(JSON.stringify({ error: "Invalid JSON" }));
    return;
  }

  // Validate the message
  if (!message.type) {
    ws.send(JSON.stringify({ error: "Missing type" }));
    return;
  }

  // Handle each message type
  if (message.type === "subscribe") {
    // 20 lines of subscription logic
  } else if (message.type === "command") {
    // 25 lines of command logic
    if (message.command === "start") {
      // nested logic
    } else if (message.command === "stop") {
      // nested logic
    }
  } else if (message.type === "heartbeat") {
    // heartbeat logic
  }

  // Update metrics
  // Send acknowledgment
  // Log the interaction
}
```

```typescript
// Good: dispatch to focused handlers
async function handleAgentMessage(ws: WebSocket, raw: string) {
  const parsed = messageSchema.safeParse(JSON.parse(raw));
  if (!parsed.success) {
    ws.send(JSON.stringify({ error: "Invalid message" }));
    return;
  }

  const handlers: Record<MessageType, MessageHandler> = {
    subscribe: handleSubscribe,
    command: handleCommand,
    heartbeat: handleHeartbeat,
  };

  const handler = handlers[parsed.data.type];
  await handler(ws, parsed.data);
}
```

**Rule:** If a function has more than 3 levels of nesting or handles more than one concern, split it. Each function should answer one question.

### 4. God Files

Files that accumulate unrelated functionality because they were the first place the AI found to put code.

```typescript
// Bad: utils.ts with 500 lines of unrelated helpers
// utils.ts
export function formatDate(date: Date): string { ... }
export function parseAgentId(raw: string): string { ... }
export function calculateRetryDelay(attempt: number): number { ... }
export function truncateString(str: string, len: number): string { ... }
export function isValidWebSocketUrl(url: string): boolean { ... }
export function mergeDeep(target: unknown, source: unknown): unknown { ... }
export function debounce<T extends (...args: unknown[]) => unknown>(fn: T, ms: number): T { ... }
```

**Smells:**
- A file named `utils.ts`, `helpers.ts`, or `common.ts` with more than 5 exports
- A file over 400 lines
- A file where the top and bottom halves could have different names
- Functions in a file that share no imports or types

**Rule:** Organize by domain, not by technical role. `formatDate` belongs near date-related code, not in a generic utils file.

### 5. Wrapper/Bridge/Adapter Layers

AI loves creating intermediate layers between existing code and new code, rather than modifying the existing code.

```typescript
// Bad: adapter wrapping an existing function with identical signature
function agentServiceAdapter(agentManager: AgentManager) {
  return {
    getAgent: (id: string) => agentManager.getAgent(id),
    listAgents: () => agentManager.listAgents(),
    startAgent: (id: string) => agentManager.startAgent(id),
  };
}

// Bad: bridge class that adds nothing
class WebSocketBridge {
  constructor(private ws: WebSocket) {}

  send(data: unknown) {
    this.ws.send(JSON.stringify(data));
  }

  onMessage(handler: (data: unknown) => void) {
    this.ws.addEventListener("message", (e) => handler(JSON.parse(e.data)));
  }
}
```

```typescript
// Good: use the existing interface directly
// If AgentManager already has getAgent/listAgents/startAgent,
// pass it directly — no adapter needed.

// If WebSocket needs JSON serialization, add it where the messages are sent,
// or create a typed client with domain-specific methods — not a generic bridge.
```

**Rule:** If the adapter is a 1:1 passthrough, delete it and use the original. Adapters earn their existence when they transform interfaces, not when they forward calls.

### 6. Barrel Files (Index Re-exports)

```typescript
// Bad: index.ts that only re-exports
// providers/index.ts
export { ClaudeProvider } from "./claude";
export { CodexProvider } from "./codex";
export { OpenCodeProvider } from "./opencode";
export type { Provider, ProviderConfig } from "./types";
```

**Rule:** Import directly from the source file. Barrel files create circular dependency risks and make it harder to find where things are defined.

### 7. Configuration Objects for Simple Behavior

```typescript
// Bad: configuration-driven architecture for 3 options
interface NotificationConfig {
  type: "success" | "error" | "warning";
  duration: number;
  position: "top" | "bottom";
  dismissible: boolean;
  icon: string;
  animationType: "slide" | "fade";
}

const DEFAULT_CONFIG: NotificationConfig = {
  type: "success",
  duration: 3000,
  position: "top",
  dismissible: true,
  icon: "check",
  animationType: "fade",
};

function showNotification(message: string, config: Partial<NotificationConfig> = {}) {
  const resolved = { ...DEFAULT_CONFIG, ...config };
  // ...
}
```

```typescript
// Good: when there are only 3 variations, just have 3 functions
function showSuccess(message: string) { ... }
function showError(message: string) { ... }
function showWarning(message: string) { ... }
```

**Rule:** If the configuration space is small and stable, prefer explicit variants over configurable abstractions. Configuration earns its keep when the combinations are genuinely open-ended.
