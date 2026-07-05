# AGENTS — bash-guard

> **Read [`CONTEXT.md`](./CONTEXT.md) first.** It has the full architecture,
> configuration schema, constants, and design decisions. This file is the short
> rulebook.

## What this is

`bash-guard` is a single-file pi extension (`bash-guard.ts`) that intercepts
`bash` tool calls and runs a multi-model adversarial security review before
allowing execution. Users toggle it with `/guard on|off|debug`.

## Operational facts (must-know before editing)

- **It's a pi package now.** `package.json` declares `bash-guard.ts` under the
  `pi.extensions` manifest, so users install it with
  `pi install git:github.com/rhian-cs/pi-bash-guard` (cloned to
  `~/.pi/agent/git/...`). The old manual-symlink flow still works for hacking
  on the source — see README install section.
- **No build step.** TypeScript is loaded via jiti. Reload with `/reload` or a
  new session. The config + voter caches are invalidated on `session_start`.
- **Runtime config is NOT in this repo.** It lives at
  `~/.pi/agent/bash-guard.json` (resolved via pi's `getAgentDir()`). Never copy
  it in or commit it — it can hold provider/model choices and must not leak
  secrets.
- **Git:** branch `main`, origin `git@github.com:rhian-cs/pi-bash-guard.git`,
  no tags yet. Don't stage/commit unless explicitly asked.
- **This is a security control.** The `SAFE_COMMAND_PATTERNS` whitelist and the
  voter/explainer prompts are safety-critical — be deliberate and conservative
  when changing them. A too-permissive whitelist silently bypasses the review.

## Maintenance contract

**Every change to the extension or its configuration story must keep these
docs in sync:**

1. Update [`CONTEXT.md`](./CONTEXT.md) — flow, constants, config schema (§5),
   design decisions (§8).
2. Update this `AGENTS.md` if the high-level shape or the "Operational facts"
   above changed.
3. If the JSON config schema changes, update CONTEXT.md §5 and the live-config
   note in §5.
4. Currently **only `votesPerModel`** is configurable; `voteTimeoutMs` and
   `explainerContextMessages` are still constants. If you wire those up, update
   CONTEXT.md §5/§7.

## Quick map

| Topic | Where |
| --- | --- |
| Full architecture / runtime flow | CONTEXT.md §3 |
| Config schema + fallbacks | CONTEXT.md §5 |
| Whitelist & shell guards | CONTEXT.md §6 |
| Constants & tunables | CONTEXT.md §7 |
| Why decisions were made | CONTEXT.md §8 |
| Change checklist | CONTEXT.md §9 |
