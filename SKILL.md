---
name: codex
description: >
  Orchestrate the local OpenAI Codex CLI via `codex exec` for status checks, research, coding tasks, and implementation delegations. The skill always starts with (or can be explicitly invoked for) the built-in connectivity test using the exact commands: `which codex && codex --version` followed by a safe `codex exec "Current directory status check only..."`. Use when the user mentions codex, Codex CLI, "orchestrate codex", "delegate to codex", "codex exec", "run codex connectivity test", "confirm grok can orchestrate codex cli", runs /codex (or /codex --check, /codex check), or wants safe external CLI delegation with optional super-heavy parallel agents (up to 12). Also use for "best of N with codex", "super heavy 8 agents codex", etc.
when-to-use: "Use for Codex CLI connectivity verification (the exact two-command test), safe or full delegations to codex exec, or when the user wants Grok Build to orchestrate the local codex tool (including super heavy mode with multiple parallel agents). Triggers on /codex, codex test, run codex connectivity test, confirm you can reach codex cli, delegate to codex, best of with codex, super heavy codex, etc."
argument-hint: "[--check | --test-connectivity | --super-heavy N | <task prompt>]"
metadata:
  short-description: "Codex CLI orchestration, exact connectivity test, safe delegation, and super-heavy (up to 12) parallel subagent+codex mode for Grok Build"
---
# Codex — Local Codex CLI Orchestration Skill

Orchestrate the local Codex CLI (`codex` at ~/.local/bin/codex, codex-cli 0.135.0+) from within Grok Build sessions using `run_terminal_command` (the exact tool name; supports `background: true` for parallel work). This skill makes the original connectivity confirmation first-class, repeatable, and the foundation for all delegations. Grok always retains verification authority using its native tools (run_terminal_command, wait_commands_or_subagents, get_command_or_subagent_output, list_dir, read_file, etc.).

**You are the orchestrator.** Never pretend Codex output is authoritative without your independent verification. All `codex exec` invocations go through `run_terminal_command`. Subagents (via `spawn_subagent` if available in context, or directly via parallel background `run_terminal_command` as reliable alternative) are used only for true parallelism in super-heavy mode or optional verification.

## Usage
- `/codex` or `/codex <prompt>` — perform/confirm connectivity then safely delegate the prompt.
- `/codex --check` (or "run codex connectivity test", "confirm grok can orchestrate codex cli", "test codex connectivity") — run the **exact** two commands from the validated test and emit the 3-bullet report + **SUCCESS** banner on pass.
- `/codex --super-heavy 8 <task>` or "solve X using codex super heavy with 6 agents" or "best of 12 with codex" — high-parallelism mode (see Super-Heavy section).
- Auto-triggered on matching intent (e.g. "have codex implement the foo", "use codex for this research").

Always prefer invoking this skill over raw ad-hoc `run_terminal_command` calls to `codex`.

## Core Principles (never violate)
1. **Connectivity first**: For any delegation in a new session/context, or on explicit --check, run the exact connectivity test first (Step 1). Only skip if user says "skip check" or "assume connected" and a recent success is in context.
2. **Guard every prompt to codex**: Embed "status check only / no unrequested changes", "report every action and file touched", "stay in the specified directory".
3. **You (Grok) verify everything**: After any `codex exec`, use `list_dir`, `read_file`, `run_terminal_command` (git status, git diff, ls, etc.) to observe effects yourself. Do not report Codex claims as fact until verified.
4. **Error handling is explicit**: Surface PATH, auth, --skip-git-repo-check needs, trust failures, internal Codex errors verbatim.
5. **Tool names**: Always reference and invoke using the exact available tool names: `run_terminal_command` (for shell/codex; use background:true + description for parallel), `spawn_subagent` (preferred when available for general-purpose worktree-isolated subagents), `get_command_or_subagent_output`, `wait_commands_or_subagents`, `read_file`, `list_dir`, `search_replace`, `write` (for temp artifacts), `kill_command_or_subagent` as needed. (Note: depending on your environment, parallel background `run_terminal_command` + wait/get is often the reliable, always-usable mechanism for super-heavy even if `spawn_subagent` tool is not directly exposed.)
6. **Super-heavy discipline**: When parallelism requested, cap at 12, use worktree isolation + background where appropriate, evaluate like best-of-n with tables, apply **only** the winner (or safe merge of non-conflicting parts).

## Step 1: Connectivity Verification (first action; repeatable via --check)
This is the canonical test. Execute **exactly** these two commands (in order) via `run_terminal_command`. Use the current working directory for the first attempt.

1. `which codex && codex --version`
   - Capture full stdout/stderr + exit code.
   - Expect path like `~/.local/bin/codex` and version string containing "codex-cli 0.135.0" (or newer).

2. `codex exec "Current directory status check only. List files in root and confirm whether .git exists or not. No changes allowed. Report all actions."`
   - If this fails with git-repo/trust or "not a git repo" style error, immediately retry the identical prompt but with `--skip-git-repo-check` prepended to the codex command:
     `codex exec --skip-git-repo-check "Current directory status check only. ..."`
   - You may also use `-C <dir>` if you want to test a specific known git dir (e.g. /path/to/your/git/repo) as part of a thorough check.
   - Capture full output. Look for Codex status footer (version, workdir, model, sandbox, session), execution trace (ls/pwd/test -e .git), truthful .git report, and "No changes were made."

**Reporting rules for this step (always produce)**:
- On full success (both commands exit 0, correct path/version, Codex produced recognizable status + correct dir/.git report + "No changes were made." or equivalent):
  Emit exactly:
  ```
  SUCCESS: Grok Build can orchestrate Codex CLI
  ```
  Follow with 3-bullet summary:
  - Codex found at: <path> (version: <ver>)
  - Connectivity test: passed in <cwd> ( .git present? <yes/no> ; used --skip-git-repo-check? <yes/no>)
  - Codex response excerpt: <key 1-2 lines including dir/.git/no-changes>
- On any failure or partial:
  Report exactly 3 bullets:
  - Was `codex` found and executable? (path or "not found")
  - Any errors (PATH/auth/trust/git-check/internal Codex errors)? Quote relevant stderr/stdout.
  - Did Codex respond successfully? (include key excerpts + full relevant output if short)
- Non-fatal internal Codex errors (e.g. model refresh, path warnings) are logged but do not fail the banner if the core exec + "No changes" succeeded.
- Always state the actual cwd used for the exec.

This step can be the entire run (`/codex --check`). For other uses, run it first unless explicitly waived, then proceed.

## Step 2: Safe Single Delegation (normal mode)
After a successful (or recently confirmed) connectivity:

1. Craft the prompt for Codex:
   - Start with the user's exact request.
   - Prefix/append strong guardrails:
     "You are Codex running via `codex exec` invoked by Grok Build orchestrator. Perform ONLY the requested work in the current (or -C specified) directory. Report every file you read, every terminal command you execute internally, and every change (or 'no changes'). At the end, include the output of `git status --short`, `git diff --name-only`, and a directory listing. No unrequested changes to files, git, or environment."
   - For pure status/read-only: use the exact "Current directory status check only. List files... No changes allowed. Report all actions." language.
   - Scope explicitly: "Operate only in <dir>. Do not touch parent or sibling dirs unless the request says so."

2. Invoke via `run_terminal_command`:
   ```
   codex exec [--skip-git-repo-check] [-C /absolute/target/dir] [--json] [--sandbox <policy>] "FULL PROMPT"
   ```
   - Add `--skip-git-repo-check` when the target dir lacks .git or when previous test showed it was needed.
   - Use `-C` for explicit target (absolute path preferred).
   - `--json` when you want machine-parseable events (rare for normal mode).
   - Set `description` on the tool call, e.g. "Codex exec: research current dir state safely". For heavy use descriptions like "[codex-demo-N]".
   - Choose `timeout`: 30000-60000 for status/checks; 120000+ or `background: true` for real work. For background, immediately follow with `get_command_or_subagent_output` (block=true, appropriate timeout) or `wait_commands_or_subagents`.
   - Capture stdout/stderr/exit code fully.

3. Parse Codex response:
   - Extract its status footer.
   - Extract its claimed actions, files touched, and final "No changes" or diff summary.
   - Note any Codex-internal session id or usage info.

4. Independent verification (mandatory):
   - `list_dir` (target dir).
   - `read_file` on every file Codex claims it created or edited (at least the key ones; full for small files).
   - `run_terminal_command` for: `git status --short`, `git diff --name-only`, `git diff` (or `ls -la` + `find . -type f` if no git), `pwd`.
   - Cross-check: do the observed files/changes exactly match what was requested and what Codex reported? Any surprises (extra files, deletions, outside-scope edits)?
   - If mismatch or unwanted changes: surface verbatim, then decide (revert via terminal/git, or use `search_replace` yourself to correct, or ask user). Codex cannot be the final source of truth.

5. Report: excerpts from Codex + your verification output + net effect ("X files changed as requested", "No changes made", etc.).

Use `background: true` + wait tools for long delegations so you can report progress.

## Step 3: Super-Heavy / High-Parallelism Mode (N up to 12)
Activate when user says phrases like "super heavy 8 agents", "best of 12 with codex", "parallel codex 6 ways", "use codex with best-of-n 5", etc. Or when task is complex and user requests maximum exploration.

1. Confirm connectivity (Step 1) or note recent success.

2. Parse N from request (first number 2-12 if present; default 3-5 for vague "heavy"; hard cap at 12). Tell user the N you will use and the cap.

3. Generate N diverse candidates in parallel (single turn, multiple tool calls):
   - Preferred: Use `spawn_subagent` (subagent_type: "general-purpose", isolation: "worktree" if the task may produce file changes, background: true, description: "[codex-heavy-<i>] Candidate <i> of <N>").
     Prompt for each subagent: the original task + "You are candidate <i> of <N>. Produce a self-contained, guarded prompt suitable for `codex exec` (or a variant decomposition). Then either (a) run the codex exec yourself inside your worktree using the available tools, or (b) output the exact codex command + prompt you recommend. In all cases, at end write a JSON summary to /tmp/grok-codex-heavy-<i>.json with keys: {approach, codex_prompt_used, observed_changes, confidence, rationale}. Also write a short .md summary. End your output with CANDIDATE <i> COMPLETE."
   - Alternative/hybrid: Directly launch N `run_terminal_command` (background:true, description e.g. "[codex-demo-<i>]") with each running a variant `codex exec` (different prompt phrasings, ...). Capture to /tmp/grok-codex-heavy-<i>.log . (This alternative is reliable when spawn_subagent not directly callable as a tool.)
   - Mix is allowed: some subagents, some direct codex.

4. Wait for all:
   - Use `wait_commands_or_subagents` (mode: "wait_all", with list of task_ids) or loop `get_command_or_subagent_output(task_id=..., block=true, timeout_ms=300000)` (5 min generous per candidate).
   - If any times out, `kill_command_or_subagent` the stragglers and continue with what you have.

5. Collect & evaluate (you do the synthesis; optionally spawn 1 lightweight reviewer subagent for the comparison):
   - For each candidate: `read_file` its /tmp/grok-codex-heavy-*.json or .md or .log (resolve paths from worktree if subagent used; subagent result usually includes worktree_path).
   - Build a comparison table (like best-of-n):
     | Candidate | Approach | Correctness (solves request?) | Safety (no extra changes, no breakage) | Fidelity to guardrails | Quality/Completeness | Confidence (self) | Notes |
   - Detailed findings per candidate (positive + defects).
   - Criteria order: Correctness > Safety/Guardrail adherence > Completeness > Explanation quality. Prefer the one that actually delivered verified correct work with least side effects.

6. Decide & apply only the best:
   - Pick single winner (or 1-2 non-conflicting winners to merge).
   - Apply: Either (a) craft a final "apply only the approach from candidate K: <summary of winner>" guarded prompt and run one last `codex exec` (in main workspace or chosen worktree), or (b) read the winner's artifacts and apply yourself with `search_replace` + terminal (preferred for precision).
   - Re-verify the final state with list_dir/read_file/git via `run_terminal_command`.
   - Never apply more than the chosen winner.

7. Report: full table, key excerpts from winner + verification, "WINNER: <i>", any SUCCESS banner, and net changes.

For simple tasks, refuse super-heavy and fall back to single delegation (tell user why).

## Error Handling
- `which codex` fails or path wrong → report "Codex not found in PATH. Expected ~/.local/bin/codex. Run the install steps or check PATH."
- `codex --version` or exec fails with auth → "Codex reports not logged in. Run `codex login` (or check ~/.codex/auth.json)."
- Git/trust check failures → retry with `--skip-git-repo-check` as documented; surface the exact error Codex gave.
- Internal Codex non-fatal errors (timeouts, "skills paths", model refresh) → log them, continue if core task output + "No changes" appeared.
- Subagent spawn failures in heavy mode → log, reduce N, continue with remaining candidates.
- Timeout on wait → kill, report partial results, suggest re-run with smaller N or longer timeout.
- Codex claims changes but `git status` / `list_dir` / `read_file` show none (or wrong) → treat as failure; surface both Codex output and your probe output; do not claim success.
- Always use absolute paths for any /tmp/grok-codex-* artifacts you or subagents create.

## In-Progress Reporting
- Connectivity start: "Running canonical Codex connectivity test (which + version; safe exec in cwd)..."
- After Step 1 success: "Codex connectivity: SUCCESS ... (details)"
- After Step 1 fail: the 3-bullet failure block.
- Launch single: "Delegating to codex exec (description: ...). Awaiting (timeout ~Xs)..."
- Launch heavy: "Launching N=<N> super-heavy candidates (mix of spawn_subagent if avail + direct background run_terminal_command codex exec variants)..."
- After wait: "All candidates complete. Reading artifacts and building evaluation table..."
- After verification: "Verification complete via list_dir + read_file + git probes. Observed: <summary>."
- After apply (heavy): "Applied winner only. Re-verified. WINNER: <i>"

## Final Report Structure
Always end with a clear block containing:
- Connectivity status (re-state SUCCESS banner if the test ran this turn).
- Mode: single delegation or super-heavy (N=...).
- Task summary.
- Exact command(s) or prompt(s) sent to codex (abbreviated if long).
- Codex output excerpt (footer + result claim).
- Your independent verification results (git status --short, changed files list, sample read_file content or diff).
- Net effect ("N files created/modified as requested", "No filesystem changes", "Reverted unwanted X").
- If heavy: the full comparison table + "WINNER: <i> (rationale)" + which candidate's changes (or merge) were kept.
- Any temps left for user inspection (rare).
- Suggested next steps.

## Examples
1. Connectivity:
   User: "run codex connectivity test" or "confirm you can orchestrate the codex cli"
   → Step 1 exact commands (with --skip if needed in <cwd> and/or /path/to/your/git/repo) → 3-bullet + **SUCCESS: Grok Build can orchestrate Codex CLI**

2. Safe delegation (status):
   User: "/codex list the files here and report git status with no changes"
   → Connectivity (or recent) → codex exec with exact guardrail prompt → your list_dir + run_terminal_command "git status --short" + read any touched files → report "Matches Codex claim. No changes made."

3. Safe delegation (work):
   User: "/codex add a one-line comment to foo.py explaining its purpose"
   → Connectivity → guarded prompt with "report all..." → codex exec → your read_file foo.py + git diff → "Verified: only the requested comment added at line 12."

4. Super heavy:
   User: "use codex in super heavy mode with 4 agents to research best way to X and implement the winner"
   → Connectivity → spawn 4 (or direct background run_terminal_command variants) → wait_commands_or_subagents → table evaluation → apply only winner via final codex or direct edits + re-verify → full table + WINNER + SUCCESS if test ran + final state.

## Cleanup
At end of any turn using this skill:
```bash
# via run_terminal_command tool:
run_terminal_command command="rm -f /tmp/grok-codex-* /tmp/grok-codex-heavy-* 2>/dev/null || true" description="Cleanup temp codex heavy artifacts (leave real ~/.codex alone)"
```
(Do not clean the real ~/.codex/ or user files.)

## Notes (for skill maintenance / /create-skill users)
- This skill was created via direct mkdir + search_replace (empty old_string) on ~/.grok/skills/codex/SKILL.md per the pattern in .grok/skills/create-skill/SKILL.md and 08-skills.md. It can also be (re)generated by running /create-skill (or /skillify) with a description like "orchestrate local codex CLI with built-in exact connectivity test, safe exec, verification using Grok tools, and super-heavy up to 12 parallel subagents or codex variants following best-of-n patterns".
- During /create-skill interview: choose User scope (~/.grok/skills) for personal reuse across all projects; supply the exact test commands and heavy-mode requirements from the original confirmation.
- Keep the connectivity test commands byte-for-byte stable as the "source of truth" for this capability.
- Future extensions: add support for specific codex flags the user discovers, or helper scripts under scripts/ only if prompt-only becomes insufficient.
- Test after install: `/codex --check`, a small safe delegation, and a "super heavy 2" (or 3) run on a trivial task; confirm SUCCESS banner and that only winner changes were applied.

This turns the original one-off Codex orchestration confirmation into a permanent, slash-commandable, auto-discoverable, high-parallelism-capable skill.
