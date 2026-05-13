# Agent Teams Skills — Design Spec

**Date:** 2026-05-02
**Status:** Design approved, pending implementation plan
**Scope:** Build two user-level skills that integrate Claude Code's experimental Agent Teams feature with the superpowers plugin workflow.

---

## Problem

Superpowers 5.0.7 has `subagent-driven-development` and `dispatching-parallel-agents` for parallel work, but neither uses Claude Code's Agent Teams primitives (`TeamCreate`, `SendMessage`, persistent teammates via the Agent tool with `team_name`). Subagents are one-shot and can't talk to each other; teammates are persistent, addressable, and share a task list. Issue obra/superpowers#429 documents this gap, with community workarounds from bok-. The upstream maintainer has not merged agent-teams support and the community PRs (#470, #598, #733) were closed.

Bryan wants Agent Teams support for heavy superpowers use. Bok's skills are the closest prior art but were written Feb 2026 before Opus 4.7, use protocol shapes that don't match the current tool surface (`recipient:`/`content:` vs actual `to:`/`summary:`/`message:`), and predate several primitives worth exploiting (1M context, teammate self-claim, verified `addBlockedBy` task dependencies).

## Goal

Two user-level skills installed at `~/.claude/skills/`:

1. **`writing-plans-for-teams`** — Gate + plan author. Runs a team fitness check on a spec, hands off to `superpowers:writing-plans` if team execution won't pay off, otherwise produces a team-format plan.
2. **`agent-team-driven-development`** — Executor. Reads a team-format plan, orchestrates a team of specialist implementers (persistent, via Agent Teams) with two-stage review (spec + quality, via subagents) per task, and hands off to `superpowers:finishing-a-development-branch` at completion.

## Non-goals

- Hook integration (`TaskCompleted`, `TeammateIdle`)
- Model selection logic (environment uses Opus 4.7-1M universally)
- Context-rot self-detection or refresh heuristics
- Cost tracking or token reporting
- Automated eval harness
- Migration tooling from bok's skills
- Multi-team / nested-team support (Agent Teams docs forbid nesting)
- Plan metadata beyond the superpowers-convention blockquote

---

## Architecture

### Two skills, loose coupling via superpowers blockquote convention

```
┌─────────────────────────────────────────────────────────────────────┐
│  writing-plans-for-teams                                            │
│  - Runs team fitness check                                          │
│  - If fit: emits team-format plan with REQUIRED SUB-SKILL           │
│    blockquote naming agent-team-driven-development                  │
│  - If unfit: hands off to superpowers:writing-plans                 │
└────────────────────────────┬────────────────────────────────────────┘
                             │ plan document
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│  agent-team-driven-development                                      │
│  - Verifies blockquote matches this skill                           │
│  - TeamCreate, Wave 1 kickoff via lead-assigned spawn               │
│  - Teammates self-claim subsequent tasks in their role              │
│  - Two-stage review per completed task (spec → quality subagents)   │
│  - Journal entries written to task descriptions on completion       │
│  - Final cross-cutting review, journal snapshot, TeamDelete         │
└─────────────────────────────────────────────────────────────────────┘
```

### Role matrix

| Role | Count | Spawn mechanism | Lifetime | Model |
|---|---|---|---|---|
| Lead | 1 | Main session | Session lifetime | Opus 4.7-1M |
| Specialist implementer | 1–3 simultaneous | `Agent` tool with `team_name` + `name` | S3: spawn-per-wave default, upgrade to full-session if ≥2 waves of work | Opus 4.7-1M |
| Spec reviewer | Per task | `Agent` tool, no `team_name` | One-shot | Opus 4.7-1M |
| Code quality reviewer | Per task | `Agent` tool, `subagent_type: superpowered-teams:code-reviewer` | One-shot | Opus 4.7-1M |
| Final cross-cutting reviewer | 1 | `Agent` tool, `subagent_type: superpowered-teams:code-reviewer` | One-shot at completion | Opus 4.7-1M |

**Key asymmetry:** Implementers are teammates (persistent, survive across waves, remember codebase); reviewers are subagents (fresh context, no accumulated bias from watching code get written). This mirrors bok's original insight.

### Communication surface

- **`TeamCreate`** at session start → creates `~/.claude/teams/<plan-slug>/` + `~/.claude/tasks/<plan-slug>/`
- **`TaskCreate`** for every plan task — `subject` uses `[<role>] Task N: <name>` prefix (for self-claim filtering), `description` carries full task text including Specialist/Depends on/Produces metadata
- **`TaskUpdate addBlockedBy`** after create pass to wire cross-wave dependencies
- **`TaskUpdate owner:`** — used by lead for Wave 1 kickoff assignments only; teammates use it to self-claim thereafter
- **`SendMessage`** — initial task briefing, question answers, verbatim reviewer feedback, shutdown
- **Idle state is normal** — lead ignores idle notifications except to trigger reviews or wave-complete checks

### Self-claim filter

Teammates claiming work in Wave 2+ call `TaskList`, filter to tasks where `subject` starts with `[<my-role>]` AND `status: pending` AND `owner` empty AND `blockedBy` empty, prefer lowest ID, claim via `TaskUpdate owner:`. Subject prefix is the chosen filter key because:

- `subject` is a constrained, grep-intended field (unlike arbitrary description prose)
- Bracketed role names avoid substring collisions (`[frontend]` ≠ `[frontend-designer]`)
- Typos surface loudly — unclaimed task visible in `TaskList` triggers lead investigation
- Self-documenting in `TaskList` output

---

## Plan format (output of `writing-plans-for-teams`)

Based on `superpowers:writing-plans`'s existing format with three additions. Same file location convention: `docs/superpowers/plans/YYYY-MM-DD-<feature>.md`. Same bite-sized checkboxed steps. Same No-Placeholders rules.

### Addition 1 — Blockquote names this skill

The square-bracketed text below is placeholder syntax the skill fills in when emitting a plan; it is not a placeholder in this spec.

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use agent-team-driven-development to implement this plan in parallel with a specialist team. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [one sentence]
**Architecture:** [2-3 sentences]
**Tech Stack:** [key tech]

---
```

### Addition 2 — Wave Analysis (after header)

```markdown
## Wave Analysis

### Specialists

| Role | Expertise | Tasks |
|------|-----------|-------|
| backend-engineer | API, DB, server conventions | Tasks 1, 2, 3 |
| react-engineer | React/TS, component patterns | Tasks 4, 6 |
| docs-writer | Technical writing | Task 5 |

### Waves

**Wave 1: Foundation** — schema + types
- Task 1 (backend-engineer) — Zod schemas

  *Parallel-safe because:* only one task in wave

**Wave 2: Independent consumers**
- Task 2 (backend-engineer) — DB migration
- Task 5 (docs-writer) — iOS model docs

  *Parallel-safe because:* different directories, no import relationship
  *Depends on Wave 1:* Task 1's schemas at `packages/shared/src/schemas/`

[... additional waves ...]

### Dependency Graph

```
Task 1 → Task 2 → Task 3 → Task 4
Task 1 → Task 5
```

### Lifetime Plan

| Specialist | Waves | S3 decision |
|---|---|---|
| backend-engineer | 1, 2, 3 | Full-session |
| react-engineer | 3, 4 | Full-session |
| docs-writer | 2 | Spawn-per-wave, shut down after Wave 2 |
```

### Addition 3 — Task metadata block on every task

```markdown
### Task N: [Component Name]

**Specialist:** backend-engineer
**Depends on:** Task 1 (Zod schemas at `packages/shared/src/schemas/`)
**Produces:** `apps/server/src/db/schema.ts`, migration at `apps/server/drizzle/`

**Files:**
- Create: `exact/path/to/file.ts`
- Test: `exact/path/to/test.ts`

- [ ] **Step 1: Write the failing test**
[... standard superpowers bite-sized steps ...]
```

### Plan-writing rules

- Max 3 tasks per wave
- Same-wave tasks must not touch same files
- Same-wave tasks must not have import relationships
- Dependency graph must be acyclic
- Max 3 distinct specialist roles
- Every task must have Specialist, Depends on, Produces fields

### Self-review additions (beyond `superpowers:writing-plans`'s checklist)

1. Wave grouping safety — confirm no file overlap or import relationships for each same-wave pair
2. Task metadata completeness — every task has all three required fields
3. Specialist role consistency — every `Specialist:` reference matches a row in Specialists table

---

## Fitness check (inside `writing-plans-for-teams`)

Runs after spec handoff, before drafting any task content. All four criteria must hold for team execution; any failure triggers handoff to `superpowers:writing-plans`.

| Criterion | Threshold |
|---|---|
| Total tasks | ≥ 4 |
| Waves with 2+ tasks | ≥ 2 |
| Distinct specialist roles | ≥ 2 |
| No pervasive shared state across tasks | (judgment) |

On failure: announce reason to user, invoke `superpowers:writing-plans` as sub-skill, do not attempt conversion.

---

## Execution flow (inside `agent-team-driven-development`)

### Phase 1 — Boot

1. Read plan file end-to-end, extract tasks, Specialists, Waves, Dependency Graph, Lifetime Plan.
2. Verify blockquote names `agent-team-driven-development`. Halt otherwise.
3. Verify environment: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, CC ≥ 2.1.32. Halt with remediation if not.
4. Verify git worktree (unless user explicitly authorized `main`/`master`).
5. `TeamCreate` with `team_name: <plan-slug>` and `description: <plan Goal>`.
6. `TaskCreate` for every plan task with subject prefix `[<role>] Task N: <name>` and full description.
7. Second pass: `TaskUpdate addBlockedBy` for all cross-wave dependencies from Depends-on metadata.
8. Announce: `"Starting agent-team-driven-development. Team <slug> created with N tasks across W waves."`

### Phase 2 — Wave 1 kickoff (lead-assigned)

For each Wave 1 task (max 3 simultaneous):

Spawn specialist via `Agent` tool:
```
subagent_type: general-purpose
team_name: <plan-slug>
name: <role-from-plan>
description: "<role>: Wave 1 kickoff"
prompt: <from implementer-prompt.md, filled in>
```

Spawn prompt includes: role + expertise, "persistent team member" framing, self-claim protocol, full text of first task, self-review checklist, status protocol, journal format for completion report.

Lead waits for idle notifications — does not poll.

### Phase 3 — Wave execution (hybrid γ: lead intervention + self-claim)

**Lead's continuous loop:**

- On teammate completion message → dispatch spec reviewer (don't wait for other teammates)
- On spec reviewer pass → dispatch code quality reviewer
- On code quality reviewer pass → append journal block to task description, `TaskUpdate status: completed`
- On reviewer issues → forward reviewer report verbatim via `SendMessage`, wait for fix report, re-dispatch fresh reviewer of same stage
- On teammate `NEEDS_CONTEXT` → answer via `SendMessage`
- On teammate `BLOCKED` → diagnose per four-way rubric (see Status Protocol below)
- On teammate `DONE_WITH_CONCERNS` → correctness/scope concern → address before review; observation → note and proceed
- On wave completion → verify via `TaskList` scan, then Wave N+1 kickoff

**Teammate self-claim loop (Wave 2+):**

- After lead marks their current task `completed`, teammate calls `TaskList`
- Filters: `status: pending`, `owner: empty`, `blockedBy: empty`, `subject` starts with `[<my-role>]`
- Claims lowest ID via `TaskUpdate owner: <my-name>`
- `TaskGet` for full description (including journal entries from completed dependencies)
- Implement, test, commit, self-review, report via `SendMessage`

**Wave transition (S3 application):**

- Specialists with work in new wave, already alive → `SendMessage` telling them new tasks are claimable
- Specialists in Lifetime Plan but not yet alive → spawn now
- Specialists alive but no further work → `{type: "shutdown_request"}`

### Phase 4 — Completion

1. All tasks `completed` and reviewed.
2. Dispatch final cross-cutting reviewer (`superpowered-teams:code-reviewer` subagent) with base-SHA-to-head-SHA diff.
3. Write `docs/superpowers/plans/<plan-slug>.journal.md` by extracting journal blocks from completed task descriptions in order. **`TeamDelete` blocked until this file exists.**
4. Shutdown live teammates via `SendMessage {type: "shutdown_request"}`.
5. `TeamDelete` after all teammates idle-terminate.
6. Hand off to `superpowers:finishing-a-development-branch`.

---

## Status protocol (adopted from `superpowers:subagent-driven-development`)

Every teammate completion message ends with one of four statuses:

| Status | Meaning | Lead's next action |
|---|---|---|
| `DONE` | Task complete, self-review clean, ready for review | Dispatch spec reviewer |
| `DONE_WITH_CONCERNS` | Complete but teammate flagged doubts | Read concerns. Correctness/scope → address first. Observation → note and review. |
| `NEEDS_CONTEXT` | Can't proceed without info plan didn't provide | Answer via `SendMessage`, teammate continues |
| `BLOCKED` | Can't complete (test won't pass, task wrong) | Diagnose per rubric (context / reasoning / too-large / plan-wrong) |

### `BLOCKED` recovery rubric

| Cause | Recovery |
|---|---|
| Missing context the plan didn't provide | Lead answers via `SendMessage`, teammate continues |
| Reasoning limitation | N/A at Opus 4.7-1M — escalate as plan-too-large or plan-wrong |
| Task too large | Lead splits task in plan, creates subtasks with dependencies, shuts down teammate, respawns |
| Plan is wrong | Lead halts, surfaces to user, does not attempt plan rewrite autonomously |

---

## Completion report format (mandatory from every teammate)

Enforced by lead refusing to proceed without the Journal block:

```
Status: DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED

Summary: <one-sentence what I built>

Journal:
- Files created: [paths]
- Files modified: [paths]
- Exports added: [names]
- Schema/migrations: [if any]
- Patterns established: [notes, or "none"]
- Notes: [free text, or "none"]

Tests: <N passing / M failing / reason if failing>
Commits: <SHAs>
Self-review: <findings, or "clean">
Concerns: <if DONE_WITH_CONCERNS>
Blocker: <if BLOCKED>
Questions: <if NEEDS_CONTEXT>
```

Lead copies the Journal block verbatim into task description on `TaskUpdate status: completed`. The rest is transient.

---

## Review feedback handling — verbatim rule

```
SendMessage:
  to: <teammate>
  summary: "[Spec|Quality] review issues for Task N"
  message: |
    [stage] review found issues:

    --- reviewer report (verbatim) ---
    <full reviewer output>
    --- end reviewer report ---

    Please address these specifically, re-run tests, commit, report back
    with updated Status block. Do NOT mark task complete — only lead does that.
```

Lead does not paraphrase reviewer findings. Accumulated reviewer prose at Opus 4.7-1M is within working range by token arithmetic. There is no reliable self-detection mechanism for context quality degradation. If the user observes a teammate making mistakes inconsistent with their earlier work, the user prompts controlled refresh via shutdown + respawn.

---

## Error handling and edge cases

### Teammate-level

- **`BLOCKED`** — four-way rubric (see Status Protocol)
- **`DONE_WITH_CONCERNS`** — route by concern type
- **`NEEDS_CONTEXT`** — answer; update plan file if the question reveals a real gap
- **Idle without completion** — nudge once via `SendMessage`; if unresponsive, shut down and respawn with context
- **Catastrophically wrong code** — after 2 full review cycles (spec + quality, with fixes in between) without convergence, shut down and respawn. If replacement also fails, escalate to user.

### Reviewer-level

- **Contradictory reviews across iterations** — dispatch fresh third reviewer with both prior reports; if unresolved, escalate to user (spec is ambiguous). Never average.
- **Approved but actually broken (caught by final reviewer)** — reopen task, respawn specialist, fix, re-review. Note the escalation in journal snapshot.
- **Reviewer timeout/failure** — redispatch once with same prompt; on second failure, lead does inline review (acceptable since review is read-only context addition).

### Wave-level

- **Wave stuck on one teammate** — other teammates do NOT advance until resolved. Follow teammate-level recovery.
- **Unclaimed task after wave declared complete** — halt Wave N+1 kickoff, inspect `TaskList`, fix subject prefix or assign manually.
- **`addBlockedBy` blocks Wave N+1 task** — structural backstop catching wave-tracking mistakes; resolve the real blocker, cascading auto-unblocks.

### Team-level

- **`TeamCreate` fails (slug collision)** — append session date to slug. For orphaned teams, surface to user before deleting.
- **`TeamDelete` fails (active members)** — re-send `shutdown_request`; if stuck, surface to user.
- **Session dies mid-wave** — documented Agent Teams limitation. On resume, lead reads `TaskList`, shuts down any `in_progress` tasks with ghost owners, respawns with context about the prior teammate's last commit SHA.
- **User interrupts mid-task** — lead `SendMessage` asking teammate to pause and report uncommitted state; surface to user.

### Plan-level

- **Missing required fields** — boot-time halt with specific error, suggest re-running `writing-plans-for-teams`
- **Cycle in dependency graph** — detect after Phase 1 `addBlockedBy` pass; halt with cycle identified
- **Overlapping specialist roles** — file locking on `TaskUpdate owner:` serializes; lead clarifies via `SendMessage` to loser

### Wave tracking

Lead maintains mental model of wave boundaries from the plan's Wave Analysis. Before declaring wave complete, runs `TaskList` and visually confirms every task listed in the current wave (by plan task ID) has `status: completed`. `addBlockedBy` is the structural backstop — even if lead miscalibrates, tasks with unmet dependencies remain un-claimable.

---

## Red Flags summary

**Sequencing — never:**
- Start code quality review before spec compliance passes
- Start Wave N+1 before current wave's reviews all pass
- Let an implementer start a new task while their current task has open review issues
- Start work on `main`/`master` without explicit user consent

**Parallelism — never:**
- More than 3 simultaneous implementers
- Tasks touching the same files in the same wave
- Proceed when unsure about independence (serialize instead)
- Give an implementer more than one task at a time

**Communication — never:**
- Paraphrase reviewer reports to teammates (verbatim rule)
- Make implementers read plan files (provide full text in spawn prompt)
- Omit prior-wave journal data when a task has cross-wave dependencies
- Rush teammates past their `NEEDS_CONTEXT` questions
- Treat idle state as failure (it's normal)

**Reviews — never:**
- Skip re-review after fixes
- Let self-review replace actual review (both needed)
- Skip either review stage (spec compliance AND code quality required)
- Proceed with unfixed issues
- Accept reviewer contradictions without a third-reviewer tiebreak

**Recovery — never:**
- Try to fix manually from the lead (context pollution)
- Attempt plan rewrite autonomously (escalate)
- Ignore a stuck teammate hoping they recover (nudge once, then shut down)
- Delete or modify `~/.claude/teams/<slug>/` files mid-session

**Journal — never:**
- Mark a task `completed` without a Journal block in the completion report
- Call `TeamDelete` before writing the journal snapshot file
- Paraphrase journal entries — teammates paste their Journal block verbatim; lead appends verbatim to task description

---

## Integration with superpowers skills

**Upstream:**
- `superpowers:brainstorming` → produces spec consumed by `writing-plans-for-teams`
- `superpowers:using-git-worktrees` → required before team execution; halt if on `main`/`master` without user consent

**Peers / alternatives:**
- `superpowers:writing-plans` → fallback when fitness check fails
- `superpowers:subagent-driven-development` → serial alternative; plans written by `superpowers:writing-plans` point here

**Invoked as sub-skills:**
- `superpowers:test-driven-development` → implementer prompts reference this
- `superpowered-teams:code-reviewer` agent → used for code quality review and final review
- `superpowers:requesting-code-review` → reviewer methodology reference

**Downstream:**
- `superpowers:finishing-a-development-branch` → merge/PR/cleanup after `TeamDelete`

**`update-config` relationship:**
- Boot-time check for `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`
- If not set, halt with message pointing user to `update-config`. Do not auto-invoke.

---

## File structure

```
~/.claude/skills/
├── agent-team-driven-development/
│   ├── SKILL.md
│   ├── implementer-prompt.md
│   ├── spec-reviewer-prompt.md
│   ├── code-quality-reviewer-prompt.md
│   └── journal-snapshot-template.md
└── writing-plans-for-teams/
    └── SKILL.md
```

`SKILL.md` files carry YAML frontmatter (`name:`, `description:`). Prompt templates are plain markdown referenced by relative path from `SKILL.md` (e.g., `./implementer-prompt.md`). Matches `superpowers:subagent-driven-development`'s layout.

---

## Testing approach

Three validation layers — no automated harness.

1. **Smoke tests** — invoke each skill manually on contrived scenarios (serial spec → handoff; team-format plan → team spawn; non-team plan → blockquote mismatch halt)
2. **First real use on low-stakes work** — observe failures on small tasks before critical projects
3. **Iterate on observed failures** — skill failures become Red Flags or structural changes

---

## Design decisions and rationale

### Why two skills instead of one

`writing-plans-for-teams` and `agent-team-driven-development` are invoked at different points in the workflow (plan authoring vs plan execution) and have different trigger conditions. Merging would produce a large file with mode dispatch, which works against skill readability. Two skills with a blockquote handoff match the `superpowers:writing-plans` / `superpowers:subagent-driven-development` pairing.

### Why hybrid task assignment (γ) over pure lead-dispatch or pure self-claim

Pure lead-dispatch (bok's original) makes the lead a coordination bottleneck and underuses `TaskList`'s built-in self-claim. Pure self-claim loses the calibration benefit of lead verifying each specialist started correctly. Hybrid: lead explicit for Wave 1 (calibrate), teammates self-claim from Wave 2 (scale).

### Why journal in task records, not a separate file during session

C3-original had the lead write journal entries to a file per task — many forget-opportunities. Current design puts journal data in the task's description field via `TaskUpdate` (one write per completion), with a single file snapshot at session end gated on `TeamDelete`. Two structural checkpoints instead of per-task behavioral discipline.

### Why verbatim review pass-through

Paraphrase drift is a correctness risk (lead summarizes reviewer findings inaccurately); context accumulation at Opus 4.7-1M is not a real capacity risk (reviewer prose occupies ~1% of working window). Correctness beats efficiency; keep verbatim.

### Why no context-rot refresh heuristic

Agents can't reliably self-detect context quality degradation. Inventing thresholds (5 tasks, 8 tasks) based on no empirical signal risks making the skill worse (refreshing too aggressively destroys S3's specialist continuity). Honest limitation acknowledgment in skill body; detection lives with the user.

### Why two-stage review always (R1)

Considered hooks-based mechanical gates + single judgment reviewer (R4). Rejected: hooks add setup complexity and break the "drop in and it works" property. Kept bok's proven two-stage flow as a first version.

### Why subject prefix filter over metadata

`TaskGet`'s documented output doesn't confirm `metadata` readability. `subject` is a constrained, grep-intended field that surfaces in `TaskList` output, avoids substring collisions when bracketed, and makes typos visible.

### Why S3 over S1 (bok's spawn-per-wave) or S2 (always-full-session)

S1 forces re-onboarding every wave, wasting the continuity benefit of teammate persistence. S2 keeps idle specialists alive even when they have no remaining work. S3 (spawn-per-wave default, upgrade to full-session when ≥2 waves of work) uses the Lifetime Plan table to make the decision once at plan time.

---

## Open items for implementation plan

- Exact wording of the blockquote-verification regex in `agent-team-driven-development` Phase 1
- Whether `TaskUpdate description:` can append to an existing description or requires passing full new description (affects journal write implementation)
- Exact format of the journal snapshot file (`journal-snapshot-template.md` defines this — needs detail at implementation time)
- Exact self-claim filter expression for teammates (depends on whether `TaskList` output surfaces `subject` in a directly-filterable form, or requires iterating `TaskGet`)
- Implementer prompt template content (expansion of Section 3's description into actual prompt text)
