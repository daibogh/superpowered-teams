# Changelog

All notable changes to this plugin are documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and
this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] — 2026-05-13

### Changed

- **Version-hardened CC-coupled assumptions.** Added a § Compatibility section to `agent-team-driven-development/SKILL.md` that centralizes the experimental flag name (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`), minimum Claude Code version, model-tier policy, and metadata field names used for Phase 3 dispatch. Inline references to "Opus 4.7-1M" across SKILL.md, prompt templates, and `docs/design-spec.md` now point to that section instead of repeating model-version specifics. The motivation: when CC renames a flag or a new Opus-family model ships, only the table needs to change rather than ~10 scattered occurrences.

### Why

Skill files inherit a snapshot of the harness at authoring time. Hardcoding `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` and `Opus 4.7-1M` across the skill couples behavior to specific values that drift with each Claude Code release. Centralizing them gives future maintainers a single edit point and makes the brittleness explicit.

## [0.2.0] — 2026-05-13

### Added

- `agents/code-reviewer.md` — plugin-namespaced subagent (`superpowered-teams:code-reviewer`) for code quality review and final cross-cutting review. Replaces the v0.1.0 references to `superpowers:code-reviewer`, which was a prompt template (not a subagent type) and would fail at runtime.
- Plan-approval flow integration. `writing-plans-for-teams` now supports a `Plan approval required:` task metadata field for risky changes (schema migrations, destructive ops, auth/session logic, refactors >5 files). `agent-team-driven-development` documents how to spawn approval-required teammates and respond to their `plan_approval_request` envelopes.
- Optional hooks integration (`TeammateIdle`, `TaskCreated`, `TaskCompleted`) documented in `agent-team-driven-development` for harness-enforced quality gates.

### Changed

- **Echo defense — Phase 3 dispatch is now metadata-driven, not prose-driven.** Previously, the lead's control loop pattern-matched on prose markers (`Status: DONE`, `SPEC ISSUES`, `APPROVED`) in incoming messages, which made it vulnerable to two echo loops: peer-DM idle summaries and teammate quote-backs. Both could re-trigger reviewer dispatch on stale state. The lead now dispatches reviewers based on `task.metadata.report_ready` and `task.metadata.status_code`, read via `TaskGet`. Implementer prompts now require teammates to write the Completion Report into the task description (so the lead reads from `TaskGet`, never from message bodies) and forbid quoting prior messages in replies.
- The Completion Report Format is now split into two artifacts: metadata flags (the dispatch trigger) and a prose report block in the task description (lead-readable). The teammate writes both before sending a one-line "ready for review" notification.
- `code-quality-reviewer-prompt.md`, `journal-snapshot-template.md`, and SKILL.md `subagent_type` references all updated from `superpowers:code-reviewer` (broken) to `superpowered-teams:code-reviewer` (this plugin's agent).
- `implementer-prompt.md` removed duplicated `SendMessage` protocol notes — the live tool description is authoritative for those.

### Fixed

- Lead operating on stale directives ("echoes from teammates") in v0.1.0 sessions. Root cause was prose-pattern triggering combined with peer-DM summary echoes; fix is metadata-driven dispatch + no-quote rule. See "Echo defense" above.
- Three references to a non-existent `superpowers:code-reviewer` agent type that would have raised "unknown subagent type" errors at first reviewer dispatch.

## [0.1.0] — 2026-05-02

Initial release.

### Added

- `writing-plans-for-teams` skill — runs a four-criterion team fitness check on a spec; produces a team-format plan (Wave Analysis, Lifetime Plan, per-task metadata) if fit, or hands off to `superpowers:writing-plans` if not.
- `agent-team-driven-development` skill — executes team-format plans by orchestrating a team of persistent specialist implementers (Agent Teams) with two-stage review (spec compliance + code quality) per task, hybrid lead-assign / self-claim task flow, and a structured journal snapshot at completion.
- Post-install CLAUDE.md snippet (`claude-md-snippet.md`) that routes `superpowers:brainstorming`'s terminal step to `writing-plans-for-teams` instead of `superpowers:writing-plans`.

### Requirements

- Claude Code ≥ 2.1.32
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings
- `superpowers` plugin installed (hard dependency — `superpowers:writing-plans` as fitness-check fallback, `superpowers:subagent-driven-development` as serial-execution fallback, `superpowers:finishing-a-development-branch` at completion. NOTE: v0.1.0 also referenced a `superpowers:code-reviewer` agent that does not exist; v0.2.0 ships its own `superpowered-teams:code-reviewer` to fix this.)
