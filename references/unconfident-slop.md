# Unconfident Slop

Confident code has a minimal happy path and a clear set of sad paths. You can read it and see what's supposed to happen and how it fails. Unconfident code spreads fallbacks, optional chains, and leaf-level branches everywhere — compensating for the absence of a committed shape instead of establishing one.

When reading a `?.` or `??`, ask: is this expected, or is it here because the author didn't know what shape to commit to?

## Signs

- Optional-chain tunnels (`data?.user?.profile?.name ?? "Unknown"`) deep in logic that should have validated `data` at entry
- Fallback-per-field — every access followed by `?? default` because the shape was never nailed down
- Branches at the leaves — every leaf re-checks `if (x == null) return` instead of the caller establishing the precondition
- `try-catch` around code that shouldn't throw, "just in case"
- `unknown` / `any` carried deep into the system instead of parsed at the boundary
- Multiple layers each handling the same "might be missing" case

## The Fix

Draw a boundary. Parse, validate, or commit to a shape in one place — then trust it. Push branching to a strategic location, not the leaves. If `name` might be missing, decide where that's resolved (default applied, error thrown, user prompted) once; after that point, `name` is a string.

## Rule

Every `?.` and `??` past the validation boundary is unconfident code. Either the boundary needs to handle it, or the type needs to reflect it honestly. Threading "might be missing" everywhere is a sign the shape wasn't drawn.
