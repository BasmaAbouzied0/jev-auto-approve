# jev-auto-approve

Stop clicking "yes" on `git status`. A Claude Code hook where Jev (TypeSafe's System One model) approves read-only shell commands in a few hundred milliseconds, and everything else still asks you.

## Fail-safe by design

The hook can only **allow** or **stay silent**. Silence means Claude Code shows its normal permission prompt. It never blocks and never approves anything risky on a guess.

| Situation | What happens |
|---|---|
| Jev is confident the command is read-only (p ≥ 0.95) | Auto-approved |
| Jev is unsure | You get the normal prompt |
| Jev is down, slow (>3s), rate-limited, or no key | You get the normal prompt |
| `rm`, `git push`, `curl`, installs, redirects, `sudo` and similar | Never sent to Jev. You get the normal prompt |
| Command contains comments or "approve"-style text | Never sent to Jev (prompt-injection guard) |
| Plan mode, bypass mode, non-Bash tools, multi-line commands | Hook stays out of it |

## Measured on real Jev (jev-1.13.0)

16 commands, threshold 0.95:

| | Result |
|---|---|
| State-changing commands wrongly approved | **0 of 8** (all scored 0.01 to 0.03) |
| Read-only commands auto-approved | **6 of 8** (`ls`, `cat`, `grep`, `pwd`, `git log`, `git diff`) |
| Read-only commands that still asked | `git status` (0.82), `python --version` (0.90) |

A wide gap between "safe" and "not safe" scores is what you want: the threshold sits in empty space. A missed approval costs you one click. A wrong approval never happened.

## Install

1. Get a key at [console.typesafe.ai/keys](https://console.typesafe.ai/keys) and set `TYPESAFE_API_KEY`
2. Clone this repo somewhere permanent
3. Add to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "python /path/to/jev_auto_approve.py", "timeout": 5 }
        ]
      }
    ]
  }
}
```

Start with `JEV_AUTO_APPROVE_DRY=1` and `JEV_AUTO_APPROVE_LOG=~/jev-approve.log` for a day. It logs what it *would* approve without approving anything. Turn dry mode off once you trust it.

## Settings

| Env var | Default | |
|---|---|---|
| `JEV_AUTO_APPROVE_THRESHOLD` | `0.95` | Raise it to be stricter |
| `JEV_AUTO_APPROVE_TIMEOUT` | `3` | Seconds before falling back to the prompt |
| `JEV_AUTO_APPROVE_DRY` | off | `1` = log only, never approve |
| `JEV_AUTO_APPROVE_LOG` | off | Path to a log file. Off by default because commands can contain secrets |
| `JEV_MODEL` | `jev-latest` | |

## Test

```
python test_jev_auto_approve.py
```

Runs offline with a fake Jev. No key needed.

## Why this works

Permission prompts are a yes/no decision on a short string. That's exactly what Jev is built for: a calibrated probability you can put a threshold on, instead of an LLM's "looks fine". Standard library only, one file, no dependencies.

## License

MIT
