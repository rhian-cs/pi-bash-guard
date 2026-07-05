# Bash Guard

An adversarial security review extension for [pi](https://github.com/earendil-works/pi-coding-agent).

Bash Guard intercepts every `bash` command an agent tries to run and asks a small
panel of fast, independent LLM "voters" whether the command is safe **before**
letting it execute. If the voters disagree or say no, you get a human-readable
explanation of the risk and a chance to allow or block the command.

> **Status:** This is a derivative port of the original Bash Guard extension by
> [Pascal de Ladurantaye](https://github.com/pascal-de-ladurantaye), retargeted
> at the `@earendil-works/*` pi packages. The canonical implementation and
> design live upstream:
> **<https://github.com/pascal-de-ladurantaye/pi-agent/blob/main/extensions/bash-guard>**

---

## Table of contents

- [What it does](#what-it-does)
- [Requirements](#requirements)
- [Install](#install)
- [Quick start](#quick-start)
- [How voting works](#how-voting-works)
- [Configuration](#configuration)
- [What counts as "safe" automatically](#what-counts-as-safe-automatically)
- [Modes](#modes)
- [Troubleshooting](#troubleshooting)
- [Further reading](#further-reading)
- [License](#license)

---

## What it does

- Hooks into pi's `bash` tool calls before they execute.
- Fires off several "voter" model calls **in parallel**, each answering a single
  `YES` (safe) or `NO` (unsafe).
- Tallies the results:

  | Outcome                  | What happens                                                       |
  | ------------------------ | ------------------------------------------------------------------ |
  | All voters say `YES`     | Command runs automatically (you'll see a notification).            |
  | All voters say `NO`      | Command is blocked; you see an explanation and an allow/block prompt. |
  | Mixed votes, or abstains | Same as `NO` — block, explain, and prompt.                         |

- When you **allow** a blocked command, Bash Guard remembers that exact command
  string and auto-approves it next time. (Past overrides are also fed back to
  the voters as context, so they learn your preferences.)
- Obviously-safe read-only commands (like `ls` or `git status`) skip the review
  entirely — see [What counts as "safe" automatically](#what-counts-as-safe-automatically).

## Requirements

- A working [pi](https://github.com/earendil-works/pi-coding-agent) install.
- At least one configured model provider for the **voters** and one for the
  **explainer**. By default Bash Guard uses
  `anthropic` / `claude-haiku-4-5` for both, so you'll need an Anthropic API
  key set up in pi. You can point it at any provider/model you like via
  [configuration](#configuration).

## Install

Bash Guard is a [pi package](https://pi.dev/packages). The one-line install:

```bash
pi install git:github.com/rhian-cs/pi-bash-guard
```

That's it. Restart pi (or run `/reload` in a session) and Bash Guard is on by
default — you'll see a `🔒 guard` status indicator.

> **What this does:** pi clones the repo into `~/.pi/agent/git/github.com/rhian-cs/pi-bash-guard/`,
> registers it in `~/.pi/agent/settings.json`, and loads `bash-guard.ts` as an
> extension. The repo's `package.json` declares it as a pi package.

### Updating

```bash
pi update --extensions   # update all installed pi packages, including this one
```

To move to a specific tag or commit later, pin a ref:

```bash
pi install git:github.com/rhian-cs/pi-bash-guard@v0.1.0
```

### Uninstall

```bash
pi remove git:github.com/rhian-cs/pi-bash-guard
```

### Development / hacking on the extension

If you want to edit the source and have changes picked up instantly, skip
`pi install` and link the repo directly into pi's extensions directory instead.
There's no build step — TypeScript is loaded via
[jiti](https://github.com/unxs/jiti), so just edit and `/reload`.

```bash
# 1. Clone the repo somewhere convenient.
git clone git@github.com:rhian-cs/pi-bash-guard.git ~/src/pi-bash-guard

# 2. Make sure pi's extensions directory exists.
mkdir -p ~/.pi/agent/extensions

# 3. Symlink the extension in. Use an ABSOLUTE path to the source file:
ln -s ~/src/pi-bash-guard/bash-guard.ts ~/.pi/agent/extensions/bash-guard.ts
```

Then `/reload` after every edit. (If you previously ran `pi install`, run
`pi remove git:github.com/rhian-cs/pi-bash-guard` first so the two don't
both load.)

## Quick start

Bash Guard exposes one slash command:

```
/guard on      # turn the guard on (default)
/guard off     # turn it off for this session
/guard debug   # toggle debug mode (shows voter details on allowed commands too)
/guard         # toggle on/off
```

State (on/off, debug) is persisted per session branch, so it survives restarts.

That's it — every `bash` command your agent runs will now be reviewed first.

## How voting works

For each non-whitelisted command, Bash Guard:

1. Picks the configured **voter models** (see [Configuration](#configuration)).
2. Expands them into `len(models) × votesPerModel` parallel votes, round-robined
   across the models. With the default config (one voter model, `votesPerModel`
   = 5) that's **5 voters**. See [Configuration](#configuration) to add more
   models for genuine cross-family consensus.
3. Sends each voter the command plus a system prompt asking for a single
   `YES`/`NO`. Votes have a 5-second timeout; a timeout or API error counts as
   an **abstention** and is ignored.
4. If the result is anything other than unanimous-YES, the **explainer** model
   writes a short 3-field summary (what it does / why it's being run / the
   risk), which is shown alongside an allow/block prompt.

The explainer is deliberately a stronger model than the voters — voters need to
be fast and cheap (they run on every command), while the explainer only runs on
blocked or split decisions.

## Configuration

Out of the box, Bash Guard uses `anthropic/claude-haiku-4-5` for both the
voters and the explainer. If you have an Anthropic API key configured in pi,
you don't need to configure anything — it just works.

To use different models (recommended — see [Choosing models](#choosing-models)
below), create:

```
~/.pi/agent/bash-guard.json
```

(The path is resolved through pi's `getAgentDir()`, so it honors a
`PI_CODING_AGENT_DIR` override the same way pi's own `settings.json` does.)

If the file is missing or malformed, all defaults are used. Partial configs are
merged field-by-field — each valid field overrides its default, invalid values
fall back to the default.

### Quick start: point it at your own models

A minimal config that swaps both the voters and the explainer to models you
already have set up in pi:

```jsonc
{
  "voters": [
    { "provider": "openai",       "id": "gpt-5-mini" },
    { "provider": "google",       "id": "gemini-2.5-flash" }
  ],
  "explainer": { "provider": "openai", "id": "gpt-5" }
}
```

`provider` and `id` must match a model you've configured in pi's model registry
(i.e. something you can already select as your main model). `votesPerModel`
defaults to `5`, so this example runs 2 models × 5 votes = **10 parallel
voters** per command — drop `votesPerModel` to `2` or `3` if that's too many
for your rate limits.

### Full schema

```jsonc
{
  // How many votes each voter model casts. Positive integer. Default: 5.
  "votesPerModel": 2,

  // The voter models. Array; default is a single anthropic/claude-haiku-4-5 entry.
  "voters": [
    {
      "provider": "zai",          // required — must match a provider in pi's model registry
      "id": "glm-5-turbo",        // required — the model id
      "label": "glm-5-turbo"      // optional — shown in the UI; defaults to `id`
    }
  ],

  // The explainer model. Default: anthropic/claude-haiku-4-5.
  "explainer": {
    "provider": "zai",            // required
    "id": "glm-5.1"               // required
  },

  // Shell command run (fire-and-forget, non-blocking) when an approval
  // prompt appears. Any shell command works. Empty/absent = no sound.
  "approvalSoundCommand": "paplay /usr/share/sounds/Oxygen-Sys-Warning.ogg"
}
```

### Choosing models

- **Voters** should be **fast and cheap** — they run on every command. Use two
  *different* model families if you can, so the votes represent genuine
  independent consensus rather than the same model voting N times.
- **Explainer** should be a model capable of reasoning and clear writing — it
  only runs on blocked/split decisions, so latency is less of a concern.
- `votesPerModel` × number of voter models = total parallel API calls per
  command. Tune for your rate limits and latency tolerance.

### Defaults

When the config file is absent or invalid, Bash Guard uses:

| Setting        | Default                          |
| -------------- | -------------------------------- |
| `voters`       | `[{ anthropic, claude-haiku-4-5 }]` |
| `explainer`    | `{ anthropic, claude-haiku-4-5 }`   |
| `votesPerModel`| `5`                              |
| `approvalSoundCommand` | _(disabled — empty string)_ |

### When config takes effect

The config is cached and reloaded at the start of every session, so edits apply
on your next `/reload` or new session.

## What counts as "safe" automatically

To avoid spamming voters on harmless commands, Bash Guard **short-circuits the
review** for an allowlist of obviously-safe, read-only commands (for example
`ls`, `cat`, `git status`, `git log`, `git diff`).

The whitelist is conservative by design. A command is disqualified from the
whitelist (and therefore always reviewed) if it contains:

- Shell metacharacters like `|`, `;`, `&`, backticks, or newlines,
- Subshells like `$(...)`,
- Output redirection (`>`, `>>`).

Tools like `find`/`fd` (which have `-exec`/`-delete`), `awk` (which has
`system()`), and `sort` (which can write files with `-o`) are deliberately
**not** whitelisted.

> ⚠️ This whitelist is a security control. If you're forking or maintaining the
> extension, be very conservative about adding entries — a too-permissive
> pattern silently bypasses the entire review. See [`CONTEXT.md`](./CONTEXT.md)
> §6 for details.

## Modes

Bash Guard works across pi's runtime modes:

- **TUI** — Rich review dialog with the explainer summary, vote tally, and an
  allow/block prompt. `Esc` cancels an in-progress vote.
- **RPC** (e.g. editor integrations) — Falls back to a simple `confirm()` prompt
  over pi's extension UI sub-protocol.
- **Headless** (`print`/`json` modes, no UI) — Unanimous `YES` results
  auto-allow; everything else is blocked (there's no one to prompt).

## Troubleshooting

- **No `🔒 guard` status indicator.** Run `/reload` or restart pi. If you
  installed via `pi install`, confirm it's listed: `pi list`. If you symlinked
  manually, confirm the link: `ls -la ~/.pi/agent/extensions/bash-guard.ts`.
- **Every command is blocked.** Most likely no voter model could be resolved —
  check that the `provider`/`id` values in your config match models you've set
  up in pi, and that you have API keys configured for them.
- **Config changes aren't taking effect.** Config is cached per session — run
  `/reload` or start a new session.
- **Voters keep timing out.** Either your voter models are too slow, or your
  network/API is rate-limited. Try faster models or lower `votesPerModel`.
- **Turn it off entirely for a session:** `/guard off`.

## Further reading

- [`CONTEXT.md`](./CONTEXT.md) — Full architecture, runtime flow, constants,
  and design decisions. Start here if you want to modify the extension.
- [`AGENTS.md`](./AGENTS.md) — Short rulebook for working on the codebase.

## License

MIT — see [`LICENSE`](./LICENSE).

This project retains the original copyright of
[Pascal de Ladurantaye](https://github.com/pascal-de-ladurantaye); the upstream
implementation is the canonical source.
