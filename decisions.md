# Codex PTY Wrapper - Decisions

This file lists the concrete decisions needed before (or during) implementation of the interactive PTY wrapper.

## D1: Implementation Stack (Language + PTY Library)
**Status:** Decided

**Decision:** Go (PTY via `github.com/creack/pty` unless we later decide to avoid third-party deps)

Options:
- **Python 3 (stdlib `pty` + `termios`)**
  - Pros: no new dependencies; easy to ship as a single script; good on macOS/Linux.
  - Cons: careful work needed for raw mode, resize ioctls, and non-blocking IO; testing PTY logic is trickier.
- **Go (`github.com/creack/pty` + stdlib)**
  - Pros: produces a single static-ish binary; excellent concurrency; robust PTY handling patterns.
  - Cons: introduces Go toolchain requirement (and likely a new dependency).
- **Node.js (`node-pty`)**
  - Pros: very good PTY ergonomics; easy to build a small CLI.
  - Cons: adds npm dependency tree; native module build concerns; heavier runtime footprint.
- **Rust (`nix`/`pty` crates)**
  - Pros: strong correctness; good binaries.
  - Cons: most work; adds toolchain + dependencies.

Notes:
- If we use `github.com/creack/pty`, we will add a Go module under `ai/wrapper/` for the wrapper only.
- If you want zero third-party deps, we can use `golang.org/x/sys/unix` (still a module dep) or raw syscalls; this is a separate decision during implementation.

## D2: Orchestrator Integration Approach
**Status:** Decided

**Decision:** A: Update `ai/setup/models.md` to use the wrapper as the phase command

Options:
- **A: Update `ai/setup/models.md` to use wrapper as command**
  - Fastest to adopt; may run under `tee` logging (no `script`), which is usually OK with a wrapper-managed PTY.
- **B: Update `ai/scripts/orchestrator.sh` to treat wrapper like `codex` for `script -q`**
  - Best UX/logging reliability; requires a small change to orchestrator logic.
- **C: PATH shadowing (wrapper named `codex`)**
  - Avoids changing models/orchestrator, but is implicit and risky.

Notes:
- This aligns with the plan’s “A) Change `ai/setup/models.md` command to wrapper” integration pattern.

## D3: Phase Completion Signals
**Status:** Decided

**Decision:**
- Planning: artifact-based `step-plan-v2` detection (updated `ai/step_plans/step-*.md` containing `AI_RUN_COMMAND_VERSION: 2`)
- Review: sentinel file (recommended: `ai/tmp/phase_done/review.<run_id>` with `mtime > phase_start_time`)
- Fallback: manual `/phase-done` (always enabled)

Planning (recommended):
- Use **artifact-based** detection: updated `ai/step_plans/step-*.md` contains `AI_RUN_COMMAND_VERSION: 2`.

Review (choose one primary):
- **Sentinel file**: `ai/tmp/phase_done/review.<run_id>` exists with `mtime > phase_start_time`.
- **Output marker**: prompt guarantees a final line `PHASE_DONE: review` (wrapper scans output).
- **Manual-only**: user types `/phase-done` to end the phase (always keep as fallback).

Notes:
- This aligns with `ai/wrapper/implementation_plan.md` review-signal options and keeps “manual-only” as a safety hatch.

## D4: Termination Method (How Wrapper Exits Codex)
**Status:** Pending

Options:
- Send `/exit\n` into Codex input (if supported)
- Send SIGINT to child (equivalent to `Ctrl+C`)
- Escalation ladder: `/exit` -> SIGINT -> SIGTERM -> SIGKILL

Recommendation: implement the ladder; default to `/exit` then SIGINT.

## D5: UI Mode
**Status:** Pending

Options:
- Default Codex behavior (alt-screen)
- Force `--no-alt-screen` for simpler logging and fewer terminal edge cases

Recommendation: keep default, but allow a wrapper flag to add `--no-alt-screen` to the child `codex` command.

## D6: Scope / Platform Support
**Status:** Pending

Options:
- macOS only (your current environment)
- macOS + Linux

Recommendation: macOS + Linux (PTY logic is similar), but validate on macOS first.
