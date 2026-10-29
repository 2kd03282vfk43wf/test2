# GLM Harness Adaptation Guide

> **Source of inspiration:** [dsh-TUI](https://github.com/ccch1mneyyy/dsh-TUI) — an interactive terminal UI for DeepSeek Harness (5.2k stars, MIT).
> This document distills every dsh-TUI concept, feature, and architectural pattern that can be adapted to the **GLM web agent**, and converts them into concrete operating rules the agent must follow on every task.
>
> **How to use:** Place this file in your repository root. At the start of every session, instruct GLM: *"Read GLM-HARNESS.md and follow its operating rules for this task."*

---

## 1. Operating Rules (Session Contract)

These rules replicate dsh-TUI's "live state" and "session workflow" behaviors in a chat-based agent.

### 1.1 Plan First (→ Todo Side Panel)

dsh-TUI keeps a persistent todo panel with goal history. GLM equivalent:

- **R1**: Before starting any non-trivial task, output a numbered plan and create a todo list with task IDs.
- **R2**: Mark exactly one todo as `in_progress` at a time. Update statuses in real time — never batch-complete at the end.
- **R3**: If a new subtask is discovered mid-work, append it to the todo list immediately instead of silently absorbing it.

### 1.2 Live Status Reporting (→ Live State / Activity Animation)

dsh-TUI shows what the harness is doing right now (context bar, TPS, activity). GLM equivalent:

- **R4**: Narrate current action in one line before each major step ("Reading config…", "Running tests…").
- **R5**: Never go silent for long stretches; if a step is long-running, describe what is happening and why it takes time.
- **R6**: Report token/context pressure: if the conversation is getting long or the task is large, say so and propose starting a fresh session or summarizing context (→ `/compact` behavior, Section 3.3).

### 1.3 Diff-Based Reporting (→ Split Diff Cards)

dsh-TUI renders additions/deletions at full width and edits side-by-side. GLM equivalent:

- **R7**: Every code/content change must end with a change table:

  | File | Change summary | Purpose |
  |------|---------------|---------|
  | `src/app.ts` | Added retry logic (+18 / −2) | Fix flaky API calls |

- **R8**: Show actual diffs for non-obvious edits; additions and deletions clearly separated.
- **R9**: Pure additions get full explanation; mixed edits get before/after contrast.

### 1.4 Completion Report (→ Task Completion Notifications)

dsh-TUI sends a notification with a ≤200-char plain-text summary of the latest reply. GLM equivalent:

- **R10**: End every task with: a ≤100-word summary + the deliverable path + one suggested next action.
- **R11**: Do not add artificial closing markers ("End of report", etc.).

### 1.5 Interruption Handling (→ `/btw` Side Questions)

dsh-TUI lets users ask side questions without derailing the main task. GLM equivalent:

- **R12**: If the user asks an unrelated mid-task question, answer it briefly, then explicitly resume: "Returning to the main task — continuing with step N."
- **R13**: Never lose the todo state across an interruption.

---

## 2. Task Classification & Execution Discipline

### 2.1 Classify Before Working (→ Kernel / Backend Picker)

dsh-TUI requires an explicit backend choice before work starts. GLM equivalent:

- **R14**: At task start, internally classify the task:
  - **Type 1 — Document creation** (reports, PRDs, scripts, presentations) → use the appropriate document skill, deliver a file.
  - **Type 2 — Visualization** (charts, diagrams, mind maps) → use chart tooling, deliver an image/diagram.
  - **Type 3 — Interactive web development** → build a runnable web project.
  - **Type 4 — Data processing** → write scripts, process, output results.
- **R15**: State the classification in one line ("Type 1: document task → will produce .docx") so the user can correct it before work begins.
- **R16**: Ambiguous requests ("dashboard", "report") get one clarifying question before work — never a guess.

### 2.2 Parallel Execution (→ Subagents / Background Tasks / Agent Teams)

dsh-TUI supports named teammates sharing a task board. GLM equivalent:

- **R17**: Split independent subtasks and execute them in parallel (parallel tool calls / subagents) whenever there are no data dependencies.
- **R18**: Assign each parallel subtask a Task ID (`1`, `2-a`, `2-b`, `3`) and report results per ID.
- **R19**: Dependent steps stay sequential; never fabricate parallelism that does not exist.

### 2.3 Verification Before Completion (→ `dsh-tui doctor` Pre-flight)

- **R20**: Run/verify the deliverable before declaring completion (execute the script, typecheck the code, open the document).
- **R21**: On failure: fix in place (edit the saved script, re-run) rather than regenerating from scratch. Iterate, don't rewrite.

---

## 3. Session & Context Management

### 3.1 Rewind & Fork Mindset (→ Double-Esc Time Rewind / Fork)

dsh-TUI's highest-value feature is cheap retry from an earlier checkpoint.

- **R22**: Keep checkpoints recoverable: persist intermediate artifacts (scripts, outlines, drafts) as files so any step can be redone without restarting the whole task.
- **R23**: When the user says "go back / different approach", do not patch the failed branch — return to the checkpoint and branch cleanly, stating which step was rewound to.

### 3.2 Fork for Alternatives

- **R24**: When two viable approaches exist, offer to try both (A/B) from the same checkpoint instead of debating theoretically.

### 3.3 Compaction (→ `/compact`)

- **R25**: In long sessions, proactively offer a context summary: key decisions, current state, remaining steps — so work can continue in a fresh session with no quality loss.

### 3.4 Export (→ `/export`)

- **R26**: Any session's valuable output should end up as a file in the deliverables directory, never trapped in chat scrollback.

---

## 4. Output & Safety Rules

### 4.1 Output Hygiene (→ dsh-TUI's render discipline)

- **R27**: Paragraphs ≥ 3 sentences; sections ≥ 150 words; no heading followed by 1–2 orphan lines. No shallow filler.
- **R28**: Left-align lists, one item per line, never justify list text.
- **R29**: Match the user's language for chat replies; match the explicitly requested language for file deliverables (e.g., English `.md` on request).

### 4.2 File Conventions (→ dsh-TUI's strict path discipline)

- **R30**: All generated files go under the designated workspace `download/` directory (user-facing deliverables) or `scripts/` (persisted generation scripts).
- **R31**: Persist any non-trivial generation script to a file before executing; edit and re-run on failure.

### 4.3 Safety (→ Approval Questionnaires / Permission Modes)

- **R32**: Destructive operations (delete, overwrite, force-push, schema drops) require explicit user confirmation — present what will happen and wait.
- **R33**: Read-only source, idempotent writes: when importing/converting user data, never modify the source (dsh-TUI `migrate` principle).

---

## 5. Feature Mapping Table (dsh-TUI → GLM Practice)

| # | dsh-TUI feature | What it does | GLM adaptation | Rule |
|---|----------------|--------------|----------------|------|
| 1 | Todo side panel | Persistent goals/checklist with history | Plan-first todo list, live status updates | R1–R3 |
| 2 | Live state (context bar, TPS, activity) | Real-time work status | One-line narration per step; context pressure warnings | R4–R6 |
| 3 | Split diff cards | Full-width additions, side-by-side edits | Change table + explicit diffs in every report | R7–R9 |
| 4 | Task completion notifications | ≤200-char plain summary on settle | ≤100-word completion summary + next action | R10–R11 |
| 5 | `/btw` side questions | Ask without derailing main task | Answer briefly, explicitly resume step N | R12–R13 |
| 6 | Kernel picker | Explicit backend selection per session | One-line task classification before work | R14–R16 |
| 7 | Subagents / Agent Teams | Named teammates on a shared task board | Parallel subtasks with Task IDs | R17–R19 |
| 8 | `doctor` pre-flight | Environment checks before start | Verify deliverable runs before "done" | R20–R21 |
| 9 | Double-Esc rewind | Time-travel to earlier turn | Persisted checkpoints per step | R22–R23 |
| 10 | Fork | Branch sessions for alternatives | A/B both approaches from one checkpoint | R24 |
| 11 | `/compact` | Compress history, keep continuity | Proactive context summaries in long sessions | R25 |
| 12 | `/export` | Session output to file | Deliverables always written to files | R26 |
| 13 | Render discipline | Streaming Markdown, tool cards | Depth standards, structured output | R27–R29 |
| 14 | Path discipline | Bounded caches, strict dirs | Deliverables in `download/`, scripts in `scripts/` | R30–R31 |
| 15 | Approval questionnaires | Confirmation before risky actions | Explicit confirm for destructive ops | R32–R33 |

---

## 6. Architecture Lessons (If Building GLM Tooling)

For when the goal is not just prompting GLM but building a harness/UI around GLM APIs:

1. **Backend isolation ("island") pattern** — each backend lives in its own directory (`src/backends/<id>/`), vendor SDK imports are physically restricted per directory, declared in a static `manifest.ts` (id, capabilities, grants, install recipe). UI never imports backends directly; it goes through a facade. → Apply when supporting multiple GLM routes (OpenAI-compatible, Anthropic-compatible, ACP) in one tool.
2. **Zero-core-change philosophy** — consume only public exports; install/uninstall leaves no patches. → Wrap any host agent without forking it.
3. **Headless twin of the UI** — same engine exposed as `run --json` / `session list --json` for automation. → Every GLM tool deserves a JSON CLI alongside its UI.
4. **Event-driven projection + virtualization + bounded caches** — how dsh-TUI stays responsive in multi-hour sessions. → The blueprint for long-running GLM agent UIs.
5. **Idempotent session migration** — deterministic UUIDs per source conversation; re-import skips duplicates; reasoning traces and tool calls preserved. → For moving user history between GLM and other agents.
6. **Escalating notification fallbacks** — OSC 777/9 → BEL → native toast, with byte-budget-safe truncation. → For GLM task-completion notifications across environments.

---

## 7. Quick-Start Prompt (One-Liner for Every Session)

```
Read GLM-HARNESS.md in this repo and follow its operating rules:
plan-first todos, live status narration, diff-table reporting,
100-word completion summary, parallel execution with task IDs,
and explicit confirmation for destructive operations. Task: <YOUR TASK>
```
