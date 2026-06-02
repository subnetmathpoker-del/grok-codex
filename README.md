# grok-codex

A reusable **Grok Build skill** for safely orchestrating the local OpenAI Codex CLI (`codex`).

This skill turns the original "Grok Build → Codex CLI" connectivity test into a first-class, permanent capability.

## What It Does

- **Connectivity Test**: Built-in verification using the exact commands from the original test (`which codex && codex --version` + safe `codex exec` status check). Prints `**SUCCESS: Grok Build can orchestrate Codex CLI**` on success.
- **Safe Delegation**: Delegate coding, research, or implementation tasks to Codex with strong guardrails. Grok always independently verifies results using its native tools (`list_dir`, `read_file`, git checks, etc.).
- **Super-Heavy Parallel Mode**: For complex tasks, run up to 12 parallel Codex (or sub-agent + Codex) attempts, compare results in a best-of-N table, and apply **only the winner** (or safe merge). Increases quality and accuracy.

**Grok stays in control as the orchestrator.** Codex is a powerful helper, never the boss.

## Installation

```bash
mkdir -p ~/.grok/skills/codex
cp SKILL.md ~/.grok/skills/codex/
```

Skills auto-reload when files change. You can also use `/skills reload` or restart Grok Build.

Alternatively, use the built-in creator:
```
/create-skill
```
(Provide a description like: "orchestrate local codex CLI with built-in exact connectivity test, safe exec, verification using Grok tools, and super-heavy up to 12 parallel subagents or codex variants following best-of-n patterns")

## Usage

See the [cheat sheet](#cheat-sheet) below for quick reference.

Basic examples:
- `/codex --check` — Run the canonical connectivity test
- `/codex list files here and report git status with no changes` — Safe read-only delegation
- `/codex --super-heavy 4 implement the new feature safely` — High-parallelism mode

The skill is also auto-triggered by natural language containing "codex", "use codex", "delegate to codex", "codex cli", etc.

## Cheat Sheet

### User Invocation Options

| Mode | Syntax / Trigger | Parallelism | Notes |
|------|------------------|-------------|-------|
| Connectivity Check | `/codex --check`<br>`run codex connectivity test`<br>`confirm grok can orchestrate codex cli` | 1 (test only) | Runs the **exact** original two commands. Emits SUCCESS banner + 3-bullet summary. |
| Normal Delegation | `/codex <prompt>`<br>or natural language ("use codex to...") | 1 | Runs connectivity first, one guarded `codex exec`, then Grok verifies. |
| Super-Heavy | `/codex --super-heavy 8 <task>`<br>`use codex super heavy with 6 agents`<br>`best of 12 with codex`<br>`parallel codex 4 ways` | Up to 12 (hard cap) | Launches N parallel Codex/sub-agent runs, evaluates with table, applies **only winner**. |

**Argument hint**: `[--check | --test-connectivity | --super-heavy N | <task prompt>]`

### Codex CLI Flags (used internally)

```bash
codex exec [--skip-git-repo-check] [-C /abs/dir] [--json] [--sandbox <policy>] "GUARDED PROMPT"
```

Common:
- `--skip-git-repo-check` — For non-git dirs (auto-added by skill when needed)
- `-C` — Change to specific directory
- `--json` — Structured output for parsing
- `--sandbox` — e.g. `workspace-write` or `read-only`

### Internal Parallel Agent Options (for heavy mode)

**Preferred (when spawn_subagent available):**
- `spawn_subagent` with:
  - `subagent_type: "general-purpose"`
  - `isolation: "worktree"` (recommended)
  - `background: true`
  - `description: "[codex-heavy-<i>] Candidate <i> of <N>"`
- Each sub-agent produces guarded codex prompt or runs it, writes JSON summary to `/tmp/grok-codex-heavy-<i>.json`, ends with `CANDIDATE <i> COMPLETE`

**Reliable Alternative (used in current env demos):**
- Direct `run_terminal_command` (background: true, description: "[codex-demo-<i>]") running variant `codex exec`
- Capture to `/tmp/grok-codex-heavy-<i>.log`

**Control Flow:**
- `wait_commands_or_subagents` (mode: "wait_all")
- `get_command_or_subagent_output` (block: true, timeout_ms)
- `kill_command_or_subagent` for stragglers
- Build best-of-n table → pick winner → apply only winner → re-verify with native tools → cleanup `/tmp/grok-codex-*`

### Key Tool References (exact names)
- `run_terminal_command` (for codex + verification; supports `background`, `description`, `timeout`)
- `spawn_subagent`
- `wait_commands_or_subagents`
- `get_command_or_subagent_output`
- `list_dir`, `read_file`, `search_replace`, etc. for verification

**Hard cap**: N ≤ 12. Defaults to 3-5 for vague "heavy".

## Safety & Principles

- Connectivity test is always first (unless "skip check").
- Every prompt to Codex includes guardrails: "no unrequested changes", "report all actions", "stay in scope".
- Grok **always** verifies results independently after Codex runs. Never trust Codex output alone.
- Explicit error handling for PATH, auth, git checks, timeouts, etc.
- Only the winner's changes are applied.

## Development / Contributing

The canonical source for the skill logic is `SKILL.md`.

To test:
- `/codex --check`
- A small safe delegation
- `"use codex in super heavy mode with 3 agents on a trivial read-only task"`

See the embedded Notes section in SKILL.md for maintenance (re-generate via `/create-skill` or `/skillify`).

## License

MIT (or your preferred — add LICENSE file if forking).

---

**Originally created after successful raw CLI connectivity test in Grok Build (June 2026).** The skill makes that confirmation a permanent, slash-commandable feature with optional super-heavy parallelism.
