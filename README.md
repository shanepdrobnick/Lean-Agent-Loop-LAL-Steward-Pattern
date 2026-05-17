# Lean Agent Loop (LAL) — Steward Pattern
_Conceived 2026-05-17 — Shane Drobnick & Claude (Anthropic)_
_Built on the Multi-AI Bridge Pattern v1.2_

---

## What This Is

The Lean Agent Loop is an autonomous AI development workflow that uses email
as an async message bus between a reasoning AI node and a stateless executor AI.
It enables multi-step agentic work on real codebases at minimal cost, with
human oversight preserved and budget-aware self-regulation built in.

It is not a framework. It is a pattern — a set of roles, message formats,
and disciplines that can be implemented with tools you already own.

---

## The Problem It Solves

Most autonomous AI development pipelines require either:
- Expensive enterprise infrastructure (orchestration services, managed APIs, databases)
- A human writing every ticket manually in a linear loop
- An agent with no reasoning node — just execution following static scripts

The Lean Agent Loop uses a two-node architecture with a free async message bus
(email) to get most of the benefit of expensive pipelines at near-zero infrastructure cost.

**The key insight:** A reasoning AI that holds the overall goal and makes decisions
is more valuable than raw execution capacity. Budget spent on a smart reasoning node
that fires cheap execution tasks is more efficient than budget spent on a single
expensive end-to-end agent with no goal-awareness.

---

## The Two Nodes

### The Steward (claude.ai)
- Holds the overall job goal and success criteria
- Reads execution reports and decides: proceed / correct / escalate
- Writes precise task emails for the Executor
- Manages budget awareness across the run
- Escalates to the human only when genuinely blocked
- Has persistent context via project knowledge
- Cost: low — reasoning and task writing is short-context work

### The Executor (Claude CLI)
- Stateless — fresh instance per task, no memory between runs
- Reads one task email, executes, writes one structured report, exits
- Works only on explicitly scoped files
- Runs git operations (branch, commit) as part of task protocol
- Self-checks success criteria before reporting
- Cost: per-execution — tokens spent only when work is happening

---

## The Message Bus

Standard Gmail. Free. Persistent. Auditable. Accessible from any device.

Three Gmail labels manage state:
- **VS-TASKS** — task waiting for Executor
- **VS-PROCESSING** — Executor currently running
- **VS-DONE** — completed, archived

The watcher script (`lal_watcher.py`) polls VS-TASKS, moves emails through
the state machine, fires CLI, captures output, sends report back.

---

## Email Schema

### Task Email (Steward → Executor)
```
Subject: [VS-TASK] TASK_ID · phase · description

TASK_ID: VS-GAL-001
PHASE: 1 of 3
BUDGET_MODE: session_only | weekly_cap | paid_aud
BUDGET_WEEKLY_CAP_PERCENT: 40
GIT_BASE_BRANCH: Experimental

OBJECTIVE: Clear one-sentence statement of what to achieve.
FILES_IN_SCOPE: explicit list — Executor touches nothing else
SUCCESS_CRITERIA:
  - testable condition 1
  - testable condition 2
CONSTRAINTS: what not to do
CONTEXT: everything CLI needs — it wakes up cold with no memory
```

### Report Email (Executor → Steward)
```
Subject: [VS-REPORT] TASK_ID · phase · description

=== LAL REPORT ===
TASK_ID: VS-GAL-001
PHASE: 1 of 3
STATUS: complete | partial | blocked
COMPLETED: what was done
FILES_MODIFIED: list
BRANCH_CREATED: LAL/VS-GAL-001-description
TEST_RESULTS: pass/fail/output
TOKENS_USED: from /usage
BLOCKERS: if status is blocked
NOTES: anything Steward should know
=== END REPORT ===
```

### Other Prefixes
```
[VS-ESCALATE]  — Steward → human, needs decision above Steward authority
[VS-COMPLETE]  — Steward → human, job finished
[VS-IDLE]      — Watcher → human, idle warning, shutdown imminent
KEEPALIVE      — Human → watcher, reset idle timer
```

---

## Git Protocol (mandatory every task)

```
1. git checkout [GIT_BASE_BRANCH] && git pull
2. git checkout -b LAL/[TASK_ID]-[short-description]
3. Work only on FILES_IN_SCOPE
4. git add [FILES_IN_SCOPE only] — never git add .
5. Verify SUCCESS_CRITERIA pass
6. git commit -m "LAL: [TASK_ID] — [description]"
7. Do NOT merge, do NOT push to base branch
8. Report branch name as BRANCH_CREATED in report
```

Human reviews the branch and merges when satisfied. The LAL loop never
touches the base branch directly.

---

## Budget Modes

Three modes set per session, stamped into every task email:

**session_only** — use current Claude CLI session tokens only. Zero paid credit.
Executor checks `/usage` at task start, stops if session exhausted.

**weekly_cap** — stop if weekly usage % exceeds the cap.
`BUDGET_WEEKLY_CAP_PERCENT: 40` — stops at 40% of weekly allowance.

**paid_aud** — AUD ceiling for paid credit usage.
`BUDGET_CEILING_AUD: 2.00` — stops at $2.00 AUD spent.

The watcher tracks cumulative cost across the session and sends
`[VS-ESCALATE]` when any ceiling is hit.

---

## Session Start Protocol (Steward mode)

When entering a LAL session, Steward asks three questions:

1. **Budget mode** — session only, weekly cap %, or AUD ceiling?
2. **Weekly cap %** if applicable — default 40% if not specified
3. **Shutdown on idle** — yes/no

Then confirms the job scope and fires the first task email.

---

## The Watcher Script

`lal_watcher.py` — Python 3, Gmail API, no cloud services required.

**What it does:**
- Polls VS-TASKS label on configurable interval (default 2 min)
- Moves email to VS-PROCESSING before firing CLI (prevents duplicate runs)
- Fires Claude CLI as fresh subprocess with task as input
- Captures output, parses cost
- Sends VS-REPORT email back
- Moves email to VS-DONE
- Tracks cumulative budget, stops at ceiling
- Sends idle warning email after X minutes of no tasks
- Initiates PC shutdown after idle timeout (Linux and Windows)
- KEEPALIVE email reply cancels shutdown countdown

**Config file:** `lal_config.json`
**Context prepend:** `lal_context.txt` — project context stamped into every task
**Setup guide:** `LAL_SETUP.md`

---

## Why Email

- **Free** — no infrastructure cost
- **Persistent** — full audit trail of every task and report
- **Async** — Steward and Executor don't need to be running simultaneously
- **Mobile-accessible** — human can monitor from phone, reply KEEPALIVE,
  read escalations without being at the PC
- **Human-readable** — every message is inspectable, no opaque API payloads

---

## What Makes This Different from ADLC

Standard ADLC (Autonomous Development Loop with Checkpoints) is linear:
tickets go in, code comes out, the AI executes but does not reason about
whether the execution is strategically correct.

The Lean Agent Loop has a **reasoning node that holds the goal**. When
something unexpected happens mid-task, the Steward reads the report,
decides if the plan needs to change, and writes a corrected next task.
That decision-making capacity is what distinguishes this from a linear pipeline.

The Executor is cheap and stateless. The Steward is where the intelligence lives.
Budget spent on the reasoning node is multiplied across all execution tasks.

---

## Cost Profile

A typical refactor session:
- Steward task writing: minimal tokens — short context, structured output
- Executor per task: varies by task size — 1,000-10,000 tokens typical
- Watcher script: zero AI cost — pure Python polling loop

Solo developer running this on a Claude Pro subscription + occasional paid
overflow can run meaningful multi-file refactor jobs for under $5 AUD per session.

---

## Limitations and Known Issues

**Manual send step:** Steward creates Gmail drafts but cannot send directly
via current MCP tooling. Human sends draft + applies VS-TASKS label.
Workaround: configure watcher to monitor Sent folder for [VS-TASK] subjects
(one-line config change eliminates the label step).

**Cost parsing:** CLI cost extraction is best-effort regex on `/usage` output.
Zero-cost tasks (errors, test tasks) show $0.00 correctly.
Actual cost tracking depends on CLI output format consistency.

**No persistent Executor memory:** Each CLI invocation is cold. Task email
must be fully self-contained. This is a feature (clean isolation, no context
bleed) but requires discipline in task writing.

---

## Relationship to Multi-AI Bridge Pattern

The Bridge Pattern defines role separation between AI nodes (co-designer,
senior engineer, junior executor) and the discipline of single-file tasks
for Cursor agents.

The Lean Agent Loop extends the Bridge Pattern with:
- Async execution via email message bus
- Budget-aware self-regulation
- Steward mode — combined co-designer + senior operating autonomously
- PC lifecycle management (idle shutdown, KEEPALIVE)

LAL can be used without the Bridge Pattern on any codebase.
The Bridge Pattern can be used without LAL for synchronous sessions.
Together they cover both synchronous (human-in-loop) and async (autonomous) workflows.

---

## Files

| File | Purpose |
|---|---|
| `lal_watcher.py` | Watcher script — Gmail polling, CLI firing, report sending |
| `lal_config.json` | All configuration |
| `lal_context.txt` | Project context prepended to every task email |
| `LAL_SETUP.md` | Installation and setup guide |
| `credentials.json` | Gmail OAuth (user-generated, not in repo) |

---

## Origin

Conceived during a Vector Storm game development session, 2026-05-17.
Motivated by the need for autonomous multi-component refactoring on a solo
developer budget without enterprise infrastructure.

The insight that a reasoning node holding the goal is more valuable than
raw execution capacity came from comparing this setup to an enterprise ADLC
pipeline — same architecture, fraction of the cost, with decision-making
the enterprise version lacked.

_"Thirty Australian dollars used well beats thousands spent on linear execution."_
