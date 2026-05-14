# unslop

A slash-command skill for Claude Code and Codex that reviews and de-slops AI-generated code.

Slop is the characteristic residue of AI-assisted coding — code that technically works but is noisy, dense, over-engineered, convention-blind, or structurally lazy. `unslop` catalogs the patterns and tells the agent how to find and fix them.

Two modes:

- **Retroactive** — code is already written. Find the slop, fix it.
- **Forward** — there's a plan for code about to be written. Unslop the plan before it ships.

Run it with `/unslop` after a feature, refactor, or any block of AI-generated code.

## Install

Use the [`skills`](https://github.com/vercel-labs/skills) CLI from Vercel Labs. It detects your agent (Claude Code, Codex, OpenCode, Cursor, and many others), symlinks the skill into the right place, and works in either project or global scope.

Global install (available in every project):

```bash
npx skills add boudra/unslop -g
```

Project-local install (only the current repo):

```bash
npx skills add boudra/unslop
```

Pick a specific agent with `-a` (e.g. `-a claude-code`, `-a codex`). Pass `--copy` if you'd rather copy the files than symlink them.

Invoke with `/unslop` after install.

## Layout

`SKILL.md` is the entry point — the dispatcher loads it, and the agent picks the relevant `references/*-slop.md` files based on the diff or plan it's reviewing.

| Reference | What it catches |
|-----------|----------------|
| [noise-slop.md](skills/unslop/references/noise-slop.md) | Obvious comments, debug leftovers, hedging, and unnecessary defensive code — visual noise without meaning. |
| [structure-slop.md](skills/unslop/references/structure-slop.md) | Copy-paste duplication, premature abstraction, god functions and files, and wrapper layers that emerge without architectural intent. |
| [module-slop.md](skills/unslop/references/module-slop.md) | Flat directories of peer files instead of deep modules — leaky internals, directory-as-namespace, features smeared across shared files. |
| [bolt-on-slop.md](skills/unslop/references/bolt-on-slop.md) | New coordinators, flags, or helpers added on top of existing code instead of reshaping what's already there. |
| [unconfident-slop.md](skills/unslop/references/unconfident-slop.md) | Optional chains, per-field fallbacks, and leaf-level branches that compensate for an uncommitted shape instead of validating at the boundary. |
| [density-slop.md](skills/unslop/references/density-slop.md) | Nested ternaries, dense object literals, complex booleans, and other cleverness that trades clarity for concision. |
| [types-slop.md](skills/unslop/references/types-slop.md) | `any` casts, type assertions, `ts-ignore`, untyped boundaries, and duplicated types that should be inferred from a schema. |
| [errors-slop.md](skills/unslop/references/errors-slop.md) | Plain `Error` instead of typed domain errors, user-facing copy mixed with log strings, blanket `catch (e)` without branching on type. |
| [tests-slop.md](skills/unslop/references/tests-slop.md) | Tests pinned to internals, weak assertions, mocks where real deps belong, tests treated as end-of-task chores instead of refactor equipment. |
| [naming-slop.md](skills/unslop/references/naming-slop.md) | Overly literal, verbose, convention-blind, or implementation-describing names instead of names that capture intent. |
| [react-slop.md](skills/unslop/references/react-slop.md) | `useEffect` for derived state, `useRef` coordination, unstable references, missing React Query, and other render-killing patterns. |

## License

MIT — see [LICENSE](LICENSE).
