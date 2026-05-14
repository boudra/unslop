# Naming Slop

AI agents produce names that are overly literal, excessively verbose, convention-blind, or describe implementation instead of intent. Good names are semantic — they tell you what something *means*, not how it *works*.

## Patterns

### 1. Overly Literal Names

Names that describe the mechanical implementation rather than the purpose.

```typescript
// Bad: describes what it does mechanically
function handleOnClickButtonSubmit() { ... }
function processArrayOfAgentsAndReturnFilteredList() { ... }
function setIsLoadingToTrue() { ... }
const userDataFromDatabaseResponse = await db.getUser(id);
const filteredArrayOfRunningAgents = agents.filter(a => a.status === "running");

// Also bad: names that encode the type
const agentArray = [agent1, agent2];
const nameString = "my-agent";
const isRunningBoolean = true;
const agentMap = new Map<string, Agent>();
```

```typescript
// Good: describes intent and meaning
function submitForm() { ... }
function activeAgents(agents: Agent[]) { ... }
const user = await db.getUser(id);
const running = agents.filter(a => a.status === "running");

// Good: type is in the type system, not the name
const agents = [agent1, agent2];
const name = "my-agent";
const isRunning = true;
const agentsById = new Map<string, Agent>();
```

**Rule:** A name should answer "what is this?" not "how was this made?" Don't encode the type in the name — that's what the type system is for.

### 2. Excessive Verbosity

Names that are technically accurate but painfully long.

```typescript
// Bad: every word adds length but not meaning
const currentlySelectedAgentInstance = agents.find(a => a.id === selectedId);
function initializeWebSocketConnectionToServer() { ... }
function validateAndTransformIncomingAgentMessage() { ... }
const isCurrentUserAuthenticatedAndAuthorized = checkAuth(user);
const handleAgentStatusChangeNotification = (status: AgentStatus) => { ... };
```

```typescript
// Good: concise but still clear
const selectedAgent = agents.find(a => a.id === selectedId);
function connect() { ... }
function parseMessage(raw: unknown) { ... }
const isAuthorized = checkAuth(user);
const onStatusChange = (status: AgentStatus) => { ... };
```

**Rule:** The right name length is the shortest one that's still unambiguous in context. Inside `AgentManager`, you don't need to say "agent" in every method name — `start()`, `stop()`, `list()` are clear.

### 3. Convention-Blind Naming

Names that ignore the conventions established in the surrounding code.

```typescript
// If the codebase uses:
function getAgent(id: string) { ... }
function getSession(id: string) { ... }
function getProvider(name: string) { ... }

// Bad: breaking convention with a different pattern
function fetchWorkspaceData(id: string) { ... }  // should be getWorkspace
function retrieveLogEntries() { ... }             // should be getLogs
function queryAgentStatus(id: string) { ... }     // should be getAgentStatus

// If the codebase uses camelCase for files:
// agentManager.ts, sessionStore.ts
// Bad: switching convention
// agent_controller.ts, session-handler.ts
```

**Rule:** Match the naming patterns of the surrounding code — not your preferences, not a different project. Convention consistency trumps "better" names.

### 4. Implementation-Describing Names

Names that leak implementation details that callers shouldn't know about.

```typescript
// Bad: callers don't need to know it's Redis
const redisAgentCache = createCache();
function queryPostgresForAgent(id: string) { ... }
function parseJsonWebSocketMessage(raw: string) { ... }

// Bad: callers don't need to know the algorithm
function binarySearchAgentList(agents: Agent[], id: string) { ... }
function sha256HashPassword(password: string) { ... }
```

```typescript
// Good: describe the domain concept, not the implementation
const agentCache = createCache();
function getAgent(id: string) { ... }
function parseMessage(raw: string) { ... }
function findAgent(agents: Agent[], id: string) { ... }
function hashPassword(password: string) { ... }
```

**Rule:** If the implementation changes (Redis → Memcached, PostgreSQL → SQLite), should the name change? If yes, the name is wrong.

### 5. Inconsistent Vocabulary

Using multiple words for the same concept.

```typescript
// Bad: three words for the same concept
function getAgent(id: string) { ... }      // "get"
function fetchProvider(name: string) { ... } // "fetch"
function retrieveSession(id: string) { ... } // "retrieve"

// Bad: mixing synonyms for status
agent.state === "active"
session.status === "running"
provider.condition === "healthy"
```

```typescript
// Good: one word per concept across the codebase
function getAgent(id: string) { ... }
function getProvider(name: string) { ... }
function getSession(id: string) { ... }

// Good: consistent vocabulary
agent.status === "running"
session.status === "active"
provider.status === "healthy"
```

**Rule:** Pick one word for each concept and use it everywhere. "Get" or "fetch" or "retrieve" — pick one. "Status" or "state" — pick one. Document the vocabulary if needed.

### 6. Vague or Generic Names

Names that could mean anything.

```typescript
// Bad: what does "data" contain? What does "handle" do?
const data = await fetchAgents();
function handle(event: Event) { ... }
function process(input: unknown) { ... }
const info = getAgentInfo(id);
const result = doThing(agent);
const temp = calculateSomething();
const manager = new Manager(); // manager of what?

// Bad: single-letter variables outside tiny lambdas
const a = getAgent(id);
const s = a.status;
```

```typescript
// Good: specific and meaningful
const agents = await fetchAgents();
function routeCommand(command: AgentCommand) { ... }
function validateMessage(raw: unknown) { ... }
const agentStatus = getAgentStatus(id);
const uptime = calculateUptime(agent);
```

**Rule:** Every name should tell you what the thing IS, not just that it EXISTS. If you can't think of a better name than `data`, `result`, or `info`, you don't understand the thing well enough yet.

### 7. Boolean Names That Don't Read as Questions

```typescript
// Bad: doesn't read naturally in conditions
const agent = true;        // if (agent) — agent what?
const loading = true;      // ambiguous — is loading, or should load?
const visibility = true;   // is visible, or controls visibility?
const authentication = true; // is authenticated?

// Bad: negative booleans
const isNotConnected = false;
const disableAutoRestart = true;
```

```typescript
// Good: reads as yes/no question
const isRunning = true;     // if (isRunning) — clear
const isLoading = true;     // is loading — clear
const isVisible = true;     // is visible — clear
const isAuthenticated = true;

// Good: positive framing
const isConnected = true;
const autoRestart = false; // or: isAutoRestartEnabled
```

**Rule:** Boolean names should read as yes/no questions: `isX`, `hasX`, `canX`, `shouldX`. Avoid negative booleans (`isNot...`, `disable...`) — they cause double-negation confusion.
