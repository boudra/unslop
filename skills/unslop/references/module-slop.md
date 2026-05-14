# Module Slop

AI agents produce flat directories of peer files where every file exports its own surface, instead of modules with a deliberate boundary. Consumers end up reaching into implementation files, and the directory becomes a namespace instead of an encapsulation.

A **deep module** has a small interface hiding a large implementation. The directory is the module; the path is part of the name.

## Vocabulary

Use these terms exactly. They are the language for talking about module shape — don't substitute "component," "service," "API," or "boundary."

- **Module** — anything with an interface and an implementation. Scale-agnostic: applies to a function, class, file, directory, or package.
- **Interface** — everything a caller must know to use the module: type signature, invariants, ordering, error modes, required config, performance characteristics. *Not* just the type-level surface.
- **Implementation** — the code inside.
- **Depth** — leverage at the interface. **Deep** = lots of behavior behind a small interface. **Shallow** = interface nearly as complex as the implementation.
- **Seam** — where an interface lives. The place behavior can be altered without editing in place. (Use this, not "boundary" — "boundary" is overloaded with DDD's bounded context.)
- **Adapter** — a concrete thing satisfying an interface at a seam.
- **Leverage** — what callers get from depth. More capability per unit of interface.
- **Locality** — what maintainers get from depth. Change, bugs, and verification concentrate at one place. Fix once, fixed everywhere.

Two principles that recur across the patterns below:

- **The interface is the test surface.** Callers and tests cross the same seam. If you want to test past the interface, the module is the wrong shape.
- **One adapter means a hypothetical seam. Two adapters means a real one.** Don't introduce a port unless something actually varies across it.

## Patterns

### 1. Shallow Module with Flat Peers

Every file in a directory exports something consumed from outside.

```typescript
// Bad: flat peers, every file is public

// src/provider/provider-registry.ts
export function registerProvider(...) { ... }

// src/provider/provider-config.ts
export function loadProviderConfig(...) { ... }

// src/provider/provider-validator.ts
export function validateProvider(...) { ... }

// src/provider/provider-runner.ts
export function runProvider(...) { ... }

// callers:
import { registerProvider } from "~/provider/provider-registry";
import { loadProviderConfig } from "~/provider/provider-config";
import { validateProvider } from "~/provider/provider-validator";
import { runProvider } from "~/provider/provider-runner";
```

The `provider/` directory is a namespace, not a module. Every internal step is public API. The "shape" of the module is the union of everything inside it.

```typescript
// Good: one entry, internal files hidden

// src/provider/index.ts
export function runProvider(definition: ProviderDefinition) {
  const config = loadConfig(definition);
  validate(config);
  register(config);
  return run(config);
}

// src/provider/config.ts       — not imported from outside
// src/provider/validate.ts     — not imported from outside
// src/provider/registry.ts     — not imported from outside

// callers:
import { runProvider } from "~/provider";
```

**Rule:** A directory should expose one intentional surface. If every file in the directory has external callers, it's not a module.

### 2. Directory Name Repeated in Every Filename

Filenames prefixed with the directory name signal the files are being treated as peers, not as internals of a module.

```
// Bad: file names duplicate the dir, because they need to be
//      globally unique when imported individually
src/thing/
  thing-provider.ts
  thing-config.ts
  thing-runner.ts
  thing-registry.ts

// Good: internal files named for their role inside the module
src/thing/
  index.ts        // public entry
  config.ts
  runner.ts
  registry.ts
```

**Smell:** If you renamed the directory, you'd have to rename every file in it. That's a sign the files are treated as peers in a flat namespace, not as parts of a bounded module.

### 3. Barrel Entry Instead of Designed Entry

Barrel files re-export everything from a directory. Deep modules intentionally hide most things.

```typescript
// Bad: barrel that forwards every internal file
// src/provider/index.ts
export * from "./provider-registry";
export * from "./provider-config";
export * from "./provider-validator";
export * from "./provider-runner";
```

The index has the same shape as the flat-peers version — just with one extra layer of indirection. Consumers can still reach into anything.

```typescript
// Good: index is the interface, not a forwarder
// src/provider/index.ts
export function runProvider(definition: ProviderDefinition) { ... }
export type { ProviderDefinition };
```

**Rule:** The public entry is a designed interface, not a re-export of everything. If you can replace `export function X` with `export * from "./x"` without changing anything, you have a barrel, not a module.

### 4. Testing Internals Instead of the Boundary

**The interface is the test surface.** Callers and tests cross the same seam — if your test reaches past the interface to assert on internals, either the module is the wrong shape or the test is testing the wrong thing. Boundary tests survive refactors and catch the real bugs, which usually live in how pieces compose, not in the pieces themselves.

```typescript
// Bad: every internal piece has its own test, boundary untested
// provider-config.test.ts      — unit tests loadConfig
// provider-validator.test.ts   — unit tests validate
// provider-registry.test.ts    — unit tests register
// (no test for runProvider — the thing that ties it together)
```

```typescript
// Good: tests at the boundary, exercising the composition
// provider.test.ts
test("runProvider registers and runs a valid definition", () => { ... });
test("runProvider rejects invalid config", () => { ... });
```

**Rule:** If the only tests are on extracted helpers and nothing tests the composition, the module is shallow. Tests on internals are a tool, not the default — reach for them when a branch is hard to hit from the boundary.

### 5. Imports Reaching Past the Boundary

Any import from outside a module that names a specific file inside it is a boundary violation.

```typescript
// Bad: reaching into internals from another module
import { validate } from "~/provider/provider-validator";
import { loadConfig } from "~/provider/provider-config";

// Good: go through the boundary
import { runProvider } from "~/provider";
```

If the boundary doesn't expose what the caller needs, extend the boundary deliberately — don't bypass it.

**Smell:** An import path with more than one segment past the module root (`~/provider/provider-config` vs `~/provider`). The deeper the path, the more internals are leaking.

### 6. Boundary Leaks Data Shape

A module's boundary should hide not just its files, but the shape of its data. AI-generated interfaces often return the raw storage — callers then index, filter, or `get` on it, coupling themselves to the internal representation.

```typescript
// Bad: callers know clients are stored as an indexable list
function fetchClients(): Client[] { ... }
const connection = fetchClients()[index].connect();

// Bad: callers know sessions are a Map
function getSessions(): Map<string, Session> { ... }
const session = getSessions().get(sessionId);

// Bad: every caller does the same filter
const active = (await getAgents()).filter(a => a.status === "running");
```

```typescript
// Good: boundary exposes the caller's actual question
function fetchClientById(id: string): Client { ... }
const connection = fetchClientById(id).connect();

function getSession(id: string): Session | null { ... }
const session = getSession(sessionId);

function getActiveAgents(): Promise<Agent[]> { ... }
const active = await getActiveAgents();
```

**Rule:** The boundary answers the caller's question, not "here's my storage." If every caller does the same transformation after a function returns (index, filter, `get`), that transformation belongs inside the module.

**Smell:** Callers repeatedly indexing, filtering, or destructuring the result of a module function. Return types like `T[]`, `Map<K, V>`, `Record<K, V>` that callers only reach into to pull out one thing.

### 7. Filename Carrying the Whole Name

When you create a new file, the directory and the filename together form its name. If the filename does all the work, the path is empty — you skipped a module.

```
// Bad: filename is the entire noun phrase; path adds nothing
src/server/agent-status-formatter.ts
src/server/workspace-archive-helpers.ts
src/server/cursor-codec.ts
src/api/user-request-validator.ts

// Good: path carries the domain; filename is the role inside it
src/server/agent/status-formatter.ts
src/server/workspace/archive.ts
src/server/pagination/cursor.ts
src/api/user/validate-request.ts
```

**Test:** Could the filename shrink if the path were one level deeper? If yes, deepen the path.

**Smell:** Filenames ending in implementation-shaped suffixes — `-codec`, `-utils`, `-helpers`, `-manager`, `-handler`, `-controller`, `-formatter`, `-builder`. These appear when the filename has to do double duty (naming both the domain and the role) because there's no directory carrying the domain.

**Rule:** Path is part of the name. Pick the directory first; let the filename shrink to the role it plays inside that directory.

### 8. Dumping New Files at the Nearest Root

When placement isn't obvious, the failure mode is to drop the file at the package root next to whatever's already there. This is how flat-peer directories (§1) grow.

```
// Bad: agent extracted a piece of session pagination, dropped it at server root
src/server/
  session.ts
  cursor-codec.ts          ← lives next to 100 unrelated peers
  workspace-registry.ts
  voice-config.ts
  ...

// Good: extraction homed inside the boundary that uses it
src/server/
  session.ts
  pagination/
    cursor.ts              ← internal to the boundary that needs it
  ...
```

If a file's only consumers are inside one feature subdirectory, its home is probably inside that subdirectory. If it's genuinely shared across unrelated features, it needs a new sub-module — name and create it deliberately.

**Rule:** When you don't know where a new file belongs, **say so in the report**. Don't pick the most convenient directory. The architectural decision is information for a human with codebase context, not something to paper over with placement-by-default.

**Smell:** A new file added to a directory that already has 30+ peers, especially when its callers all live inside one feature subdirectory.

### 9. Pass-Through Modules

A module whose only job is to hand its caller's request to another module. Adds a hop, hides nothing. Failed deletion test.

```typescript
// Bad: AgentService just forwards to AgentManager
class AgentService {
  constructor(private manager: AgentManager) {}
  getAgent(id: string) { return this.manager.getAgent(id); }
  listAgents() { return this.manager.listAgents(); }
  startAgent(id: string) { return this.manager.startAgent(id); }
}
```

```typescript
// Good: callers depend on AgentManager directly. AgentService deleted.
import { agentManager } from "./agent-manager";
agentManager.startAgent(id);
```

**The deletion test.** Imagine deleting the module. If complexity vanishes — callers go straight to what they actually wanted — the module was a pass-through. If complexity reappears across N callers (each redoing the same work the module used to centralize), the module was earning its keep. **A pass-through fails the test; a deep module passes it.**

**Rule:** Every module must concentrate complexity that would otherwise spread. Apply the deletion test before introducing or keeping a module — if removing it makes the codebase simpler, remove it.

**Smell:** A module whose every method is a one-line forward to another module. A class that just holds dependencies and exposes them. Files where the import statements outweigh the body. Names ending in `-service`, `-facade`, `-wrapper` that don't transform the underlying interface.

### 10. Feature Smeared Across Shared Files

A new feature gets interleaved into existing multi-purpose files instead of given its own module. Each shared file gains a "section" for the new concept alongside the others. Passes line-level review; fails structurally.

The deletion test from §9 applies in reverse. §9 catches a *module that shouldn't exist*. This catches a *module that should but doesn't*.

```
// Bad: skills feature spread across the integrations layer,
//      cohabiting every file with cli-install
src/integrations/
  integrations-manager.ts    ← skills section + cli section
  daemon-manager.ts          ← 4 skill IPC handlers wedged in
  desktop-daemon.ts          ← skill wrappers next to cli ones
  use-install-status.ts      ← useSkillsStatus next to useCliInstall
  integrations-section.tsx   ← skills row next to cli row

// Good: skills owns a subdir; other layers just register it
src/integrations/
  skills/
    manifest.ts
    sync.ts
    ipc.ts
    hook.ts
    section.tsx
  integrations-manager.ts    ← imports + registers skills/
```

**The deletion test, for features.** Imagine removing this feature. **Mechanical** — delete the folder, drop a handful of registrations? It had a home. **Forensic** — read N multi-purpose files and surgically excise the feature's *logic* from each? It had no home, and every future change pays the same tax.

**Rule:** A new feature gets a home before it gets implementation. The home is a deep module — the folder is the concept, the entry exposes a small surface, integration sites just wire it in. Wide registration is fine; wide *implementation* is the smell.

**Defensible fail:** sometimes the convention pairs concept A with concept B at every layer (e.g. cli-install + skills). Mirroring an existing shape rather than splitting one without splitting the other is a passing answer — write the reason down. "We didn't think about it" is the slop, not the choice itself.

**Smell:** a diff that adds parallel logic in 5+ shared files. A new noun appearing as a "section" inside a file that already hosts other nouns.
