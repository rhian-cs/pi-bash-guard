# CONTEXT — bash-guard

Living context document for the **bash-guard** pi extension. Read this before
working on the project. See [`AGENTS.md`](./AGENTS.md) for the short rules.

> **Maintenance contract:** when you change the extension or its configuration
> story, update this file **and** `AGENTS.md` to match. Keep them in sync.

---

## 1. What this is

`bash-guard` is a [pi](https://github.com/earendil-works/pi-coding-agent)
extension that intercepts every `bash` tool call and runs an **adversarial
security review** before the command is allowed to execute.

It runs multiple fast "voter" models in parallel, each returning a single
`YES`/`NO`. Based on the consensus:

| Outcome | Action |
| --- | --- |
| Unanimous YES | Auto-allow (notification; debug dialog if `/guard debug`) |
| Unanimous NO | Block, show explanation + override prompt |
| Split / mixed | Block, show explanation + override prompt |
| All abstain (timeout/error) | Treated as not-decided; flows to split/block path |

Timeouts and API errors count as **abstentions** and are ignored in the tally.

The user toggles it with `/guard on|off|debug`. It is **not** exposed as a tool,
so the LLM cannot disable it.

Override memory: when the user overrides a NO/split decision for a command,
that exact command is recorded. Repeating it later auto-allows with a
notification, and prior overrides are fed to voters as context so they learn
user preferences.

---

## 2. Repository layout

```
/home/lotus/Dev/Pi/bash-guard/
├── AGENTS.md          # short agent instructions (points here)
├── CONTEXT.md         # this file
├── README.md          # user-facing install + usage docs
├── LICENSE            # MIT
├── package.json       # pi-package manifest (declares bash-guard.ts as an extension)
└── bash-guard.ts      # the extension (single-file, TypeScript)
```

**The runtime config is NOT in this repo.** It lives at
`~/.pi/agent/bash-guard.json` (see §5). Do not copy it in — it may contain
provider/model choices specific to a machine and must not leak secrets.

### Loading / install

- **Published as a pi package.** `package.json` carries a `pi.extensions`
  manifest pointing at `bash-guard.ts`. Users install with
  `pi install git:github.com/rhian-cs/pi-bash-guard`, which clones the repo to
  `~/.pi/agent/git/github.com/rhian-cs/pi-bash-guard/` and registers it in
  `~/.pi/agent/settings.json`. Core pi packages (`@earendil-works/*`) are
  declared as `peerDependencies` with `"*"` per pi convention — pi bundles
  them, so they must NOT be bundled here.
- **Hacking on the source** still uses the manual symlink:
  `~/.pi/agent/extensions/bash-guard.ts` → this repo's `bash-guard.ts`. (If a
  `pi install` copy is also registered, `pi remove` it first to avoid double
  loading.)
- pi auto-discovers `~/.pi/agent/extensions/*.ts` and installed pi packages.
  Editing the source here and running `/reload` (or starting a new session)
  picks up changes — **no build step**. Extensions load via
  [jiti](https://github.com/unxs/jiti), so plain TypeScript works without
  compilation.
- The git repo lives on branch `main` at origin
  `git@github.com:rhian-cs/pi-bash-guard.git` (no tags yet).
  Committing/pushing is the owner's job, not the agent's.

---

## 3. How it works (runtime flow)

Entry point: `export default function (pi: ExtensionAPI)`.

1. **`pi.registerCommand("guard", ...)`** — `/guard on|off|debug`. State is
   persisted via `pi.appendEntry("bash-guard-state", ...)` and restored on
   `session_start` (reverse scan for last entry).
2. **`pi.on("session_start", ...)`** — restore `guardEnabled`/`debugEnabled`,
   rebuild `overrideHistory` (forward scan of all `bash-guard-override`
   entries), invalidate caches (`cachedVoterModels`, `cachedConfig`), set
   status line.
3. **`pi.on("tool_call", ...)`** with `isToolCallEventType("bash", event)` —
   the main hook. Order of checks:
   1. `guardEnabled` off → passthrough.
   2. `isWhitelisted(command)` → passthrough (see §6 whitelist).
   3. Exact-match in `overrideHistory` → auto-allow + notify.
   4. `resolveVoterModels(ctx)` → get available voter models from config.
   5. If no voter models available: in headless → block; in UI → `confirm()`.
   6. `distributeVoters(models)` → expand to `len(models) × votesPerModel`
      voters, round-robin across models.
   7. `runVoteTracking(...)` → fire all votes in parallel (each with its own
      `VOTE_TIMEOUT_MS`); live-updates the TUI. Esc cancels.
   8. `computeVoteResult(...)` → unanimous yes/no or split.
   9. Unanimous YES → auto-allow. Otherwise → `showReviewDialog(...)`, which
      calls `getExplanation(...)` and asks y/n (interactive) or blocks
      (headless). On user **allow**, record an override entry.

### Explainer

`getExplanation()` builds a context window from the last
`EXPLAINER_CONTEXT_MESSAGES` session entries (user/assistant text, thinking,
tool calls, truncated tool results) and asks the explainer model for a fixed
3-field summary (What it does / Why it's being run / Risk). Model resolution
(`resolveExplainerModel`) tries the configured explainer, then falls back to
`ctx.model` (the main session model). Reasoning is forced off for the
explainer.

### UI / mode handling

- `ctx.hasUI` is `true` in TUI and RPC, `false` in print/JSON modes.
- Rich TUI dialogs use `ctx.ui.custom(...)` (vote tracker, review dialog with
  `Markdown`, `DynamicBorder`, vote icons/summary).
- `ctx.ui.custom()` returns `undefined` in RPC mode → code falls back to
  `ctx.ui.confirm()`, which works over the extension UI sub-protocol.
- Fully headless (no UI): votes run without an abort signal; results either
  auto-allow (unanimous YES) or return `{ block: true, reason }`.

---

## 4. Dependencies (pi packages)

| Package | Symbols used |
| --- | --- |
| `@earendil-works/pi-ai/compat` | `complete`, `Model`, `TextContent`, `ThinkingContent`, `ToolCall`, `ToolResultMessage`, `UserMessage` |
| `@earendil-works/pi-coding-agent` | `ExtensionAPI`, `ExtensionContext`, `isToolCallEventType`, `DynamicBorder`, `getMarkdownTheme`, **`getAgentDir`** |
| `@earendil-works/pi-tui` | `Container`, `Markdown`, `Spacer`, `Text`, `Key`, `matchesKey` |
| `node:fs` / `node:path` | `readFileSync`, `join` |

### Key `ctx` / `pi` APIs relied on

- `ctx.modelRegistry.find(provider, id)` and
  `ctx.modelRegistry.getApiKeyAndHeaders(model)` (note: `find` throws if the
  model is unknown — always wrapped in try/catch).
- `ctx.model`, `ctx.hasUI`, `ctx.signal`, `ctx.cwd`.
- `ctx.ui.{notify,confirm,custom,setStatus,theme}`.
- `ctx.sessionManager.{getBranch,getEntries}`.
- `pi.on`, `pi.registerCommand`, `pi.appendEntry`.

---

## 5. Configuration (`~/.pi/agent/bash-guard.json`)

The extension is configured by a JSON file resolved via pi's own
`getAgentDir()`:

```ts
const CONFIG_PATH = join(getAgentDir(), "bash-guard.json"); // → ~/.pi/agent/bash-guard.json
```

Using `getAgentDir()` (not a hardcoded `~/.pi/...`) is intentional: it matches
how pi locates its own `settings.json` / `models.json`, respects the
`PI_CODING_AGENT_DIR` env override, and follows rebranded config-dir names.

### Schema

```jsonc
{
  "votesPerModel": 2,            // positive integer; default 5
  "voters": [                    // array; default = single haiku entry
    {
      "provider": "zai",         // required, must match modelRegistry
      "id": "glm-5-turbo",       // required
      "label": "glm-5-turbo"     // optional, shown in UI; defaults to id
    }
  ],
  "explainer": {                 // object; default = anthropic/claude-haiku-4-5
    "provider": "zai",           // required
    "id": "glm-5.1"              // required
  }
}
```

### Fallback semantics

- Missing / unreadable / malformed file → **all** hardcoded defaults.
- Partial config is merged **field by field**: each valid field overrides its
  default; invalid values keep the default. Concretely:
  - `voters`: must be a non-empty array of objects with string `provider` +
    `id`; bad entries are filtered; if **all** are bad, the default list is
    kept. `label` defaults to `id` when omitted/empty.
  - `explainer`: must be an object with string `provider` + `id`.
  - `votesPerModel`: must be a positive integer (`Number.isInteger && > 0`).
- The config is cached (`cachedConfig`) and reloaded on every `session_start`,
  so edits take effect next session.

### Defaults (in code, when config is absent/invalid)

- `DEFAULT_VOTERS` = `[{ provider: "anthropic", id: "claude-haiku-4-5", label: "haiku-4.5" }]`
- `DEFAULT_EXPLAINER` = `{ provider: "anthropic", id: "claude-haiku-4-5" }`
  (built from the `EXPLAINER_PROVIDER` / `EXPLAINER_MODEL_ID` constants).
- `VOTES_PER_MODEL` = `5`.

### Current live config (this machine)

Lives at `~/.pi/agent/bash-guard.json` (outside this repo):

```jsonc
{
  "votesPerModel": 2,
  "voters": [
    { "provider": "zai", "id": "glm-5-turbo", "label": "glm-5-turbo" },
    { "provider": "zai", "id": "glm-4.5-air", "label": "glm-4.5-air" }
  ],
  "explainer": { "provider": "zai", "id": "glm-5.1" }
}
```

→ 2 models × 2 votes = **4 parallel voters**. Rationale: `turbo`/`air` tiers
are fast/cheap (good for the per-command voters); two different families give
real consensus diversity vs. one model × N. The explainer uses a more capable
model since it runs only on blocked/split commands and needs reasoning +
writing.

---

## 6. Whitelist & shell guards

`isWhitelisted(command)` short-circuits the review entirely for obviously-safe
read-only commands.

- Collapses line-continuations / stray newlines first.
- Disqualifies the whitelist if any of: `UNSAFE_SHELL_CHARS`
  (`|;&` backtick newline), `SUBSHELL_PATTERN` (`$(`), `REDIRECT_PATTERN`
  (`>` / `>>`).
- Then matches `SAFE_COMMAND_PATTERNS` — a curated allowlist of read-only
  verbs and safe `git` subcommands (`status|log|diff|show|branch|tag|remote|
  stash list|config --get`). Deliberately **omits** `find`/`fd` (`-exec`,
  `-delete`), `awk` (`system()`), and `sort` (`-o` writes files).

> This is a security control. Be very deliberate about adding patterns here —
> a too-permissive entry silently bypasses the whole review.

---

## 7. Key constants & tunables

| Constant | Value | Notes |
| --- | --- | --- |
| `VOTES_PER_MODEL` | `5` | Default only; overridable by `config.votesPerModel` |
| `VOTE_TIMEOUT_MS` | `5000` | Per-vote abort timeout; abstain on timeout |
| `EXPLAINER_CONTEXT_MESSAGES` | `20` | Session entries fed to the explainer |
| `EXPLAINER_PROVIDER` / `EXPLAINER_MODEL_ID` | `anthropic` / `claude-haiku-4-5` | Source of `DEFAULT_EXPLAINER` |
| `VOTER_SYSTEM_PROMPT` / `EXPLAINER_SYSTEM_PROMPT` | — | The review + explanation instructions |
| `SAFE_COMMAND_PATTERNS` | regex list | See §6 |

`votesPerModel`, `voteTimeoutMs`, and `explainerContextMessages` are **not**
all wired to the config — only `votesPerModel` is. If you make the others
configurable, update this table and §5.

---

## 8. Design decisions worth knowing

- **Why multiple voter models?** One model voting N times is nearly
  deterministic; spreading votes across 2+ families gives genuine consensus.
  `distributeVoters` round-robins so votes interleave across models.
- **Why `getAgentDir()` / not hardcoded `~/.pi`?** Rebrand-safe, env-aware,
  matches pi's own config resolution. (Project-local config, by contrast,
  should use `CONFIG_DIR_NAME` + a `ctx.isProjectTrusted()` gate — not used
  here since this config is global.)
- **Explainer fallback chain** is configured model → `ctx.model`. Both have
  reasoning forced off.
- **Override memory is append-only.** Restore reads the *last* state entry
  (reverse scan) but *all* override entries (forward scan).
- **Caches** (`cachedVoterModels`, `cachedConfig`) are intentionally
  invalidated on `session_start` so new API keys / config take effect.

---

## 9. Maintenance checklist

When changing the extension:

- [ ] Update `CONTEXT.md` (flow, constants, config schema, decisions).
- [ ] Update `AGENTS.md` if the high-level shape or operational facts changed.
- [ ] If you change the config schema, update §5 and the live config comment.
- [ ] If you touch `SAFE_COMMAND_PATTERNS` or the prompts, re-read §6 — this is
  a security control.
- [ ] Keep `VOTES_PER_MODEL` as the documented default even if it's
  configurable.
- [ ] Do **not** stage/commit unless explicitly asked (see global agent rules).
  Do **not** commit the runtime `~/.pi/agent/bash-guard.json`.
