# unslop

Slash-command skill for Claude Code and Codex that reviews and de-slops AI-generated code.

Slop is the residue of AI-assisted coding: code that works but is noisy, dense, over-engineered, convention-blind, or structurally lazy. `unslop` catalogs the patterns and tells the agent how to find and fix them.

Two modes:

- **Retroactive.** Code is already written. Find the slop, fix it.
- **Forward.** A plan exists for code about to be written. Unslop the plan first.

Run with `/unslop`.

## Install

```bash
npx skills add boudra/unslop
```

See [vercel-labs/skills](https://github.com/vercel-labs/skills) for flags and supported agents.

## References

`SKILL.md` is the entry point. The agent loads category-specific docs on demand.

| Reference | Catches |
|-----------|---------|
| [noise-slop.md](skills/unslop/references/noise-slop.md) | Obvious comments, debug leftovers, hedging, defensive code without cause. |
| [structure-slop.md](skills/unslop/references/structure-slop.md) | Copy-paste duplication, premature abstraction, god functions and files, wrapper layers. |
| [module-slop.md](skills/unslop/references/module-slop.md) | Flat peer files instead of deep modules, leaky internals, directory-as-namespace. |
| [bolt-on-slop.md](skills/unslop/references/bolt-on-slop.md) | New coordinators, flags, or helpers stacked on top instead of reshaping existing code. |
| [unconfident-slop.md](skills/unslop/references/unconfident-slop.md) | Optional chains, per-field fallbacks, leaf branches compensating for an uncommitted shape. |
| [density-slop.md](skills/unslop/references/density-slop.md) | Nested ternaries, dense object literals, complex booleans. Cleverness over clarity. |
| [types-slop.md](skills/unslop/references/types-slop.md) | `any`, type assertions, `ts-ignore`, untyped boundaries, duplicated types. |
| [errors-slop.md](skills/unslop/references/errors-slop.md) | Plain `Error` instead of typed domain errors, user copy in log strings, blanket `catch (e)`. |
| [tests-slop.md](skills/unslop/references/tests-slop.md) | Tests pinned to internals, weak assertions, mocks where real deps belong. |
| [naming-slop.md](skills/unslop/references/naming-slop.md) | Literal, verbose, convention-blind, implementation-describing names. |
| [react-slop.md](skills/unslop/references/react-slop.md) | `useEffect` for derived state, `useRef` coordination, unstable refs, missing React Query. |

## License

MIT. See [LICENSE](LICENSE).
