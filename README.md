<div align="center">

```
 ██████╗  ██╗  ██╗          ███████╗ ██╗       █████╗  ██╗   ██╗ ██████╗  ███████╗
██╔═══██╗ ╚██╗██╔╝         ██╔════╝  ██║      ██╔══██╗ ██║   ██║ ██╔══██╗ ██╔════╝
██║   ██║  ╚███╔╝  ███████ ██║       ██║      ███████║ ██║   ██║ ██║  ██║ █████╗  
██║   ██║  ██╔██╗  ███████ ██║       ██║      ██╔══██║ ██║   ██║ ██║  ██║ ██╔══╝  
╚██████╔╝ ██╔╝ ██╗         ╚██████╗  ███████╗ ██║  ██║ ╚██████╔╝ ██████╔╝ ███████╗
 ╚═════╝  ╚═╝  ╚═╝          ╚═════╝  ╚══════╝ ╚═╝  ╚═╝  ╚═════╝  ╚═════╝  ╚══════╝
```

**[oxclaude.vercel.app](https://oxclaude.vercel.app)**

**Run Claude Code on OpenRouter's free Ox Alpha stealth model — without touching your Anthropic account.**

[![Claude Code skill](https://img.shields.io/badge/claude--code-skill-6e56cf.svg?style=flat-square)](SKILL.md)
[![Model: stealth/ox-alpha](https://img.shields.io/badge/model-stealth%2Fox--alpha-black.svg?style=flat-square)](https://openrouter.ai/stealth/ox-alpha)
[![Price: $0 preview](https://img.shields.io/badge/price-%240%20in%20%2F%20%240%20out-3fb950.svg?style=flat-square)](https://openrouter.ai/stealth/ox-alpha)
[![Context: 1M tokens](https://img.shields.io/badge/context-1M%20tokens-blue.svg?style=flat-square)](https://openrouter.ai/stealth/ox-alpha)

</div>

---

`oxclaude` is a tiny shell wrapper: same Claude Code CLI, same project, but routed to
[`stealth/ox-alpha`](https://openrouter.ai/stealth/ox-alpha) on OpenRouter — **free during its
preview** ($0 in / $0 out, 1M-token context, tool calling). Your plain `claude` command and your
Anthropic subscription stay exactly as they were.

This repo ships it two ways:

1. **`/init-oxclaude` skill** — [`SKILL.md`](SKILL.md) teaches a coding agent (Claude Code,
   opencode, …) to set everything up for you. Drop it in your skills folder and type
   `/init-oxclaude`.
2. **Manual install** — copy one shell function into your profile yourself (below).

Either way you end up with two commands:

| Command | What runs |
| --- | --- |
| `claude` | Your normal Anthropic account — untouched. |
| `oxclaude` | Same CLI on OpenRouter's `stealth/ox-alpha`, with **Ox Alpha** selectable in `/model`. |

## Quick start (manual)

**1. Get a free OpenRouter key.**
Sign in at [openrouter.ai/settings/keys](https://openrouter.ai/settings/keys) (Continue with
GitHub is one click), click **Create Key**, copy the `sk-or-v1-…` value. No credits needed while
Ox Alpha is in free preview.

Store it where only you can read it:

```bash
mkdir -p ~/.config/openrouter && printf '%s' 'sk-or-v1-YOUR-KEY-HERE' > ~/.config/openrouter/key && chmod 600 ~/.config/openrouter/key
```

**2. Add the wrapper to `~/.zshrc`** (bash: `~/.bashrc`; macOS bash: `~/.bash_profile`):

```bash
cat >> ~/.zshrc <<'EOF'

# Ox Alpha (OpenRouter) — only inside `oxclaude`; plain `claude` stays on Anthropic
oxclaude() {
  local key="${OPENROUTER_API_KEY:-$(cat ~/.config/openrouter/key 2>/dev/null)}"
  if [ -z "$key" ]; then
    echo "No OpenRouter key: put it in ~/.config/openrouter/key or export OPENROUTER_API_KEY" >&2
    return 1
  fi
  ANTHROPIC_BASE_URL="https://openrouter.ai/api" \
  ANTHROPIC_AUTH_TOKEN="$key" \
  ANTHROPIC_API_KEY= \
  ANTHROPIC_MODEL="stealth/ox-alpha" \
  ANTHROPIC_SMALL_FAST_MODEL="stealth/ox-alpha" \
  ANTHROPIC_CUSTOM_MODEL_OPTION="stealth/ox-alpha" \
  ANTHROPIC_CUSTOM_MODEL_OPTION_NAME="Ox Alpha" \
  ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION="OpenRouter stealth - free - 1M context" \
  command claude "$@"
}
EOF
source ~/.zshrc
```

Already export `OPENROUTER_API_KEY`? The wrapper prefers it — skip the key file.

**3. Verify.**

```bash
oxclaude          # in any project
/status           # API base: https://openrouter.ai/api · model: stealth/ox-alpha
/model            # "Ox Alpha" appears as a pickable entry
```

Usage shows up within seconds on [openrouter.ai/activity](https://openrouter.ai/activity).

## Install the skill instead

Let your agent do steps 1–3 (it never touches the key — you paste it into your own terminal):

```bash
mkdir -p ~/.claude/skills/init-oxclaude
curl -fsSL https://raw.githubusercontent.com/Ege-BULUT/oxclaude/main/SKILL.md \
  -o ~/.claude/skills/init-oxclaude/SKILL.md
```

Then in Claude Code: `/init-oxclaude`. Windows users get a PowerShell version of the wrapper
from the same skill.

## Why a wrapper and not exports?

Putting `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` at the top level of a shell profile sends
**every** `claude` session to a third party and silently benches your Anthropic subscription.
The wrapper scopes all eight variables to one process: `claude` → Anthropic, `oxclaude` →
OpenRouter. Nothing else changes.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `401 Unauthorized` | Bad key, or a real `ANTHROPIC_API_KEY` is winning. The wrapper blanks it — make sure no profile re-sets it. |
| `404 not found` | Base URL must be `https://openrouter.ai/api`, **not** `/api/v1`. Claude Code appends `/v1/messages` itself. |
| `model not found` | ID must be exactly `stealth/ox-alpha` — check the [model page](https://openrouter.ai/stealth/ox-alpha); stealth models get renamed or retired. |
| `oxclaude: command not found` | Profile not reloaded: `source ~/.zshrc` (or `. $PROFILE`). |
| Ox Alpha missing from `/model` | Needs Claude Code 2.1+ for `ANTHROPIC_CUSTOM_MODEL_OPTION`; typing the model name still works. |
| Plain `claude` also hits OpenRouter | Leftover top-level `export ANTHROPIC_*` lines in a profile — remove them; only the function should set these. |

Full table plus setup details live in [`SKILL.md`](SKILL.md).

## Notes

- One process = one provider: a running session can't mix Anthropic and OpenRouter models. Start
  `oxclaude` for Ox Alpha, `claude` for Anthropic.
- Stealth models may retain prompts for the provider behind them — keep secrets and client code
  out. Logging options: [openrouter.ai/settings/privacy](https://openrouter.ai/settings/privacy).
- Uninstall: delete the `oxclaude` block from your profile and `rm ~/.config/openrouter/key`.

## Links

- **Guide / landing page:** [oxclaude.vercel.app](https://oxclaude.vercel.app)
- **Model:** [openrouter.ai/stealth/ox-alpha](https://openrouter.ai/stealth/ox-alpha)
- **API keys:** [openrouter.ai/settings/keys](https://openrouter.ai/settings/keys)
