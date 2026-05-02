# Changelog

All notable changes to this plugin are documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and
this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] — 2026-05-02

Initial release.

### Added

- `writing-plans-for-teams` skill — runs a four-criterion team fitness check on a spec; produces a team-format plan (Wave Analysis, Lifetime Plan, per-task metadata) if fit, or hands off to `superpowers:writing-plans` if not.
- `agent-team-driven-development` skill — executes team-format plans by orchestrating a team of persistent specialist implementers (Agent Teams) with two-stage review (spec compliance + code quality) per task, hybrid lead-assign / self-claim task flow, and a structured journal snapshot at completion.
- Post-install CLAUDE.md snippet (`claude-md-snippet.md`) that routes `superpowers:brainstorming`'s terminal step to `writing-plans-for-teams` instead of `superpowers:writing-plans`.

### Requirements

- Claude Code ≥ 2.1.32
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings
- `superpowers` plugin installed (hard dependency — the skills invoke `superpowers:code-reviewer` agent, `superpowers:writing-plans` as fitness-check fallback, and `superpowers:finishing-a-development-branch` at completion)
