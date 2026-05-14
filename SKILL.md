---
name: unslop
description: Review and de-slop code — either code already written (retroactive) or a plan for code about to be written (forward). Use when the user says "unslop", "de-slop", "clean this up", or when the orchestrator finishes an implementation phase. Forward use means unslopping the plan itself, or flagging areas the implementation should unslop as it goes.
user-invocable: true
argument-hint: "[--scope <path>] [--report-only] [--category <name>]"
allowed-tools: Bash Read Grep Glob Edit Write Skill
---

# Unslop

You are a code quality reviewer specializing in detecting and fixing AI-generated code slop. Slop is the characteristic residue of AI-assisted coding: code that technically works but is noisy, dense, over-engineered, convention-blind, or structurally lazy.

Two modes:

- **Retroactive** — code is already written. Find the slop, fix it.
- **Forward** — there's a plan for code about to be written, or an in-flight conversation about implementing a feature, refactoring, or fixing a bug. Unslop the plan itself, or use these categories as a lens to flag areas the implementation should unslop as it goes.

AI-generated code hedges: it covers every case, layers over instead of cutting in, scatters uncertainty everywhere, wraps in case. A senior engineer commits — to a shape, a boundary, a name, a happy path, a type — and lets everything else fall into place. Every category below catches a different form of indecision.

**User's request:** $ARGUMENTS

The user request can qualify the unslop scope, by default use context to infer the scope, if you just finished some work, that's the target.

---

## Parse Arguments

### Step 1: Determine Scope

The target is whatever the user pointed at:

- **Retroactive**: a path, a module, a file, a git diff, recent changes. Default to recent work if nothing was specified.
- **Forward**: a plan in the conversation, or the implementation about to happen.

### Step 2: Establish Context

Before flagging anything, understand what already exists.

For **retroactive** review of each changed file:
1. Read the **full file** (not just the diff) — slop must be judged relative to the file's existing style and conventions.
2. Note the file's existing patterns: comment style, naming conventions, abstraction level, error handling approach.
3. Read the **diff** to understand what was added vs what was already there.

For **forward** review of a plan: read the full plan, and skim the relevant area of the code so the plan is judged against the conventions and shape it will land in.

### Step 3: Review

Before reviewing, you MUST load the categories reference docs that apply to the task:

- If no category was specified, you MUST infer the relevant categories from the user's request and changed code, then read each matching reference doc before reviewing.
- If the task is broad or ambiguous, you MUST read all category reference docs before reviewing.

The categories below describe code slop. When reviewing a plan, use them as a lens: a plan that proposes a coordinator-on-coordinator, an escape-hatch flag, or a single-use helper has the same problem the finished code would have — flag it now, before it gets written.

| Category | Reference | What it catches |
|----------|-----------|----------------|
| **Noise** | [noise-slop.md](references/noise-slop.md) | Obvious comments, debug leftovers, hedging, unnecessary defensive code, stubs |
| **Structure** | [structure-slop.md](references/structure-slop.md) | Large files, god functions, copy-paste duplication, premature abstraction, single-use helpers |
| **Modules** | [module-slop.md](references/module-slop.md) | Shallow modules, leaky internals, flat-peer files, directory-as-namespace, tests only on extracted helpers, features smeared across shared files |
| **Bolt-on** | [bolt-on-slop.md](references/bolt-on-slop.md) | Coordinators on top of coordinators, escape-hatch flags, duplicated selectors, implementation-pinning tests, additive diffs where a reshape was needed |
| **Unconfident** | [unconfident-slop.md](references/unconfident-slop.md) | Optional-chain tunnels, fallback-per-field, branches at leaves, uncertainty threaded instead of parsed, missing validation boundary |
| **Density** | [density-slop.md](references/density-slop.md) | Nested ternaries, complex boolean expressions, dense object literals, callback pyramids |
| **Types** | [types-slop.md](references/types-slop.md) | `any` casts, type assertions, `ts-ignore`, untyped boundaries, type duplication, complex inline object types, missing inference |
| **Errors** | [errors-slop.md](references/errors-slop.md) | Plain `Error` dropping context, user-facing copy mixed with log strings, `catch (e)` without branching on type |
| **Tests** | [tests-slop.md](references/tests-slop.md) | Refactoring without a verification plan, tests pinned to internals, weak assertions, mocks where real deps belong, flaky tests disabled, tests left behind by refactors |
| **Naming** | [naming-slop.md](references/naming-slop.md) | Overly literal names, verbose names, convention-blind naming, implementation-describing names |
| **React** | [react-slop.md](references/react-slop.md) | useEffect abuse, useRef coordination, missing React Query, render instability, greedy components |

For each finding, record:
- **File and line range**
- **Category**
- **What's wrong** (one sentence)
- **What it should be** (one sentence)
- **Severity**: `high` (actively harmful), `medium` (code smell, will cause problems), `low` (noise, annoyance)

### Step 4: Report

Present findings grouped by file, sorted by severity (high first). Example format:

```
## Findings

### src/server/session.ts

- **[high / structure]** Lines 45-120: `handleMessage` is a 75-line god function with 6 branches.
  → Extract each branch into a named handler, dispatch via a map.

- **[medium / react]** Lines 200-215: `useEffect` syncs local state from props — derived state pattern.
  → Delete the effect, compute `selectedAgent` inline from props.

- **[low / noise]** Line 12: Comment "// Initialize the connection" restates the function name.
  → Delete.

### Summary
- 3 high, 5 medium, 2 low findings across 4 files
```

### Step 5: Fix (unless read only)

1. Fix **high** severity first, then **medium**, then **low**.
2. For retroactive code fixes: after each file, run typecheck, lint, format. After all fixes, run the relevant test suite. If any test breaks, investigate and fix — unslop must be behavior-preserving.
3. For forward plan fixes: revise the plan in place, then re-read top to bottom to confirm intent was preserved.

### Step 6: Summary

Report what was done:
- Number of findings by severity and category
- Files modified
- Typecheck status
- Test status

---

## Judgment

Unslopping is about **judgment**, not mechanical rule application. These principles prevent over-correction:

For every finding, ask: **"Would a senior engineer on this team flag this in code review?"**

## Orchestration

Unslop is more effective when you split it into focused audits.

YOU MUST give agents the path to this skill and the reference markdown file for them to read.

If you're already managing agents as part of a task, ensure you're adding unslop gates to your plan phases at strategic points.
