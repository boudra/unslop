# unslop

A slash-command skill for Claude Code and Codex that reviews and de-slops AI-generated code.

Slop is the characteristic residue of AI-assisted coding — code that technically works but is noisy, dense, over-engineered, convention-blind, or structurally lazy. `unslop` catalogs the patterns and tells the agent how to find and fix them.

Two modes:

- **Retroactive** — code is already written. Find the slop, fix it.
- **Forward** — there's a plan for code about to be written. Unslop the plan before it ships.

Run it with `/unslop` after a feature, refactor, or any block of AI-generated code.

## Install

Clone the repo and symlink it into your harness's skills directory.

### Claude Code

```bash
git clone git@github.com:boudra/unslop.git ~/dev/unslop
ln -s ~/dev/unslop ~/.claude/skills/unslop
```

Verify:

```bash
ls -la ~/.claude/skills/unslop
```

Should resolve to `~/dev/unslop`. Invoke with `/unslop` in Claude Code.

### Codex

Symlink into the Codex skills directory instead:

```bash
ln -s ~/dev/unslop ~/.codex/skills/unslop
```

### Other harnesses

Any harness that loads `SKILL.md` from a directory will work — symlink `~/dev/unslop` wherever the harness reads skills from.

## Layout

`SKILL.md` is the entry point — the dispatcher loads it, and the agent picks the relevant `references/*-slop.md` files based on the diff or plan it's reviewing.

| Reference | What it catches |
|-----------|----------------|
| [noise-slop.md](references/noise-slop.md) | Obvious comments, debug leftovers, hedging, and unnecessary defensive code — visual noise without meaning. |
| [structure-slop.md](references/structure-slop.md) | Copy-paste duplication, premature abstraction, god functions and files, and wrapper layers that emerge without architectural intent. |
| [module-slop.md](references/module-slop.md) | Flat directories of peer files instead of deep modules — leaky internals, directory-as-namespace, features smeared across shared files. |
| [bolt-on-slop.md](references/bolt-on-slop.md) | New coordinators, flags, or helpers added on top of existing code instead of reshaping what's already there. |
| [unconfident-slop.md](references/unconfident-slop.md) | Optional chains, per-field fallbacks, and leaf-level branches that compensate for an uncommitted shape instead of validating at the boundary. |
| [density-slop.md](references/density-slop.md) | Nested ternaries, dense object literals, complex booleans, and other cleverness that trades clarity for concision. |
| [types-slop.md](references/types-slop.md) | `any` casts, type assertions, `ts-ignore`, untyped boundaries, and duplicated types that should be inferred from a schema. |
| [errors-slop.md](references/errors-slop.md) | Plain `Error` instead of typed domain errors, user-facing copy mixed with log strings, blanket `catch (e)` without branching on type. |
| [tests-slop.md](references/tests-slop.md) | Tests pinned to internals, weak assertions, mocks where real deps belong, tests treated as end-of-task chores instead of refactor equipment. |
| [naming-slop.md](references/naming-slop.md) | Overly literal, verbose, convention-blind, or implementation-describing names instead of names that capture intent. |
| [react-slop.md](references/react-slop.md) | `useEffect` for derived state, `useRef` coordination, unstable references, missing React Query, and other render-killing patterns. |

## License

MIT — see [LICENSE](LICENSE).
