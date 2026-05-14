# Noise Slop

Surface-level slop that adds visual noise without adding meaning. This is the easiest category to detect and fix — it's also the most common output of AI coding agents.

## Patterns

### 1. Obvious Comments

Comments that restate what the code already says.

```typescript
// Bad: comment restates the code
// Initialize the database connection
const db = initDatabase();

// Set the user's name
user.name = newName;

// Check if the user is authenticated
if (isAuthenticated(user)) {

// Loop through all items
for (const item of items) {

// Return the result
return result;
```

```typescript
// Good: comment explains WHY, not WHAT
// Retry with exponential backoff — the provider rate-limits after 10 req/s
const db = initDatabase({ retries: 3 });

// Must match the format the relay expects (ISO 8601 with timezone)
const timestamp = formatISO(date, { representation: "complete" });
```

**Rule:** Delete any comment where removing it loses zero information. Keep comments that explain _why_, document non-obvious constraints, or reference external requirements.

### 2. Tutorial-Style Explanations

AI often writes comments explaining basic language features or framework APIs, as if teaching.

```typescript
// Bad: explains basic JavaScript/TypeScript
// Use destructuring to extract the name property
const { name } = user;

// Create a new array with the map function
const names = users.map((u) => u.name);

// Use optional chaining to safely access nested properties
const city = user?.address?.city;

// Async function that returns a promise
async function fetchData() {
```

**Rule:** Never explain language features or standard library APIs in comments. The reader knows TypeScript.

### 3. Section Dividers and Decorative Comments

```typescript
// Bad: decorative noise
// ==========================================
// HELPER FUNCTIONS
// ==========================================

// --- Agent Management ---

/* ****************************
 * Connection Handling
 * ****************************/

// ============ Types ============
```

**Rule:** Delete all decorative comments. If you need to organize a file, use actual code structure (separate files, modules, or at minimum blank lines).

### 4. Debug and Logging Leftovers

```typescript
// Bad: debug artifacts left in production code
console.log("DEBUG: reached here");
console.log("user:", JSON.stringify(user, null, 2));
console.warn("TODO: remove this log");
debugger;
// eslint-disable-next-line no-console
console.log(response);
```

**Rule:** Remove all `console.log`/`console.warn` that aren't part of a deliberate logging strategy. Remove `debugger` statements. Remove commented-out console.log calls.

### 5. Hedging Comments

AI-generated comments that express uncertainty or temporariness.

```typescript
// Bad: hedging — the AI wasn't sure
// This should work for most cases
// Hopefully this handles the edge case
// This might need to be updated later
// For now, this is sufficient
// Not sure if this is the best approach
// TODO: consider a better solution
// May need refactoring in the future
```

**Rule:** If you're not sure, investigate and write correct code. If there's a genuine known limitation, document it as a concrete issue — not a vague hedge.

### 6. Unnecessary Defensive Code

Defensive checks for conditions that can't happen given the code's context.

```typescript
// Bad: defensive checks in trusted internal code
function processAgent(agent: Agent): void {
  if (!agent) return; // Agent is required by the type system
  if (typeof agent.id !== "string") return; // id is typed as string
  if (!agent.name || agent.name.length === 0) return; // name is required

  // actual logic...
}

// Bad: try-catch around code that doesn't throw
try {
  const name = user.firstName + " " + user.lastName;
  return name.trim();
} catch (error) {
  return "Unknown";
}

// Bad: null checks after non-nullable access
const agent = agents.get(id)!;
if (agent === null || agent === undefined) {
  throw new Error("Agent not found");
}
```

```typescript
// Good: trust the type system for internal code
function processAgent(agent: Agent): void {
  // actual logic directly — types guarantee shape
}

// Good: defensive code at REAL boundaries
function handleWebSocketMessage(raw: unknown): void {
  const parsed = messageSchema.safeParse(raw);
  if (!parsed.success) {
    log.warn("Invalid message", { error: parsed.error });
    return;
  }
  processMessage(parsed.data); // From here on, types are trusted
}
```

**Rule:** Validate at system boundaries (user input, network, file I/O). Trust the type system for internal code. If a `try-catch` can't explain what it's catching and why, it shouldn't exist.

### 7. Placeholder Stubs

Code that was generated as a skeleton but never filled in.

```typescript
// Bad: stubs that do nothing
function handleError(error: Error): void {
  // TODO: implement error handling
}

function validateInput(input: unknown): boolean {
  return true; // TODO: add validation
}

catch (error) {
  // silently swallow
}

function processWebhook(payload: WebhookPayload): void {
  // Implementation goes here
  throw new Error("Not implemented");
}
```

**Rule:** If the function needs to exist, implement it. If it doesn't need to exist yet, don't create it. Empty catch blocks are always wrong — either handle the error or don't catch it.

### 8. Commented-Out Code

```typescript
// Bad: dead code preserved in comments
function getAgent(id: string) {
  // const cached = cache.get(id);
  // if (cached) return cached;
  const agent = db.getAgent(id);
  // cache.set(id, agent);
  return agent;
}

// Old implementation — keeping for reference
// function legacyGetAgent(id: string) {
//   return agents.find(a => a.id === id);
// }
```

**Rule:** Delete commented-out code. Git preserves history. If you need it back, `git log` it.
