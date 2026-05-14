# Types Slop

Type system abuse and neglect. AI agents frequently work around the type system instead of working with it — casting to `any`, duplicating types that should be inferred, or leaving boundaries untyped.

## Patterns

### 1. `any` Escape Hatches

```typescript
// Bad: any to bypass type errors
const data = response.data as any;
const result = (value as any).toString();
const handler = eventHandlers[type] as any;

// Bad: any in function signatures
function processMessage(msg: any) {
  if (msg.type === "subscribe") { ... }
}

// Bad: any in generics
const cache = new Map<string, any>();
```

```typescript
// Good: proper types
const data: AgentResponse = response.data;

function processMessage(msg: AgentMessage) {
  if (msg.type === "subscribe") { ... }
}

const cache = new Map<string, CachedAgent>();
```

**Rule:** `any` is never the answer for internal code. If you don't know the type, figure it out. If the type is genuinely dynamic, use `unknown` and validate at the boundary.

### 2. Type Assertions Instead of Type Narrowing

```typescript
// Bad: assertion — trusts the developer, not the compiler
const agent = agents.find((a) => a.id === id) as Agent;
agent.start();

// Bad: non-null assertion where result might genuinely be undefined
const config = configMap.get(provider)!;
const port = config.port;
```

```typescript
// Good: narrowing — trusts the compiler
const agent = agents.find((a) => a.id === id);
if (!agent) throw new AgentNotFoundError(id);
agent.start(); // TypeScript knows agent is Agent here

// Good: handle the undefined case
const config = configMap.get(provider);
if (!config) throw new UnsupportedProviderError(provider);
const port = config.port;
```

**Rule:** Type assertions (`as X`, `!`) tell TypeScript to stop checking. Type narrowing (`if`, `switch`, schema validation) tells TypeScript to check harder. Prefer narrowing.

### 3. `@ts-ignore` and `@ts-expect-error`

```typescript
// Bad: silencing legitimate type errors
// @ts-ignore
ws.send(data);

// @ts-expect-error — types are wrong but it works at runtime
const result = provider.execute(command);
```

**Rule:** Fix the type error. If the type definitions are genuinely wrong (e.g., a third-party library), create a proper type declaration file (`.d.ts`) with the correct types. `@ts-ignore` is never acceptable in new code.

### 4. Duplicated Types vs Schema Inference

```typescript
// Bad: hand-written type that duplicates a Zod schema
const agentSchema = z.object({
  id: z.string(),
  name: z.string(),
  status: z.enum(["running", "stopped", "error"]),
  provider: z.string(),
});

// This type will drift from the schema over time
interface Agent {
  id: string;
  name: string;
  status: "running" | "stopped" | "error";
  provider: string;
}
```

```typescript
// Good: infer from the schema
const agentSchema = z.object({
  id: z.string(),
  name: z.string(),
  status: z.enum(["running", "stopped", "error"]),
  provider: z.string(),
});

type Agent = z.infer<typeof agentSchema>;
```

**Rule:** If a Zod schema exists, the TypeScript type MUST be `z.infer<typeof schema>`. Never hand-write a parallel type. This applies to all schema libraries, not just Zod.

### 5. Type Duplication Across Layers

```typescript
// Bad: same concept redefined per layer
// server/types.ts
interface ServerAgent {
  id: string;
  name: string;
  status: AgentStatus;
  pid: number;
}

// api/types.ts
interface ApiAgent {
  id: string;
  name: string;
  status: AgentStatus;
  pid: number;
}

// app/types.ts
interface AppAgent {
  id: string;
  name: string;
  status: AgentStatus;
}
```

```typescript
// Good: one canonical type, layer-specific wrappers only if genuinely different
// shared/types.ts
interface Agent {
  id: string;
  name: string;
  status: AgentStatus;
  pid: number;
}

// app/types.ts — only if the app genuinely doesn't need pid
type AppAgent = Pick<Agent, "id" | "name" | "status">;
```

**Rule:** One canonical type per concept. Use `Pick`, `Omit`, or `Partial` to derive layer-specific views. Do not redefine fields.

### 6. Untyped Boundaries

```typescript
// Bad: network response used without validation
const response = await fetch("/api/agents");
const agents = await response.json(); // agents is `any`
agents.forEach((a) => renderAgent(a)); // no type safety

// Bad: WebSocket messages without schema validation
ws.on("message", (data) => {
  const msg = JSON.parse(data.toString());
  handleCommand(msg.command, msg.args); // completely untyped
});
```

```typescript
// Good: validate at boundary, then use typed data
const response = await fetch("/api/agents");
const raw = await response.json();
const agents = z.array(agentSchema).parse(raw);
agents.forEach((a) => renderAgent(a)); // fully typed

// Good: schema validation on WebSocket messages
ws.on("message", (data) => {
  const parsed = commandSchema.safeParse(JSON.parse(data.toString()));
  if (!parsed.success) return;
  handleCommand(parsed.data.command, parsed.data.args);
});
```

**Rule:** Every boundary where data enters your system (network, file I/O, user input, IPC) must have schema validation. After validation, the types flow automatically.

### 7. Loose String Types

```typescript
// Bad: stringly-typed code
function handleMessage(type: string, payload: Record<string, unknown>) {
  if (type === "subscribe") { ... }
  else if (type === "command") { ... }
  // What other types exist? Who knows.
}

function setStatus(status: string) {
  // Any string is accepted — "runing" (typo) compiles fine
}
```

```typescript
// Good: discriminated unions or string literal types
type MessageType = "subscribe" | "command" | "heartbeat";

function handleMessage(type: MessageType, payload: MessagePayload) {
  // TypeScript enforces valid types, exhaustive switch is possible
}

type AgentStatus = "running" | "stopped" | "error" | "starting";

function setStatus(status: AgentStatus) {
  // Typos are caught at compile time
}
```

**Rule:** If a string can only be one of a known set of values, use a string literal union type. This catches typos at compile time and enables exhaustive checking.

### 8. Complex Inline Object Types

```typescript
// Bad: object shape hidden inside a local declaration
const serviceDeclarations: Array<{ scriptName: string; port?: number }> = [];

// Bad: complex inline parameter type
function startServices(
  declarations: Array<{ scriptName: string; port?: number; env?: Record<string, string> }>,
) {
  // ...
}

// Bad: return type is a structural blob with no domain name
function getWorkspaceScripts(): Array<{
  scriptName: string;
  command: string;
  serviceScript?: boolean;
}> {
  // ...
}
```

```typescript
// Good: name the domain shape once
type ServiceDeclaration = {
  scriptName: string;
  port?: number;
};

const serviceDeclarations: ServiceDeclaration[] = [];

function startServices(declarations: ServiceDeclaration[]) {
  // ...
}
```

**Rule:** Do not put multi-property object shapes inline in variable declarations, function parameters, generic arguments, or return types. Give the shape a named `type` or `interface` close to where the concept belongs.

**Smell:** `Array<{ ... }>` or `Promise<{ ... }>` with more than one field, optional fields, nested objects, unions, or callbacks. Inline structural types hide domain concepts, make diffs noisy, and get copy-pasted instead of reused.

**Allowed:** Tiny one-off mapped data in an expression can be inferred. Prefer no annotation over an inline object annotation when TypeScript can infer it.

### 9. Positional Argument Lists

Positional arguments work when the function name makes the arguments' meaning unambiguous. They fail the moment a call site would need a comment to be readable.

```typescript
// Good: name encodes the single argument's meaning
function getUserById(id: string) { ... }
getUserById("user_42");

function parseDate(raw: string): Date { ... }
function hashPassword(plain: string): string { ... }

// Good: symmetry or a tiny well-known domain — meaning is obvious at the call site
Math.max(a, b);
clamp(value, min, max);
divide(numerator, denominator);
```

```typescript
// Bad: caller can't tell what each argument means
createAgent("claude", "my-agent", true, false, 30000, "full-access");
// Which boolean is which? What does 30000 mean?

// Bad: single positional whose meaning isn't in the name
searchUsers("alice@example.com");
// Searching by email? name? id? full-text query? Unclear from the call.

// Bad: boolean parameter — always unreadable at the call site
startAgent("my-agent", true, false, true);
```

```typescript
// Good: object parameter — call site is self-documenting
createAgent({
  provider: "claude",
  name: "my-agent",
  detached: true,
  quiet: false,
  timeout: 30000,
  mode: "full-access",
});

searchUsers({ email: "alice@example.com" });
startAgent({ name: "my-agent", detached: true, quiet: false, autoRestart: true });
```

**Rules:**

- **One positional** is fine when the function name names it (`getUserById(id)`, `parseDate(raw)`, `hashPassword(plain)`). If the name doesn't tell the caller what the argument is, use an object.
- **Two positionals** are fine when symmetry or a tiny well-known domain makes them obvious (`Math.max(a, b)`, `clamp(value, min, max)`, `divide(numerator, denominator)`). Otherwise, use an object.
- **Three or more parameters** → object. No exceptions.
- **Any boolean parameter** → object. `(thing, true, false, true)` is unreadable at every call site.
- **Any optional parameter** → object. Optionals force callers to write `(value, undefined, true)` to skip past one.

**Why object beats positional past the obvious-name threshold:**

- Self-documenting at the call site — the parameter name is right there.
- Evolves additively — new fields don't break existing calls. Positional lists break the moment you insert a parameter in the middle.
- Argument-order bugs vanish — you can't accidentally swap two strings of the same type.

**Smell:** Function calls with three or more arguments where you can't tell what each one means at a glance. Boolean literals as arguments. `undefined` passed positionally to skip an optional. Single-argument calls where the value's meaning isn't in the function name.
