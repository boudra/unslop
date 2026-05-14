# Errors Slop

AI agents reach for plain `Error` and string messages when they should be defining typed domain errors. The result: callers can't distinguish a timeout from a validation failure from an auth error, telemetry has nothing structured to log, and user-facing copy gets concatenated into log strings.

## Patterns

### 1. Plain `Error` Drops Context

```typescript
// Bad: structured information collapsed into a string
throw new Error("RPC failed");
throw new Error(`Provider ${name} not found`);
throw new Error(`Timed out after ${ms}ms calling ${operation}`);

// Caller has nothing to branch on:
try {
  await rpc.call(payload);
} catch (e) {
  // Was it a timeout? Auth? Validation? Network? Read the message.
  if (e.message.includes("timeout")) { ... }
}
```

```typescript
// Good: typed domain errors carry their fields
class TimeoutError extends Error {
  constructor(
    public readonly operation: string,
    public readonly waitedMs: number,
  ) {
    super(`${operation} timed out after ${waitedMs}ms`);
    this.name = "TimeoutError";
  }
}

class UnsupportedProviderError extends Error {
  constructor(public readonly provider: string) {
    super(`Provider ${provider} is not supported`);
    this.name = "UnsupportedProviderError";
  }
}

// Caller branches on type, accesses structured fields:
try {
  await rpc.call(payload);
} catch (error) {
  if (error instanceof TimeoutError) {
    logger.warn("rpc timeout", { op: error.operation, waitedMs: error.waitedMs });
    throw error;
  }
  throw error;
}
```

**Rule:** Throw a typed error class that extends `Error` and carries the fields a caller would want to read. The message is for humans; the fields are for code.

**Smell:** `throw new Error(\`some interpolated string\`)` where the interpolated values are the things the caller actually cares about. `e.message.includes(...)` in catch blocks. `error instanceof Error` as the only discrimination.

### 2. User-Facing Copy Mixed With Internal Strings

User messages, log strings, and remediation hints all collapsed into one `error.message`.

```typescript
// Bad: one string trying to serve three audiences
throw new Error(
  `Failed to invoke tool ${name}. Try refreshing your session. (request_id: ${reqId})`,
);
// User sees: garbage with internal IDs
// Log gets: prose instead of fields
// Remediation hint: buried in the message
```

```typescript
// Good: separate fields for separate audiences
class ClientFacingError extends Error {
  constructor(
    message: string,
    public readonly userFriendlyMessage: string,
    public readonly remediation?: string,
  ) {
    super(message);
    this.name = "ClientFacingError";
  }
}

throw new ClientFacingError(
  `tool.invoke failed for ${name}`,        // log message
  "We couldn't run that tool right now.",  // shown to user
  "Try refreshing the page.",              // remediation hint
);
```

**Rule:** When an error is shown to a user, give it a dedicated `userFriendlyMessage` field. Don't make the same string serve telemetry, debug logs, and the user.

**Smell:** Log messages that look like UI copy. Toast notifications that contain stack-trace-shaped strings. `error.message` rendered directly in JSX.

### 3. Catching Without Branching

`catch (e)` that treats every error the same — usually swallow, log generically, return `null`.

```typescript
// Bad: every failure looks the same
try {
  await fetchAgent(id);
} catch (e) {
  console.error("fetch failed", e);
  return null;
}
// Was it 404? 500? Network? Timeout? Auth? Caller can't tell.
```

```typescript
// Good: branch on the error types you can act on; rethrow the rest
try {
  return await fetchAgent(id);
} catch (error) {
  if (error instanceof AgentNotFoundError) {
    return null;
  }
  if (error instanceof TimeoutError) {
    metrics.increment("agent.fetch.timeout");
    throw error;
  }
  throw error;
}
```

**Rule:** Catch blocks should branch on `instanceof` for the error types they can actually handle. Anything you can't act on, rethrow. Don't collapse distinct failures into a generic "something went wrong."

**Smell:** `catch (e) { return null }`, `catch { return defaultValue }`, single `console.error` calls in catch blocks with no branching, `try/catch` wrapping a whole function with one generic handler.
