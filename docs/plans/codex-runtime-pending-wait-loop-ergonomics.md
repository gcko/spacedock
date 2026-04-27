---
id: 219
title: "Codex runtime: make pending wait loop ergonomic and handle-hidden"
status: backlog
source: "CL feedback during task 217 gate, 2026-04-27 - follow-up to task 216"
started:
completed:
verdict:
score: 0.64
worktree:
issue:
pr:
mod-block:
---

Task 216 landed the Codex preemptible-wait mitigation: when the first officer is blocked on a worker result, it should enter a preemptible `wait_agent` loop and resume that same wait after non-stopping captain input.

The current Codex runtime guidance still makes this too easy to execute incorrectly. Fresh dispatch has an explicit wait shape, but the critical-path reused-worker path is spread across separate bullets: mark the worker active again, remember that `send_input` is not completion evidence, and wait on the same handle. In practice, the first officer can emit an "active again" status, answer side discussion, and fail to immediately resume the wait.

The runtime should make the 216 behavior ergonomic and difficult to miss by treating critical-path waits as a small state machine with a canonical label-only operator banner.

Recommended direction:

- Record an internal `pending_wait` entry after every fresh `spawn_agent` that blocks the next step and after every critical-path `send_input`.
- Keep the runtime handle internal. Operator-facing status should lead with the FO-owned worker label and omit UUID handles unless debugging a tool failure.
- Emit one canonical wait banner before every blocking wait:

  ```text
  Waiting on `{worker_label}`. Esc/message interruption is safe; I’ll resume this same wait unless you say pause/stop.
  ```

- Immediately call `wait_agent` after that banner. An "active again" message is not a substitute for the wait banner.
- Before ending any turn while `pending_wait` exists, check whether the newest captain input paused/stopped the workflow or created a clarification blocker. If not, the final action must be resuming `wait_agent` on the same internal handle.
- Completion notifications are useful context only. They do not clear `pending_wait`; only `wait_agent` completion evidence does.

Ideation should include review by an AI engineer before gate presentation. The reviewer should specifically check whether the proposed Codex runtime wording is operationally crisp enough for an FO to follow under interruption, whether handle hiding preserves debuggability, and whether the test plan proves same-handle waits without requiring user-visible UUIDs.
