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

```
SKILL.md             Entry point. The dispatcher loads this.
references/          Category-specific slop patterns the skill reads on demand.
  noise-slop.md
  structure-slop.md
  module-slop.md
  bolt-on-slop.md
  unconfident-slop.md
  density-slop.md
  types-slop.md
  errors-slop.md
  tests-slop.md
  naming-slop.md
  react-slop.md
```

`SKILL.md` is the only file the dispatcher reads. The agent picks the relevant `references/*-slop.md` files based on the diff or plan it's reviewing.

## License

MIT — see [LICENSE](LICENSE).
