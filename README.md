# Bash Guard

Adversarial security review extension for [pi](https://github.com/badlogic/pi-mono).
Intercepts `bash` tool calls and runs parallel security reviews using fast LLM
voters before allowing execution.

This is a **derivative port** of the original Bash Guard extension. See the
upstream source for the canonical implementation and design:

**https://github.com/pascal-de-ladurantaye/pi-agent/blob/main/extensions/bash-guard**

## What changed from upstream

- Ported from `@mariozechner/*` packages to `@earendil-works/*` packages.
- Source file renamed `index.ts` → `bash-guard.ts`, loaded via symlink at
  `~/.pi/agent/extensions/bash-guard.ts`.
- Runtime config externalized to `~/.pi/agent/bash-guard.json`
  (currently only `votesPerModel` is configurable).
- Added `CONTEXT.md` and `AGENTS.md` documenting architecture, config schema,
  constants, and design decisions.

For full architecture and configuration details, see
[`CONTEXT.md`](./CONTEXT.md) and [`AGENTS.md`](./AGENTS.md).

## License

MIT — see [`LICENSE`](./LICENSE). Retains the original copyright of
[Pascal de Ladurantaye](https://github.com/pascal-de-ladurantaye).
