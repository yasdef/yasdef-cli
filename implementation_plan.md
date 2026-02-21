# Codex PTY Wrapper (Interactive) - Implementation Plan

## Context / Problem
`ai/scripts/orchestrator.sh` runs multi-phase automation (planning -> implementation -> review) by spawning the Codex **interactive** TUI. The TUI is designed to remain open after completing a task, so the orchestrator blocks until the user manually exits (today: `Ctrl+C` per phase).

Goal: keep the interactive Codex TUI experience for planning/review, but make each phase **auto-terminate** when the phase is done, so the orchestrator can proceed without manual `Ctrl+C`.

## Goals (Acceptance Criteria)
- Preserve an interactive Codex TUI for the user during each phase (keystrokes + live output).
- Automatically end the Codex session when a phase-completion rule triggers.
- Restore terminal state cleanly on exit (no broken raw mode, no stuck cursor).
- Work with current `codex-cli` behavior (interactive default) without requiring upstream changes.
- Provide a deterministic exit code so the orchestrator can proceed/fail reliably.

## Non-Goals
- Replacing Codex with a custom chat UI (that is a separate option, not this wrapper).
- Reliably parsing Codex’s ANSI-heavy TUI output for semantic states (avoid if possible).
- Changing existing `ai/scripts/*` right now; this plan may propose changes, but implementation can start with a wrapper-only approach.

## Key Design Idea
Create a small executable (`ai/wrapper/codex_pty_wrapper`) that:
1. Spawns the real `codex ...` interactive CLI under a pseudo-terminal (PTY).
2. Proxies stdin/stdout between the user terminal and the Codex PTY (so it looks and feels like normal Codex).
3. Runs a watcher loop that detects phase completion (file-based sentinel or artifact-based rule).
4. When completion triggers, gracefully terminates Codex (preferred: send `/exit\\n` if supported; fallback: send SIGINT), then exits itself.

This makes the wrapper the “session owner”, so it can decide when to end the phase.

## Decisions Needed (Tracked)
See `ai/wrapper/decisions.md` for open decisions (stack, completion signals, integration approach).

## Proposed Completion Signals
### Planning (strong, artifact-based; no prompt changes required)
Trigger when a step plan has been produced/updated:
- Condition: a file matching `ai/step_plans/step-*.md` has `mtime > phase_start_time`
- And contains: `AI_RUN_COMMAND_VERSION: 2` (or the `## User Command (manual only...)` block)

This aligns with how the existing planning flow writes step plans.

### Review (choose one; easiest if we allow a sentinel file)
Review is often “analysis-only” and may not write files, so artifact detection can be ambiguous. Options:
1. Sentinel file: require the review prompt (or operator action) to create `ai/tmp/phase_done/review.<run_id>` as the last step.
2. Output marker: update the review prompt to print a final line like `PHASE_DONE: review` (wrapper watches child output stream).
3. Explicit operator command (fallback): wrapper intercepts a reserved input command like `/phase-done` and exits the phase without forwarding it to Codex.

Recommendation: implement (3) as a safety fallback, and pick either (1) or (2) as the primary automation mechanism.

## Wrapper CLI Contract (Proposed)
Design the wrapper to be a drop-in replacement for `codex` in `ai/setup/models.md`.

### Invocation
- Pass-through mode by default:
  - `codex_pty_wrapper [--phase <name>] [--done-rule <rule>] -- <codex args...>`
- Examples:
  - Planning:
    - `codex_pty_wrapper --phase planning --done-rule step-plan-v2 -- codex -m gpt-5.2 --config model_reasoning_effort='high' "run ai/scripts/ai_plan.sh"`
  - Review:
    - `codex_pty_wrapper --phase review --done-rule sentinel-file --done-file ai/tmp/phase_done/review.<run_id> -- codex -m gpt-5.2 "run ai/prompts/review_prompts/..."`

### Core flags
- `--phase <planning|implementation|review|custom>`
- `--done-rule <step-plan-v2|sentinel-file|output-regex|manual-only>`
- `--done-file <path>` (for `sentinel-file`)
- `--output-regex <regex>` (for `output-regex`)
- `--grace-period-ms <N>`: after rule triggers, wait N ms with no new output before terminating (prevents cutting off final render)
- `--kill-method <exit-command|sigint>`: prefer `/exit` first, fallback to SIGINT
- `--idle-timeout-s <N>` (optional): if no output for N seconds after rule is armed, terminate (guardrail)

## Architecture / Flow
### PTY Proxy
- Create PTY pair (master/slave).
- Spawn child process `codex ...`:
  - Child stdio attaches to PTY slave.
  - Child becomes session leader / controlling terminal (platform-specific setup).
- Parent:
  - Put user terminal into raw mode (so keystrokes like arrows are forwarded).
  - Bidirectional copy loops:
    - User stdin -> PTY master
    - PTY master -> user stdout (and optional log file)
  - Signal handling:
    - SIGWINCH: propagate terminal size to PTY via `ioctl(TIOCSWINSZ)` so Codex renders correctly.
    - SIGINT: forward to child (and also restore terminal on exit).
    - Exit/cleanup: always restore termios.

### Watcher Loop
Run concurrently (thread/goroutine/async task):
- Capture `phase_start_time` (monotonic and wall clock for `mtime` checks).
- If `done-rule=step-plan-v2`:
  - Poll `ai/step_plans/step-*.md` and test the completion conditions.
- If `done-rule=sentinel-file`:
  - Check `--done-file` exists and `mtime > phase_start_time`.
- If `done-rule=output-regex`:
  - Scan the output byte stream (line-buffered best-effort; tolerate ANSI).
- If manual fallback enabled:
  - Intercept `/phase-done` input line before forwarding to child; trigger completion.

On trigger:
- Mark completion “armed” and record trigger time.
- Wait `--grace-period-ms` with no further output (or a fixed small delay).
- Terminate child using configured method:
  - Send `/exit\n` via PTY input (best-effort).
  - If child still alive after N ms, send SIGINT.
  - If still alive, SIGTERM then SIGKILL as last resort (configurable).
- Exit wrapper with child exit code if meaningful; otherwise `0` on successful phase completion.

## Integration With Existing Orchestrator
You have three viable integration patterns:

### A) Change `ai/setup/models.md` command to wrapper (simplest)
- Replace `planning | codex | ...` with `planning | ai/wrapper/codex_pty_wrapper | ...` (or a relative path).
- Risk: `ai/scripts/orchestrator.sh` currently uses `script -q` only when the first arg is literally `codex`. If it falls back to `tee`, the wrapper must still behave well when its stdout is piped (it can; Codex itself still sees a PTY).

### B) Teach `ai/scripts/orchestrator.sh` to treat wrapper like Codex
- Update `run_with_output_log` to use `script -q` when command is `codex` OR `codex_pty_wrapper`.
- Most reliable UX; small targeted change. (Defer until after wrapper prototype works.)

### C) PATH shadowing (avoid changing models/orchestrator)
- Put the wrapper earlier in PATH with the name `codex` so orchestrator still thinks it is running Codex.
- Least desirable: hidden behavior and potential to confuse other tools.

Recommendation: start with (A) to prototype; move to (B) if any TTY edge cases appear.

## Implementation Steps (Milestones)
1. **Pick stack + library** (see decisions).
2. **Prototype PTY proxy**:
   - Spawn `codex --version` and a trivial interactive program to validate raw-mode + resize + cleanup.
   - Verify it works when wrapper stdout is piped through `tee`.
3. **Integrate Codex spawn**:
   - Pass through all arguments after `--`.
   - Ensure terminal restoration on crashes (`try/finally` / `defer`).
4. **Add completion rules**:
   - Implement `step-plan-v2` watcher.
   - Implement `manual-only` escape hatch (`/phase-done`).
5. **Add review completion mechanism**:
   - Choose sentinel file or output marker approach.
   - If sentinel/output marker requires prompt changes, plan a small change to the review prompt generator (later step).
6. **Add hardening**:
   - Grace period, idle timeout, kill escalation ladder.
   - Robust signal handling (SIGWINCH, SIGINT, SIGTERM).
7. **Integrate with orchestrator**:
   - Update `ai/setup/models.md` command OR update orchestrator’s logging behavior per chosen approach.
8. **Testing / Validation**:
   - Manual acceptance tests (primary).
   - Unit tests for file-based rules (parsing + `mtime` logic) if stack supports easy tests.
9. **Docs**:
   - `ai/wrapper/README.md` usage, flags, and “how to end phase manually”.

## Manual Acceptance Tests (Suggested)
- Planning:
  - Start orchestrator planning phase and confirm:
    - Wrapper shows Codex TUI normally.
    - After the plan file is updated with `AI_RUN_COMMAND_VERSION: 2`, wrapper exits without user action.
    - Orchestrator proceeds to next phase.
- Review:
  - Confirm chosen completion mechanism triggers exit.
  - Confirm `/phase-done` always ends the phase, without sending that text into Codex.
- Robustness:
  - Resize terminal during run; Codex UI resizes correctly.
  - Force kill wrapper (`Ctrl+C`) mid-run; terminal is restored.

## Risks / Edge Cases
- Codex UI output contains heavy ANSI; output-regex detection may be brittle.
- Some environments need correct controlling-terminal semantics; implement proper session/pty setup.
- “Done” signals must be run-specific to avoid stale-file false positives (use `phase_start_time` and/or `run_id`).
- Terminating too fast can truncate last render; use grace period + idle detection.

## Rollout Strategy
1. Land wrapper and documentation first (no orchestrator changes required to start testing manually).
2. Integrate only planning phase first (strong artifact-based done condition).
3. Add review phase completion mechanism once proven safe.

