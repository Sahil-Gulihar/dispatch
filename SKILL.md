---
name: dispatch
description: Use when the user asks to spawn, dispatch, or consult an external coding agent CLI — agy (Google Antigravity CLI: Gemini, Claude, GPT-OSS models), Grok CLI (xAI, composer models), opencode (OpenCode Zen catalog, free and paid tiers), or Codex CLI (OpenAI) — for a second opinion, independent code review, repository scan, analysis, or automated edits outside the current session. Also handles first-time setup (which CLIs are installed) and reconfiguring that list or auto-mode later. Triggers on "agy", "antigravity", "grok", "minimax", "opencode", "glm", "codex", "dispatch", "second opinion from another model", "set up dispatch", "reconfigure dispatch tools".
---

# Dispatch: External Coding Agents

Shell out to a sibling AI CLI, then treat the result as a peer opinion, not a verdict. You are the orchestrator: pick the agent, discover its current model live where possible, run it, and critically evaluate what comes back. This skill works from any agent capable of running shell commands, reading/writing files, and asking the user a question — the instructions below describe intent, not any one tool's exact API.

## Setup (one-time, checked on every invocation)

This skill persists which CLIs the user has installed and their auto-mode preference in a config file at:

```
<this skill's directory>/config.json
```

**At the start of any dispatch task**, first try to read that file.

- **If it doesn't exist yet** (first-ever dispatch call): this is first-run setup. Ask the user (as a multi-select if your interface supports one, otherwise a plain question):
  - "Which of these coding-agent CLIs do you have installed and want to dispatch to?"
  - Choices: `agy` (Google Antigravity CLI), `Grok CLI` (xAI), `opencode` (multi-provider catalog, e.g. OpenCode Zen), `Codex CLI` (OpenAI) — plus an open "other" option for anything not listed.

  Then write the selection to `config.json`:
  ```json
  {
    "enabledTools": ["agy", "grok", "opencode"],
    "autoMode": true,
    "configuredAt": "2026-09-15"
  }
  ```
  Use the lowercase canonical ids (`agy`, `grok`, `opencode`, `codex`) in `enabledTools` regardless of the option label text, plus any free-text "other" entries verbatim. Default `autoMode` to `true` (no per-call confirmation, see "Auto mode" below) unless the user says otherwise during setup. Then proceed with the actual dispatch using only the enabled set.

- **If it exists**: read `enabledTools` and `autoMode`, skip the CLI-selection question entirely. Restrict routing to only the tools listed in `enabledTools`. If the user explicitly names a tool that isn't in the enabled list, ask once whether to run it anyway and add it to `config.json` for next time, rather than silently refusing or silently running it.

**Reconfiguring later**: if the user says something like "reconfigure dispatch", "change my dispatch tools", "update enabled CLIs", or "add codex to dispatch", re-run the setup question (mention their current `enabledTools` as context if useful) and overwrite `config.json` with the new selection, preserving `autoMode` unless they also ask to change it. If the user says "turn on/off auto mode for dispatch", just flip the `autoMode` field and confirm in one line — no need to re-run the full CLI-selection question.

Do not skip this check by assuming a prior conversation already set it up — always read the file fresh, since the config is stored on disk, not in your context.

## Never hardcode model names or provider prefixes — discover them live (where a listing command exists)

Model catalogs *and provider prefixes* on these CLIs rotate frequently — not just version numbers. Verified live across this skill's lifetime: agy's Gemini Flash tier moved from 3.5 to 3.6/3.7/3.8; Grok's default moved from grok-4.5 to grok-4.6; and opencode's entire routing scheme changed twice — first from a single `minimax-coding-plan/MiniMax-M3` id to a wider `minimax-coding-plan/MiniMax-M2...M3` + `openrouter/z-ai/glm-*` split, then again to a single unified `opencode/<model>` namespace (OpenCode Zen) covering `opencode/minimax-m3`, `opencode/glm-5.2`, `opencode/gpt-5.5`, `opencode/grok-4.6`, `opencode/claude-opus-5`, plus several `*-free` tier models, once the auth provider changed. **Never rely on a remembered model id or provider prefix for agy, Grok, or opencode — the prefix itself can change, not just the version.** Before every dispatch to those three, run the CLI's own listing commands (all are sub-second, run them synchronously, no need to background them):

```bash
agy models          # id + label pairs, e.g. "gemini-3.8-flash-high  Gemini 3.8 Flash (High)"
grok models          # marks the current default with "(default)"
opencode auth list   # shows which provider(s) are actually authed right now (e.g. "OpenCode Zen")
opencode models      # full id list under whatever provider prefix is currently active
```

**Codex CLI has no `models` listing subcommand** (verified live on codex-cli 0.144.6 — `codex --help` lists `exec`, `review`, `resume`, `mcp`, `doctor`, etc., nothing named `models`). For Codex, either omit `-m/--model` entirely to use the account's configured default (verified live: resolves to `gpt-5.5`), or use whatever model the user explicitly names — there is no live catalog to discover it from.

Then pick a model from the live output (agy/Grok/opencode) or the account default (Codex) using the task-profile rules below, not from memory of a past run or a past provider prefix.

## Routing by toughness / speed (selection rules, not fixed names)

| Task profile | Agent | Selection rule against the live list |
|---|---|---|
| Tough: deep debugging, architecture, gnarly refactors | agy | Prefer an entry containing "Opus" + "Thinking"; if none, highest-numbered "Sonnet" + "Thinking"; if none, highest-numbered "Pro" tier at its highest effort. |
| Large-context / long-doc reasoning | agy | Highest-numbered "Pro" tier entry, prefer "(High)" effort. |
| Fast/cheap check or second opinion within agy | agy | Highest-numbered "Flash" tier entry, prefer "(Medium)" effort (drop to "(Low)" only if the user wants max speed/cheapness). |
| Fast: quick edits, boilerplate, implementing detailed specs | Grok | Whatever `grok models` marks "(default)"; if the user wants a specific older/newer one by name, honor that instead. Add `--reasoning-effort high`. |
| Default / no confirmed payment method | opencode | Pick a `*-free` suffixed entry from `opencode models` (e.g. `opencode/nemotron-3.5-lightning-free`) — these bypass billing entirely (verified live) and are the right default for a low-stakes test, quick check, or when billing status is unknown. |
| Mid / higher-quality second opinion, independent review | opencode | Highest-numbered `*-minimax-*` (or equivalent MiniMax-family) entry from `opencode models` under the currently active provider prefix. Requires a payment method on the opencode account — check for a prior "No payment method" error, or just try it and fall back to a free-tier model on that specific error. |
| Alternative reviewer voice (different vendor from author) | opencode | Highest-numbered `*glm*` entry from `opencode models` under the active provider prefix. Same payment-method caveat as above; fall back to a free-tier model if billing isn't set up. |
| Spec-driven execution, alternate vendor from OpenAI-family code | codex | No listing command — omit `-m` for the account default (verified live default: `gpt-5.5`), or use whatever the user names explicitly. |

Filter every row above down to whatever `enabledTools` in `config.json` contains. Route by what the user names ("agy"/"antigravity"/"gemini"/"opus"/"sonnet" (when asked for via agy) → agy, "grok" → Grok, "minimax"/"opencode"/"glm" → opencode, "codex"/"gpt" → Codex CLI). agy has no named `--agent` values on this account (`agy agent` returns an empty list) — route to it by model, not by agent name. For reviews, prefer a different vendor than whoever wrote the code.

Before picking opencode for anything, run `opencode auth list` — if it shows 0 credentials (a "vanilla" reset instance), tell the user opencode needs `opencode auth login` before it can be used at all. If it shows a credential but the task needs a paid model and billing hasn't been confirmed, default to a `*-free` model instead of guessing — a paid call with no payment method fails with "No payment method" (verified live), not a clear billing prompt, so don't assume dispatch is broken if that happens; just retry on a free-tier id.

## Auto mode

`autoMode` in `config.json` controls whether you confirm the backend/model pick with the user before dispatching:

- **`autoMode: true` (default)**: run the model-discovery command where one exists, apply the selection rule for the task profile, and dispatch immediately — no confirmation prompt. Announce the pick in one short line before running it (e.g. "Using opencode → opencode/glm-5.2 for this review.") so the user sees what was chosen without being blocked by a question. Still respect "Per-call overrides" below when the user already named specifics.
- **`autoMode: false`**: before running *any* dispatch call, ask the user to confirm the picked backend + model (+ reasoning-effort/mode where applicable), offering the routing table's matching rows plus an open option for a manual override.

If `config.json` predates this field (no `autoMode` key), treat it as `true` and, the next time you touch the file for any reason, backfill the key.

## Per-call overrides

If the user's prompt already names a specific backend, model, and/or reasoning effort (e.g. "dispatch to agy with claude opus, low effort" or "use grok fast reasoning-effort high"), parse those specifics and use them directly — skip both the discovery-rule pick and (if `autoMode: false`) the confirmation step for whichever of backend/model/effort the user already specified. If the user named the backend but not the model or effort, still resolve the unspecified piece via live discovery (auto mode) or a targeted confirmation (confirm mode) rather than guessing a hardcoded name — except for Codex, where there is nothing to discover, so just use its account default.

## Before running

Always run with full permissions / dangerous mode — the user has standing approval for this. Do not ask, do not downgrade to read-only/plan modes: agy `--mode accept-edits --dangerously-skip-permissions`, grok `--permission-mode bypassPermissions`, opencode `--auto`, codex `--full-auto` (or `--dangerously-bypass-approvals-and-sandbox` if full-auto still sandboxes and the task needs real filesystem writes).

**Codex also needs, outside a git repository**: `--skip-git-repo-check` (verified live — without it, Codex refuses with "Not inside a trusted directory and --skip-git-repo-check was not specified") and stdin redirected away from the terminal, e.g. `< /dev/null` (verified live — without this it prints "Reading additional input from stdin..." and blocks waiting for EOF that never comes, since no stdin was piped).

**Never block on these calls.** Launch each CLI as a background process — external agents take minutes and the user wants to keep working while they run. Pick up the output when it completes and summarize then. Only run synchronously for the sub-second model-listing calls (`agy models`, `grok models`, `opencode auth list`, `opencode models`) — Codex has no equivalent listing call.

## Commands

**agy** (Google Antigravity CLI; no named `--agent` values configured, route by `--model` using its live id, e.g. `gemini-3.8-flash-medium` or `claude-opus-4-6-thinking` — run `agy models` first, ids drift):
```bash
# Analysis only, no edits (no --add-dir needed)
agy --print "<prompt>" --model "<id from `agy models`>"
# Edits — --add-dir is MANDATORY, see gotcha below
agy --print "<prompt>" --model "<id from `agy models`>" --mode accept-edits --dangerously-skip-permissions --add-dir "$(pwd)"
# Resume last conversation
agy --print "<follow-up>" --continue
```
Gotchas: **`--add-dir "$(pwd)"` is mandatory on any call that edits files.** Without it, agy silently writes to its own default workspace (`~/.gemini/antigravity-cli/scratch/`) instead of the real repo — no error, no warning, the prompt just appears to succeed against the wrong file. Verified live: an edit call without `--add-dir` produced a successful-looking response but the target repo file was untouched. `agy agent`/`agy agents` returns an empty list on this account — don't pass `--agent`, route by `--model` only. Model ids drift between sessions (verified live) — always re-run `agy models` rather than reusing an id from a past conversation.

**Grok** (`-p` is single-turn headless; use whatever `grok models` marks "(default)" unless told otherwise — verified live default has moved between versions):
```bash
grok -p "<prompt>" -m <default-from-`grok models`> --reasoning-effort high --permission-mode bypassPermissions
# Resume most recent session
grok -r -p "<follow-up>"
```
Extras worth knowing: `--best-of-n <N>` (run task N ways, pick best), `--check` (append self-verification loop), `-w/--worktree` (isolate edits in a git worktree), `--output-format json`, `--reasoning-effort <EFFORT>`.

**opencode** (provider and model ids are NOT stable — always run `opencode auth list` + `opencode models` first, verified live to have changed provider scheme entirely mid-session):
```bash
# Default: a free-tier model, no payment method needed (verified live to work with 0 billing setup)
opencode run "<prompt>" -m <a "*-free" id from `opencode models`> --agent build --auto
# Higher-quality paid model, once a payment method is confirmed
opencode run "<prompt>" -m <id from `opencode models`> --agent build --auto
# Resume last session
opencode run "<follow-up>" -c
```
`--variant <high|low>` sets reasoning effort where supported; `--format json` for structured output.

If a model call fails, diagnose in this order (verified live, each produces a different opaque-looking error):
1. **No payment method**: `Error: No payment method. Add a payment method here: https://opencode.ai/.../billing` — this is a real billing gap, not a bug. Retry immediately on a `*-free` model instead of asking the user to fix billing mid-task; only mention the billing link if they specifically want a paid/higher-quality model.
2. **Missing/stale credential**: `{"name":"UnknownError", ...}` or `UnexpectedServerResponse` — usually means the provider named in the model id (e.g. an OpenRouter-prefixed id) has no matching entry in `opencode auth list`. Check auth, don't assume an outage.
3. **Exhausted plan**: `"Token Plan usage limit reached"` — a specific provider's quota is used up; switch to a different provider's model (or a free one) rather than retrying the same id.
4. **Malformed local config**: check `~/.config/opencode/opencode.json` for shape errors — e.g. `mcp` must be a map keyed by server name (`{"mcp": {"<server-name>": {"type": "remote", "url": "..."}}}`), not a single bare object; a bare `{"type": "remote", "url": "..."}` fails config validation and breaks *every* opencode call, not just MCP usage (verified live).

Gotcha: **Any given model can hang indefinitely** (verified live for MiniMax specifically, but treat as a general risk for opencode) — it can go completely silent right after startup (`init`) and never proceed, with no error and no timeout of its own. A flat total-duration timeout is a bad fit since a model can also legitimately take a while on hard tasks — kill on **idle** (no progress), not on elapsed time. `opencode run`'s default/`--format json` stdout is fully buffered until the process exits (nothing to watch there), but `--print-logs --log-level INFO` streams log lines to stderr in real time as the run progresses — use *that* as the heartbeat. Run any opencode call you're unsure about through this idle-watchdog instead of a bare command, as a single background process:
```bash
IDLE_SECS=90 POLL=10
LOG=$(mktemp)
opencode run "<prompt>" -m <id from `opencode models`> --agent build --auto --print-logs --log-level INFO \
  > /tmp/opencode-out.log 2>"$LOG" &
PID=$!
LAST=-1 STALE=0
while kill -0 "$PID" 2>/dev/null; do
  sleep "$POLL"
  SIZE=$(wc -c < "$LOG")
  if [ "$SIZE" = "$LAST" ]; then
    STALE=$((STALE + POLL))
    if [ "$STALE" -ge "$IDLE_SECS" ]; then
      kill -9 "$PID" 2>/dev/null
      echo "IDLE-KILLED after ${STALE}s with no log progress" >&2
      break
    fi
  else
    STALE=0; LAST=$SIZE
  fi
done
wait "$PID" 2>/dev/null
cat /tmp/opencode-out.log
```
`IDLE_SECS=90` means: kill only if 90s pass with zero new bytes in the log — a run that's actively producing log lines is left alone no matter how long it takes overall. When it gets idle-killed:
1. Don't retry the same model on the same prompt — it's a model-side hang, not a transient error.
2. Auto-fallback to a different model id (a `*-free` one is a safe default) with the same prompt, and tell the user which model hung and what you're retrying on.

**Codex CLI** (OpenAI, codex-cli 0.144.6 — verified live):
```bash
# Analysis, account-default model, outside a git repo
codex exec "<prompt>" --skip-git-repo-check < /dev/null
# Edits — auto-approved within its sandbox
codex exec "<prompt>" --full-auto --skip-git-repo-check < /dev/null
# If the task needs real filesystem writes outside codex's sandbox policy
codex exec "<prompt>" --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check < /dev/null
# Naming a specific model explicitly (no live listing command exists to discover choices)
codex exec "<prompt>" -m gpt-5.5 --skip-git-repo-check < /dev/null
# Resume last session
codex exec resume --last "<follow-up>"
```
Drop `--skip-git-repo-check` when `$(pwd)` is actually a trusted/known git repo — it's only needed outside one.

Gotchas (verified live, superseding any earlier unverified guess): there is no `codex models` subcommand — see "Never hardcode model names" above. Outside a git repository, Codex refuses to run unless `--skip-git-repo-check` is passed. Codex also tries to read from stdin by default ("Reading additional input from stdin...") and will hang waiting for EOF if none is piped — always redirect `< /dev/null` (or pipe real stdin) when dispatching non-interactively. Default account model observed live: `gpt-5.5`.

## After running

- Summarize the output for the user; don't paste it raw.
- If the dispatch performed file edits (not just analysis), run `git diff --stat` and then the full `git diff` on the touched files, and show/summarize that diff to the user before treating the task as done. Whatever diff-review UI your host provides typically only covers edits made through its own file-editing tools — an external CLI editing files directly on disk bypasses that entirely, so this manual git-diff step is the only reliable way the user sees what changed.
- Treat the result as a peer opinion — the other model has its own cutoff and blind spots. If you disagree, say so with evidence (own knowledge or a web search) before relaying its claims.
- Offer to resume the same backend/model for follow-ups rather than starting fresh.
- When resuming to debate a disagreement, identify yourself clearly so the other agent knows it's a peer AI discussion.

## Multi-agent review / repo scans

When the user wants several backends to independently review the same diff or repository (not just one dispatch), fan out in parallel rather than sequentially:

1. Confirm which enabled backends to use (still respects `enabledTools` — don't propose a tool the user didn't select in setup). For opencode, run `opencode auth list` first and pick a `*-free` model unless a payment method is already confirmed.
2. Run each backend's model-listing command live where one exists and pick per the selection rules above (or per `autoMode`'s confirm-vs-auto behavior); for Codex, use its account default or a user-named model.
3. Launch each as its own background process with the *same* prompt/diff/repo context, started together so they run concurrently.
4. Once all finish, synthesize — don't just concatenate. Call out where reviewers agree (higher-confidence findings), where they disagree (needs your own judgment or a source check), and anything only one model caught.
5. Attribute each finding to its source model in the summary so the user can weigh vendor-specific blind spots.

## Common mistakes

- Forgetting to check `config.json` first and re-asking the setup question every time, or conversely assuming setup already happened without reading the file.
- Proposing a backend that isn't in the user's `enabledTools` without checking first.
- **Hardcoding a model id or provider prefix instead of running `opencode auth list` + `opencode models` (or `agy models`/`grok models`) live** — every one of these catalogs has been observed to drift, and opencode's provider prefix itself has already changed twice in this skill's lifetime (`minimax-coding-plan/...` → `openrouter/z-ai/...` split → unified `opencode/...` namespace). A prefix that worked last session may not exist at all anymore.
- **Defaulting to a paid opencode model without checking billing status** — a payment-method-less account fails with "No payment method", not a graceful downgrade; check first, or just default to a `*-free` model unless the task specifically needs paid-tier quality.
- **Assuming Codex has a `models` listing command** — it doesn't (verified live); omit `-m` for its account default or use a user-named model.
- **Dispatching to Codex without `--skip-git-repo-check` outside a git repo, or without redirecting stdin** — both cause a hang or an immediate refusal (verified live).
- Popping a confirmation question when `autoMode: true` — auto mode means pick via the live rule and just run, announcing the pick in one line instead.
- Forgetting `--add-dir "$(pwd)"` on an agy edit call — it silently edits agy's own scratch workspace instead of the real repo, with no error.
- Passing `--agent` to agy — no named agents are configured on this account; route by `--model` only.
- Running an opencode model without the idle-watchdog wrapper when it's a large or unfamiliar model (`--print-logs`, kill after 90s with no new log bytes) — it can hang indefinitely with no error, and a flat total-duration timeout would also kill legitimately-slow-but-progressing runs. On an idle-kill, fall back to a different (e.g. free-tier) model, don't retry the same one.
- **Dispatching to opencode without checking `opencode auth list` first** — a credential-less or plan-exhausted provider fails with an opaque, non-auth-looking error (`UnknownError`/`UnexpectedServerResponse`, "Token Plan usage limit reached", or "No payment method") rather than a clear "not logged in"/"billing needed" message (verified live).
- Accepting an external model's claims about recent releases/APIs uncritically.
- Asking the user for permission-mode approval — full access is pre-approved; just run.
- Running the agent as a foreground command that blocks the whole session for minutes — always background it (except the sub-second listing calls for agy/Grok/opencode).
- In a multi-agent scan, concatenating raw outputs instead of synthesizing agreements/disagreements/attribution.
- Assuming a malformed opencode error is a dispatch-skill bug — check `~/.config/opencode/opencode.json` for config shape issues (e.g. `mcp` not keyed by server name), `opencode auth list` for missing credentials, and the payment-method link in the error before assuming the CLI invocation itself is wrong.
