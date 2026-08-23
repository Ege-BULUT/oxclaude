---
name: init-oxclaude
description: Set up an `oxclaude` command that runs Claude Code against OpenRouter's free Ox Alpha stealth model (stealth/ox-alpha), leaving the user's normal `claude` command and Anthropic account untouched. Use when the user invokes /init-oxclaude, or asks to run Ox Alpha / OpenRouter models inside Claude Code.
metadata:
  trigger: /init-oxclaude, "use ox-alpha in claude code", "run claude code on openrouter"
---

# init-oxclaude

End state: the user has two commands.

- `claude`: unchanged, their normal Anthropic account.
- `oxclaude`: same CLI, same project, routed to OpenRouter's `stealth/ox-alpha`, with **Ox Alpha** selectable in `/model`.

## Rules

- **Never handle the API key.** Do not ask the user to paste it into chat, do not read the key file, do not echo it, do not write it into any command you run. Give the user a command with a `sk-or-v1-YOUR-KEY-HERE` placeholder and let them run it themselves.
- Do not put `ANTHROPIC_*` exports at the top level of a shell profile. That hijacks every `claude` session. Only the wrapper function may set them.
- Show the user each command before running it, and run only the setup commands described here.

## Step 0: detect the environment

Run what applies:

- OS: `uname -s` (Darwin / Linux). On Windows PowerShell use `$PSVersionTable.OS`.
- Shell: `echo $SHELL` (expect `/bin/zsh` on macOS, often `/bin/bash` on Linux).
- Claude Code present: `claude --version`. If missing, stop and link <https://docs.claude.com/en/docs/claude-code/setup>.
- Already installed: `grep -n oxclaude ~/.zshrc ~/.bashrc 2>/dev/null` (PowerShell: `Select-String oxclaude $PROFILE`). If found, skip to Step 3.
- Existing global override: `grep -n "ANTHROPIC_BASE_URL\|ANTHROPIC_AUTH_TOKEN" ~/.zshrc ~/.bashrc 2>/dev/null`. If found, tell the user those lines send **every** `claude` session to a third party and offer to remove them.

Pick the shell profile: zsh → `~/.zshrc`, bash → `~/.bashrc` (macOS bash → `~/.bash_profile`), PowerShell → `$PROFILE`.

## Step 1: the user gets an OpenRouter key

Tell them, with the links:

1. Open <https://openrouter.ai/settings/keys> and sign in. **Continue with GitHub** (or Google) is one click, no password.
2. Click **Create Key**, name it, copy the `sk-or-v1-…` value. It is shown only once.
3. Ox Alpha is **free** during preview ($0 in / $0 out, 1M context), so no credits are needed: <https://openrouter.ai/stealth/ox-alpha>

Also state once: stealth models retain prompts for the provider behind them: no secrets, no client code. Logging options: <https://openrouter.ai/settings/privacy>

Then give them the key-store command to run themselves (key never passes through you):

macOS / Linux:

```bash
mkdir -p ~/.config/openrouter && printf '%s' 'sk-or-v1-YOUR-KEY-HERE' > ~/.config/openrouter/key && chmod 600 ~/.config/openrouter/key
```

Windows (PowerShell):

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\.config\openrouter" | Out-Null
Set-Content -NoNewline -Path "$HOME\.config\openrouter\key" -Value 'sk-or-v1-YOUR-KEY-HERE'
icacls "$HOME\.config\openrouter\key" /inheritance:r /grant:r "$($env:USERNAME):(R,W)" | Out-Null
```

Users who already export `OPENROUTER_API_KEY` can skip this. The wrapper prefers that variable.

## Step 2: install the wrapper

### macOS / Linux (zsh or bash)

Append to the profile from Step 0 (`~/.zshrc` shown; swap the path for bash):

```bash
cat >> ~/.zshrc <<'EOF'

# Ox Alpha (OpenRouter): only inside `oxclaude`; plain `claude` stays on Anthropic
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

### Windows (PowerShell 5+ / 7)

```powershell
if (!(Test-Path $PROFILE)) { New-Item -ItemType File -Path $PROFILE -Force | Out-Null }
Add-Content -Path $PROFILE -Value @'

# Ox Alpha (OpenRouter) - only inside oxclaude; plain claude stays on Anthropic
function oxclaude {
  $keyFile = Join-Path $HOME ".config\openrouter\key"
  $key = if ($env:OPENROUTER_API_KEY) { $env:OPENROUTER_API_KEY }
         elseif (Test-Path $keyFile) { (Get-Content $keyFile -Raw).Trim() }
  if (-not $key) { Write-Error "No OpenRouter key: put it in $keyFile or set OPENROUTER_API_KEY"; return }
  $vars = [ordered]@{
    ANTHROPIC_BASE_URL                        = "https://openrouter.ai/api"
    ANTHROPIC_AUTH_TOKEN                      = $key
    ANTHROPIC_API_KEY                         = ""
    ANTHROPIC_MODEL                           = "stealth/ox-alpha"
    ANTHROPIC_SMALL_FAST_MODEL                = "stealth/ox-alpha"
    ANTHROPIC_CUSTOM_MODEL_OPTION             = "stealth/ox-alpha"
    ANTHROPIC_CUSTOM_MODEL_OPTION_NAME        = "Ox Alpha"
    ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION = "OpenRouter stealth - free - 1M context"
  }
  $old = @{}
  foreach ($k in $vars.Keys) {
    $old[$k] = [Environment]::GetEnvironmentVariable($k)
    Set-Item -Path "env:$k" -Value $vars[$k]
  }
  try { claude @args }
  finally {
    foreach ($k in $old.Keys) {
      if ($null -eq $old[$k]) { Remove-Item -Path "env:$k" -ErrorAction SilentlyContinue }
      else { Set-Item -Path "env:$k" -Value $old[$k] }
    }
  }
}
'@
. $PROFILE
```

WSL and Git Bash on Windows use the macOS/Linux block instead.

Other shells (fish, nushell): same eight variables, set for one command only.

## Step 3: verify

1. `oxclaude --version` → prints the Claude Code version. If it prints the "No OpenRouter key" error, Step 1 was not completed.
2. `oxclaude` in a project, then `/status` → API base reads `https://openrouter.ai/api`, model reads `stealth/ox-alpha`.
3. `/model` → **Ox Alpha** appears as a pickable entry.
4. `claude` (plain) still starts on the user's normal account.

Usage shows up at <https://openrouter.ai/activity>.

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `401 Unauthorized` | Bad key, or a real `ANTHROPIC_API_KEY` is winning. The wrapper blanks it. Check it was not re-set after. |
| `404 not found` | Base URL must be `https://openrouter.ai/api`, **not** `/api/v1`. Claude Code appends `/v1/messages`. |
| `model not found` | ID must be exactly `stealth/ox-alpha`. Stealth models get renamed or retired. Check <https://openrouter.ai/stealth/ox-alpha>. |
| `oxclaude: command not found` | Profile not reloaded, or written to a profile the shell does not read. Re-run `source <profile>` / `. $PROFILE`. |
| Ox Alpha missing from `/model` | Requires Claude Code 2.1+ (`ANTHROPIC_CUSTOM_MODEL_OPTION` support). Typing the model name into `/model` still works. |
| Plain `claude` also on OpenRouter | Leftover top-level `export ANTHROPIC_*` lines in a profile. Remove them; only the function should set these. |

## Notes

- One process = one provider. A running session cannot mix Anthropic models and OpenRouter models, because the base URL and token are process-wide. Start `oxclaude` for Ox Alpha, `claude` for Anthropic.
- Uninstall: delete the `oxclaude` block from the profile and `rm ~/.config/openrouter/key` (PowerShell: `Remove-Item "$HOME\.config\openrouter\key"`).
