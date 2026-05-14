# Tests Slop

Tests are part of the refactor toolkit, not a check at the end. They bound risk so refactors can be bold. AI agents treat tests as a verification step rather than equipment they pick up, sharpen, move, and use throughout the work.

The orienting rule: **tests bound risk; risk bounds leverage.** What's safe to refactor right now is whatever your tests can hold. If they can't hold it, your first move is to extend them — that's part of the refactor, not a prerequisite to it.

## Patterns

### 1. Refactoring Without a Verification Plan

Agent dives into structural changes without checking what tests cover the area. Result: changes either get blocked ("can't verify, won't ship") or shipped on hope ("tests look green, probably fine").

```typescript
// Bad: refactor first, hope tests catch regressions
// 1. Restructure SessionManager
// 2. Run npm test
// 3. Tests pass — ship it
// (Tests didn't actually exercise the path that changed.)
```

```typescript
// Good: lock invariants first, then refactor freely
// 1. Read the area. Identify behavior the refactor must preserve.
// 2. Find existing tests. If they cover the boundary, proceed.
// 3. If they don't, write tests against current behavior — locking what is into a contract.
// 4. Refactor. Tests still pass = behavior preserved.
```

**Rule:** Before refactoring, name the behavior that must stay identical and confirm a test holds it. If no test holds it, write one against the current behavior first. This is reverse-TDD: read the code, infer the invariants, lock them in, *then* change the structure.

**Smell:** Refactor diff that touches 5 files but no test diff. Either tests cover the boundary (good) or the refactor is shipping uncovered (bad — lock first).

### 2. Tests Pinned to Internals

Tests that assert on private state, spy counts, or internal call sequences. They break on every refactor and prove nothing about behavior.

```typescript
// Bad: assertions on internals
expect(manager._sessions.size).toBe(3);
expect(parseMessage).toHaveBeenCalledWith({ type: "command" });
expect(spy).toHaveBeenCalledTimes(2);

// Bad: testing the helper, not the boundary
test("formatStatus returns green for running", () => {
  expect(formatStatus("running")).toBe("green");
});
// (Nothing tests that the agent panel actually renders green.)
```

```typescript
// Good: assert observable behavior
const result = await manager.startSession(config);
expect(result).toEqual({ id: expect.any(String), status: "running" });

const html = render(<AgentPanel agent={runningAgent} />);
expect(html.querySelector(".status")).toHaveStyle({ color: "green" });
```

**Rule:** Test the boundary, not the internals. A refactor that reshapes internals shouldn't change tests. If your tests broke, either you changed the boundary (different kind of refactor) or your tests were the slop. Cross-reference `module-slop.md §4`.

**Smell:** `toHaveBeenCalledWith`, spy counts, assertions on `_private` fields, tests on extracted helpers when the composition has no test.

### 3. Tests That Don't Assert

Tests with conditional logic, weak truthiness checks, or partial assertions that pass even when behavior breaks.

```typescript
// Bad: branches in the test — if logic is wrong, the test still passes
test("handles user flow", async () => {
  const result = await doSomething();
  if (result.type === "success") {
    expect(result.data).toBeTruthy();
  } else {
    expect(result.error).toBeDefined();
  }
});

// Bad: weak assertion that misses the actual behavior
expect(result).toBeTruthy();
expect(agents.length).toBeGreaterThan(0);
```

```typescript
// Good: deterministic, specific, full-shape
test("returns timeout error details when provider times out", async () => {
  const result = await createToolCall(input);
  expect(result).toEqual({
    ok: false,
    error: { code: "PROVIDER_TIMEOUT", waitedMs: 30000 },
  });
});
```

**Rule:** No branching in tests. Strong assertions on full shapes, not fragments. A test that passes for the wrong reason is worse than no test — it lies.

**Smell:** `if` / `switch` inside a test body, `toBeTruthy` / `toBeDefined` / `toBeGreaterThan(0)` as the only assertion, partial `expect.objectContaining` that misses the field that actually matters.

### 4. Mocks Where Real Deps Belong

Reaching for `vi.mock`, `jest.mock`, or hand-rolled fakes when a real test database, real API call, or in-memory adapter would prove the behavior actually works.

```typescript
// Bad: mock hides whether the integration actually works
vi.mock("./email-service");
emailService.send.mockResolvedValue({ success: true });

await handleSignup(user);
expect(emailService.send).toHaveBeenCalledWith(user.email, expect.anything());
```

```typescript
// Good: swappable adapter — real shape, in-memory implementation
interface EmailSender {
  send(to: string, body: string): Promise<void>;
}

function createTestEmailSender() {
  const sent: Array<{ to: string; body: string }> = [];
  return {
    send: async (to: string, body: string) => { sent.push({ to, body }); },
    sent,
  };
}

const emailSender = createTestEmailSender();
await handleSignup(user, { emailSender });
expect(emailSender.sent).toEqual([
  { to: "user@example.com", body: expect.stringContaining("Welcome") },
]);
```

**Rule:** Default to real infrastructure (real DB, real API with sandbox creds, real file system in temp dir) or swappable adapters with in-memory implementations. Mocks are an explicit choice, not a default. If you reach for a mock, ask: "How can I make this code accept the dependency as a parameter?"

**Smell:** `vi.mock` / `jest.mock` calls at the top of test files, `mockResolvedValue` / `mockReturnValue` chains, assertions on what was *called* rather than what *happened*.

### 5. Flaky Tests Removed Instead of Fixed

A test fails sometimes. Agent disables it, marks it `.skip`, or deletes it.

```typescript
// Bad: silencing the signal
test.skip("agent reconnects after network drop", async () => { ... });
// "Flaky in CI, will revisit later."
```

```typescript
// Good: identify the variance source, fix it
test("agent reconnects after network drop", async () => {
  // Variance was: relied on real wall-clock for retry backoff.
  // Fix: inject a clock, advance it deterministically.
  const clock = createTestClock();
  const agent = createAgent({ clock });
  await agent.disconnect();
  clock.advance(1000);
  await waitFor(() => expect(agent.status).toBe("connected"));
});
```

**Rule:** Flaky tests are bugs — find the variance source (time, randomness, network jitter, race, shared state, non-deterministic LLM output, environment drift) and fix it in the test setup, harness, or product code. Never disable, never delete to make CI green.

**Smell:** `test.skip`, `test.todo`, `if (process.env.CI) return`, comments like "flaky, ignore."

### 6. Tests Left Behind by Refactors

When code moves, the agent leaves its tests where they were. Or extracts a module without extracting the tests that prove its behavior. Or splits a file but keeps one giant test file alongside.

```
// Bad: code moved, tests didn't
src/server/
  pagination/
    cursor.ts                    ← extracted module
session.test.ts                  ← still has the cursor tests, mixed with session tests
```

```
// Good: tests move with the code they describe
src/server/
  pagination/
    cursor.ts
    cursor.test.ts               ← extracted alongside
  session.ts
  session.test.ts                ← cursor tests removed
```

**Rule:** Tests are mobile. When you move code, move its tests. When you extract a module, extract the tests that exercise its boundary. When you raise or lower a boundary, raise or lower the tests with it. Tests aren't fixed assets — they belong wherever the behavior they describe now lives.

**Smell:** A test file that's grown to assert behaviors that now live in three different modules. A new module without an adjacent test file. Tests that import from `../../old-location/` after a move.

### 7. Tests Not Collocated With Code

Tests in a separate `__tests__/` or `test/` tree, far from the code they describe.

```
// Bad: tests in a parallel tree
src/server/session.ts
src/server/pagination/cursor.ts
test/server/session.test.ts
test/server/pagination/cursor.test.ts

// Good: collocated
src/server/session.ts
src/server/session.test.ts
src/server/pagination/cursor.ts
src/server/pagination/cursor.test.ts
```

**Rule:** Tests live next to the code they describe — `thing.ts` next to `thing.test.ts`. The mental cost of opening a test should be zero.

**Smell:** A `test/` or `__tests__/` directory mirroring `src/`. Two trees that drift independently. PRs where production and test changes touch unrelated paths.

**Note:** If you want a separate test file for "a different thing inside this file," that's a signal the production file should be split first — then collocate each test alongside its piece.

### 8. Wrong Substitution Strategy for the Dependency

§4 says "default to real over mock." Which *kind* of real depends on what kind of dependency you have. Picking the wrong strategy either makes tests slow and flaky (over-using real infra) or hollow and lying (over-using mocks).

Classify the dependency, then pick:

- **In-process** — pure computation, in-memory state, no I/O. **Test through the interface directly.** No adapter. No substitute. The function is the test surface.

- **Local-substitutable** — has a real test stand-in that runs inside the test process (PGLite for Postgres, an in-memory filesystem, an embedded Redis). **Use the stand-in.** The seam is internal to the module's tests; the production interface stays unchanged.

- **Remote but owned** — your own services across a network (microservices, internal APIs). **Define a port at the seam.** Production wires the HTTP/gRPC/queue adapter; tests wire an in-memory adapter satisfying the same port. Logic lives in one deep module; transport is injected.

- **True external** — third-party services you don't control (Stripe, Twilio, OpenAI). **Mock at the port.** The deepened module takes the dependency as an injected interface; tests provide a fake adapter. Production hits the real API.

```typescript
// Wrong substitution: mocking a Postgres call when PGLite would prove the SQL works
vi.mock("./db");
db.query.mockResolvedValue([{ id: "1" }]);

// Right: PGLite running in the test process — real SQL, deterministic, fast
const db = await createTestDatabase();
await seedAgents(db);
const result = await listActiveAgents(db);
```

```typescript
// Wrong substitution: hitting the real Stripe API on every test run
const charge = await stripe.charges.create({ ... });

// Right: port + adapter, fake adapter for tests
interface PaymentGateway {
  charge(params: ChargeParams): Promise<ChargeResult>;
}

const gateway = createFakeGateway();
await checkout(cart, { gateway });
expect(gateway.charges).toEqual([{ amount: 1999, currency: "usd" }]);
```

**Rule:** The dependency category dictates the substitution. In-process needs none. Local-substitutable uses the stand-in. Remote-but-owned uses ports + adapters. True-external uses a fake at the port. Mocking a category that has a better substitution is the slop.

**Smell:** `vi.mock("./db")` (local-substitutable — use the real test DB). Tests that hit production third-party APIs (true-external — define a port). Tests with no substitution at all that take seconds because they're spinning up real services unnecessarily (in-process — just call the function).
