# React Slop

React-specific anti-patterns that AI agents produce with alarming frequency. These cause unnecessary re-renders, stale state, effect cascades, and components that are impossible to reason about.

## Patterns

### 1. useEffect Is a Last Resort

`useEffect` is for synchronizing React with **external systems** — DOM, network, timers, WebSockets, subscriptions. It is **not** a general-purpose "do this when something changes" hook. AI agents reach for it constantly when there's a simpler answer.

Decision tree before writing an effect:

| Situation | Right answer |
|---|---|
| Reacting to a React state/prop change | Compute in render, or `useMemo` if expensive |
| Responding to a user event | Event handler, not effect |
| Resetting child state when a prop changes | `key` prop on the child |
| Calling a callback prop when state changes | Call it from the handler that caused the change |
| Fetching data | React Query |
| Syncing with DOM / network / timer / WS | `useEffect` (correct use) |

The most common React slop: using an effect to sync state that should be computed inline.

```typescript
// Bad: effect syncs derived state
function AgentList({ agents, filter }: Props) {
  const [filteredAgents, setFilteredAgents] = useState<Agent[]>([]);

  useEffect(() => {
    setFilteredAgents(agents.filter((a) => a.status === filter));
  }, [agents, filter]);

  return <List items={filteredAgents} />;
}

// Bad: effect mirrors props into state
function AgentDetail({ agent }: Props) {
  const [name, setName] = useState(agent.name);
  const [status, setStatus] = useState(agent.status);

  useEffect(() => {
    setName(agent.name);
    setStatus(agent.status);
  }, [agent]);

  return <Text>{name} — {status}</Text>;
}
```

```typescript
// Good: compute inline — no state, no effect
function AgentList({ agents, filter }: Props) {
  const filteredAgents = agents.filter((a) => a.status === filter);
  return <List items={filteredAgents} />;
}

// Good: use props directly
function AgentDetail({ agent }: Props) {
  return <Text>{agent.name} — {agent.status}</Text>;
}

// Good: if computation is expensive, useMemo
function AgentList({ agents, filter }: Props) {
  const filteredAgents = useMemo(
    () => agents.filter((a) => a.status === filter),
    [agents, filter],
  );
  return <List items={filteredAgents} />;
}
```

**Rule:** If you can compute a value from existing props/state, do it during rendering. `useEffect` is for synchronizing with **external systems** (network, DOM, timers), not for transforming React state.

### 2. useEffect Chains (Effect Cascades)

Multiple effects that trigger each other in sequence.

```typescript
// Bad: effect cascade — each effect triggers the next
function AgentPanel({ agentId }: Props) {
  const [agent, setAgent] = useState<Agent | null>(null);
  const [sessions, setSessions] = useState<Session[]>([]);
  const [activeSession, setActiveSession] = useState<Session | null>(null);

  useEffect(() => {
    fetchAgent(agentId).then(setAgent);
  }, [agentId]);

  useEffect(() => {
    if (agent) {
      fetchSessions(agent.id).then(setSessions);
    }
  }, [agent]);

  useEffect(() => {
    if (sessions.length > 0) {
      setActiveSession(sessions.find((s) => s.isActive) ?? null);
    }
  }, [sessions]);
}
```

```typescript
// Good: React Query handles the data dependency chain
function AgentPanel({ agentId }: Props) {
  const agent = useQuery({
    queryKey: ["agent", agentId],
    queryFn: () => fetchAgent(agentId),
  });

  const sessions = useQuery({
    queryKey: ["sessions", agentId],
    queryFn: () => fetchSessions(agentId),
    enabled: agent.isSuccess,
  });

  const activeSession = sessions.data?.find((s) => s.isActive) ?? null;
}
```

**Rule:** If you have 2+ effects where one sets state that triggers another, you almost certainly want React Query (for async data) or a reducer (for synchronous state transitions).

### 3. useRef for Coordination State

Using refs as mutable flags to coordinate between effects — a sign that the state model is wrong.

```typescript
// Bad: refs as coordination flags
function ConnectionManager() {
  const isConnecting = useRef(false);
  const hasConnected = useRef(false);
  const retryCount = useRef(0);
  const wasManualDisconnect = useRef(false);

  useEffect(() => {
    if (isConnecting.current) return;
    isConnecting.current = true;

    connect().then(() => {
      hasConnected.current = true;
      isConnecting.current = false;
      retryCount.current = 0;
    }).catch(() => {
      isConnecting.current = false;
      if (!wasManualDisconnect.current) {
        retryCount.current += 1;
        // retry logic...
      }
    });
  }, [/* ... */]);
}
```

```typescript
// Good: explicit state machine — all states visible, transitions defined
type ConnectionState =
  | { status: "idle" }
  | { status: "connecting" }
  | { status: "connected" }
  | { status: "reconnecting"; attempt: number }
  | { status: "disconnected"; reason: "manual" | "error" };

function connectionReducer(
  state: ConnectionState,
  event: ConnectionEvent,
): ConnectionState {
  // Pure, testable transitions — no refs needed
}

function ConnectionManager() {
  const [state, dispatch] = useReducer(connectionReducer, { status: "idle" });
  // React renders on state changes, no ref coordination needed
}
```

**Rule:** `useRef` is for two things only:
1. DOM node references (`<View ref={ref}>`).
2. Non-rendering identities that need to survive renders: timer IDs, `AbortController`, animation frame handles, `setInterval` handles, mutable cache of the *latest* callback for a stable event handler.

If the ref's value affects **what gets rendered next**, it's state — model it explicitly, usually with `useReducer` and a discriminated union.

### 4. Not Using React Query for Server State

Managing server state manually with useState + useEffect + loading/error states.

```typescript
// Bad: manual server state management
function AgentList() {
  const [agents, setAgents] = useState<Agent[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    setIsLoading(true);
    fetchAgents()
      .then((data) => {
        setAgents(data);
        setError(null);
      })
      .catch(setError)
      .finally(() => setIsLoading(false));
  }, []);

  const refresh = () => {
    setIsLoading(true);
    fetchAgents()
      .then(setAgents)
      .catch(setError)
      .finally(() => setIsLoading(false));
  };

  // And somewhere else in the app, the same fetch is duplicated...
}
```

```typescript
// Good: React Query handles caching, deduplication, refetching, loading/error states
function AgentList() {
  const { data: agents = [], isLoading, error } = useQuery({
    queryKey: ["agents"],
    queryFn: fetchAgents,
  });

  // Refresh: invalidateQueries(["agents"]) from anywhere
  // Cached: other components using the same queryKey get the same data
  // Deduped: multiple mounts don't cause multiple fetches
}
```

**Rule:** If data comes from a server or async source, use React Query (or equivalent). Manual useState + useEffect for server data is always worse — it doesn't handle caching, deduplication, background refetching, or stale-while-revalidate.

### 5. Unstable References (Inline Objects, Arrays, Functions)

Every `{}`, `[]`, or `() => ...` written inside a component body creates a new reference on every render. That's fine for most props, but catastrophic in four places:

1. **Props to a memoized child** (`React.memo`, `memo`, `PureComponent`) — the memoization is defeated.
2. **Effect / memo / callback dependency arrays** — the effect runs every render, or the memo never hits.
3. **Store selectors / `useMemo` deps** — recomputation every render.
4. **Event handlers attached to list items** — every row re-renders on every parent render.

```typescript
// Bad: new object reference every render
function AgentDashboard({ agents }: Props) {
  return (
    <AgentList
      agents={agents}
      style={{ flex: 1, padding: 16 }}  // new object every render
      onSelect={(id) => selectAgent(id)} // new function every render
      filters={["running", "stopped"]}   // new array every render
      config={{ showDetails: true, limit: 50 }} // new object every render
    />
  );
}
```

```typescript
// Good: stable references
const listStyle = { flex: 1, padding: 16 } as const;
const defaultFilters = ["running", "stopped"] as const;
const listConfig = { showDetails: true, limit: 50 } as const;

function AgentDashboard({ agents }: Props) {
  const handleSelect = useCallback((id: string) => selectAgent(id), []);

  return (
    <AgentList
      agents={agents}
      style={listStyle}
      onSelect={handleSelect}
      filters={defaultFilters}
      config={listConfig}
    />
  );
}
```

**Rule:** Static objects/arrays live at module scope (`as const`). Derived values use `useMemo`. Handlers that cross `memo` boundaries use `useCallback`. Inline literals are fine only when passed to non-memoized children that don't forward them further.

### 6. Greedy Subscribers — Narrow at the Source

A component should subscribe to exactly the data it renders, and transforms should happen **inside** the selector, not in the component body. AI agents love to pull the whole slice and then `.map`/`.filter` it in render — that subscribes the component to every unrelated change in the slice.

```typescript
// Bad: subscribe to the whole collection, transform in the body
function AgentNames() {
  const agents = useAppStore((s) => s.agents);
  const names = agents.map((a) => a.name); // re-renders when any agent field changes
  return <List items={names} />;
}

// Bad: whole object when you need one field
function AgentStatusBadge({ agent }: { agent: Agent }) {
  return <Badge color={statusColor(agent.status)} />; // re-renders on name/pid/etc.
}

// Bad: selector returns a fresh array every call without an equality check
const names = useAppStore((s) => s.agents.map((a) => a.name));
// zustand's default reference equality => always new array => renders every store update
```

```typescript
// Good: select the narrowest primitive
function AgentCount() {
  const count = useAppStore((s) => s.agents.length);
  return <Text>{count} agents</Text>;
}

// Good: pass primitives, not objects
function AgentStatusBadge({ status }: { status: AgentStatus }) {
  return <Badge color={statusColor(status)} />;
}

// Good: transform in the selector + shallow equality when returning arrays/objects
import { useShallow } from "zustand/shallow";

function AgentNames() {
  const names = useAppStore(useShallow((s) => s.agents.map((a) => a.name)));
  return <List items={names} />;
}

// Good: deep equality for nested structures that can't be flattened
import equal from "fast-deep-equal";

const filters = useAppStore((s) => s.currentFilters, equal);
```

**Rules:**
- Prefer primitives out of selectors. They compare with `===` for free.
- If the selector returns an array/object built from parts of state, wrap it in `useShallow` (shape changes only) or pass `fast-deep-equal` as the equality fn (nested changes).
- Never pull a parent object just to read one field. Destructure at the subscription site, not in the body.
- Component props follow the same rule: pass `status`, not `agent`.

### 7. Missing Error Boundaries and Loading States

```typescript
// Bad: no loading/error handling — crashes or shows nothing
function AgentDetail({ id }: Props) {
  const { data } = useQuery({
    queryKey: ["agent", id],
    queryFn: () => fetchAgent(id),
  });

  return <Text>{data.name}</Text>; // crashes when data is undefined
}
```

```typescript
// Good: handle all states
function AgentDetail({ id }: Props) {
  const { data: agent, isLoading, error } = useQuery({
    queryKey: ["agent", id],
    queryFn: () => fetchAgent(id),
  });

  if (isLoading) return <Loading />;
  if (error) return <ErrorView error={error} />;
  if (!agent) return null;

  return <Text>{agent.name}</Text>;
}
```

**Rule:** Every async data source has three states: loading, error, and success. Handle all three. If a component can't handle loading/error, push the boundary to a parent that can.

### 8. Inline Component Definitions

```typescript
// Bad: component defined inside another component — recreated every render
function AgentList({ agents }: Props) {
  // This creates a new component type every render,
  // unmounting and remounting all instances
  const AgentRow = ({ agent }: { agent: Agent }) => (
    <View>
      <Text>{agent.name}</Text>
    </View>
  );

  return agents.map((a) => <AgentRow key={a.id} agent={a} />);
}
```

```typescript
// Good: component defined at module level
function AgentRow({ agent }: { agent: Agent }) {
  return (
    <View>
      <Text>{agent.name}</Text>
    </View>
  );
}

function AgentList({ agents }: Props) {
  return agents.map((a) => <AgentRow key={a.id} agent={a} />);
}
```

**Rule:** Never define components inside other components. Every render creates a new component *type*, causing React to unmount and remount instead of updating.

### 9. Hoisted State That Forces Sibling Re-renders

State placed higher than it's actually consumed drags every sibling into the render cycle.

```typescript
// Bad: input state lifted into a parent whose other children don't need it
function Dashboard() {
  const [query, setQuery] = useState("");
  return (
    <>
      <AgentList />                                {/* re-renders on every keystroke */}
      <HeavyChart />                               {/* re-renders on every keystroke */}
      <SearchInput value={query} onChange={setQuery} />
    </>
  );
}
```

```typescript
// Good: colocate state with its consumer
function SearchInput() {
  const [query, setQuery] = useState("");
  const results = useSearch(query);
  return <Input value={query} onChange={setQuery} results={results} />;
}

function Dashboard() {
  return (
    <>
      <AgentList />
      <HeavyChart />
      <SearchInput />
    </>
  );
}
```

**Rule:** Put state at the lowest common ancestor of its consumers — not higher "just in case." If only one child uses it, it lives in that child. If you need to lift it later, you will know.

### 10. Mixing List Identity With Item Content

A single selector that returns `[{id, ...content}]` re-renders the list container every time any item's content changes — even when the list's membership and order didn't change.

```typescript
// Bad: one selector couples membership and content
function AgentList() {
  const agents = useAppStore((s) => s.agents);
  return agents.map((a) => <AgentRow key={a.id} agent={a} />);
}
// Edit one agent's status => whole list re-renders, every row re-renders
```

```typescript
// Good: split identity (ids + order) from content (per-row)
function AgentList() {
  const agentIds = useAppStore(useShallow((s) => s.agents.map((a) => a.id)));
  return agentIds.map((id) => <AgentRow key={id} id={id} />);
}

function AgentRow({ id }: { id: string }) {
  const status = useAppStore((s) => s.agentsById[id]?.status);
  const name = useAppStore((s) => s.agentsById[id]?.name);
  return <Badge color={statusColor(status)}>{name}</Badge>;
}
```

**Rule:** The list component subscribes to the list's shape. Each row subscribes to its own content. An edit to one row should not re-render the list or its siblings.

### 11. Deep JSX

JSX nested 4+ levels deep with inline conditionals, inline `.map`, and inline styles. Unreadable and impossible to reuse.

```tsx
// Bad: 5 levels, inline conditional chain, inline styles
return (
  <View style={{ flex: 1, padding: 16 }}>
    <ScrollView contentContainerStyle={{ gap: 8 }}>
      {items.map((i) => (
        <Pressable onPress={() => select(i.id)}>
          <View style={{ padding: 12, borderRadius: 8 }}>
            {i.kind === "agent" ? (
              <AgentCard agent={i} />
            ) : i.kind === "session" ? (
              <SessionCard session={i} />
            ) : (
              <UnknownCard item={i} />
            )}
          </View>
        </Pressable>
      ))}
    </ScrollView>
  </View>
);
```

```tsx
// Good: extract rows and conditional branches, lift styles
const styles = StyleSheet.create({
  root: { flex: 1, padding: 16 },
  list: { gap: 8 },
});

function ItemRow({ item }: { item: Item }) {
  if (item.kind === "agent") return <AgentCard agent={item} />;
  if (item.kind === "session") return <SessionCard session={item} />;
  return <UnknownCard item={item} />;
}

return (
  <View style={styles.root}>
    <ScrollView contentContainerStyle={styles.list}>
      {items.map((i) => (
        <Pressable key={i.id} onPress={() => select(i.id)}>
          <ItemRow item={i} />
        </Pressable>
      ))}
    </ScrollView>
  </View>
);
```

**Rule:** Max ~3 levels of JSX nesting per component. Inline `.map` is fine; inline conditional chains of JSX are not — extract to a row component or a function that returns the right element.

### 12. Array Index as Key

```typescript
// Bad: index key on a reorderable / filterable list
items.map((item, i) => <Row key={i} item={item} />);
// Reorder => React reuses the wrong DOM nodes => inputs keep old values,
// animations jump, component state leaks between rows.
```

```typescript
// Good: stable id from the data
items.map((item) => <Row key={item.id} item={item} />);
```

**Rule:** `key` identifies the logical item, not its slot. Use the data's own id. Index keys are only acceptable for permanently static, append-only lists — and even then, prefer a real id.

### 13. Context as a State Store

React Context rerenders *every* consumer on any change to the value. It's fine for things that rarely change (theme, locale, auth user). It's wrong for high-frequency state (selected item, draft input, live data). For that, use a store (zustand) with narrow selectors.

```typescript
// Bad: putting a mutable, frequently-changing object in context
const AppContext = createContext<AppState>(initial);

function Provider({ children }: { children: ReactNode }) {
  const [state, setState] = useState(initial);
  return <AppContext.Provider value={{ ...state, setState }}>{children}</AppContext.Provider>;
  // Every consumer re-renders on any setState, and the value identity changes every render.
}
```

```typescript
// Good: zustand store, each consumer picks its slice
const useAppStore = create<AppState>((set) => ({ /* ... */ }));

function Foo() {
  const count = useAppStore((s) => s.agents.length);
  // only re-renders when count changes
}
```

**Rule:** Context for stable values (theme, auth, DI). Store with selectors for state that changes.

### 14. Noise Memoization

`useMemo` / `useCallback` on things that don't benefit — primitives, trivially cheap derivations, handlers that aren't passed to memoized children. Pure overhead: an extra allocation for the dependency array plus the comparison, for zero render savings.

```typescript
// Bad: memoizing a primitive
const isActive = useMemo(() => status === "running", [status]);

// Bad: memoizing a handler that goes to a native element
const onPress = useCallback(() => doThing(), []);
return <Pressable onPress={onPress} />;
```

```typescript
// Good: no memoization needed
const isActive = status === "running";
return <Pressable onPress={() => doThing()} />;
```

**Rule:** Only memoize when there's a concrete beneficiary: a memoized child, a dependency array, a genuinely expensive computation. Otherwise the memoization is noise.
