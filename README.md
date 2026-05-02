# superpowered-teams

Parallel team execution for [superpowers](https://github.com/obra/superpowers) plans via Claude Code's Agent Teams feature.

A team fitness check at plan time, a persistent-specialist orchestrator at execution time, and graceful fallback to superpowers' standard serial flow when the work doesn't warrant a team.

## What this plugin adds

Two user-invocable skills:

- **`writing-plans-for-teams`** — runs a four-criterion fitness check on a spec. If the work benefits from parallelism, writes a team-format plan (Wave Analysis, per-task metadata, Lifetime Plan). If not, hands off to `superpowers:writing-plans` so the serial flow continues unchanged.
- **`agent-team-driven-development`** — executes team-format plans by orchestrating persistent specialist implementers through `TeamCreate` / `SendMessage` / `TaskCreate`, running two-stage review (spec compliance, then code quality) per task, and writing a journal snapshot at completion.

Together they let a single `/brainstorm` conversation fan out to a team when the feature is large enough, and collapse back to a subagent-driven flow when it isn't — with the decision made automatically by the fitness check.

## How it differs from what's already in superpowers

| Need | Skill to use |
|---|---|
| 2+ independent one-shot tasks, no shared state | `superpowers:dispatching-parallel-agents` |
| Serial plan, one role, tight dependency chain | `superpowers:subagent-driven-development` |
| 4+ tasks across 2+ waves with role specialization and cross-task context | **`superpowered-teams:agent-team-driven-development`** |

The fitness gate makes the choice automatic. You don't have to decide.

## Requirements

- Claude Code **≥ 2.1.32**
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in `~/.claude/settings.json`
- [**superpowers**](https://github.com/obra/superpowers) plugin installed (hard dependency — see below)

### Why superpowers is required

`superpowered-teams` is a superpowers extension. It invokes:

- `superpowers:code-reviewer` agent — both review stages and final cross-cutting review
- `superpowers:writing-plans` — fallback when fitness check fails
- `superpowers:subagent-driven-development` — target of the fallback handoff for executors
- `superpowers:finishing-a-development-branch` — post-team completion handoff

Soft references (documented, used as concept pointers):

- `superpowers:brainstorming`, `superpowers:using-git-worktrees`, `superpowers:test-driven-development`, `superpowers:requesting-code-review`

## Install

### Option A — Direct clone (works today)

Until this plugin lands in a marketplace, clone it directly into your user plugins directory:

```bash
# superpowers first (if not already installed)
/plugin install obra/superpowers

# then this plugin via direct clone
git clone https://github.com/narwhalishus/superpowered-teams ~/.claude/plugins/user/superpowered-teams
```

Restart Claude Code (or start a new conversation) — the skill loader will pick up the new plugin automatically.

### Option B — Marketplace install (when available)

```
/plugin install superpowered-teams@claude-plugins-official
```

Not yet submitted to the `claude-plugins-official` marketplace. Use Option A for now.

### Required post-install step

**The plugin's skills are underused without this step.** By default `superpowers:brainstorming` hands off to `superpowers:writing-plans` at its terminal state, which skips the team fitness check entirely. Add the snippet from [`claude-md-snippet.md`](./claude-md-snippet.md) to your global CLAUDE.md to route brainstorming through `writing-plans-for-teams` instead:

```bash
cat ~/.claude/plugins/cache/<marketplace>/superpowered-teams/*/claude-md-snippet.md >> ~/.claude/CLAUDE.md
```

Or copy the contents of `claude-md-snippet.md` manually. The snippet is intentionally a user-instruction-priority override — it sits at the top of the instruction stack, above any plugin skill content, and can be removed to disable the integration.

### Verify

Start a new conversation and say: *"let's brainstorm a small feature."*

- `superpowers:brainstorming` activates
- At the end, it should hand off to `superpowered-teams:writing-plans-for-teams`
- Run the fitness check — for a trivially serial spec, it will fall back to `superpowers:writing-plans` (no team plan produced)
- For a spec with multiple waves and roles, it will produce a team-format plan and offer Agent Team-Driven execution

## The fitness check

Before `writing-plans-for-teams` writes any tasks, it evaluates:

| Criterion | Threshold |
|---|---|
| Total tasks | ≥ 4 |
| Waves with 2+ tasks | ≥ 2 |
| Distinct specialist roles | ≥ 2 |
| No pervasive shared state across tasks | judgment |

**All four must hold.** If any fails, the skill announces the failure and invokes `superpowers:writing-plans` as a sub-skill — producing a standard serial plan. This gate exists because team overhead (setup, coordination, review loops) is real; parallelism only pays off at a certain scale.

## Team architecture

| Role | Count | Lifetime | Spawn mechanism |
|---|---|---|---|
| Lead (you) | 1 | Session | Main session |
| Specialist implementer | 1–3 simultaneous | Spawn-per-wave default, full-session if ≥2 waves of work (decided at plan time in the Lifetime Plan) | `Agent` tool with `team_name` + `name` |
| Spec compliance reviewer | Per task | One-shot | `Agent` tool (subagent, no `team_name`) |
| Code quality reviewer | Per task | One-shot | `Agent` tool with `subagent_type: superpowers:code-reviewer` |
| Final cross-cutting reviewer | 1 at completion | One-shot | `Agent` tool with `subagent_type: superpowers:code-reviewer` |

**Implementers are teammates** (persistent, remember the codebase across tasks, communicate via `SendMessage`). **Reviewers are subagents** (fresh context, no bias from watching code get written). The asymmetry is deliberate — it's the main reason this flow produces better reviews than single-role variants.

## Documentation inside the plugin

Read the skill files directly for the full behavior spec:

- [`skills/writing-plans-for-teams/SKILL.md`](skills/writing-plans-for-teams/SKILL.md) — fitness check, team plan format, handoff rules
- [`skills/agent-team-driven-development/SKILL.md`](skills/agent-team-driven-development/SKILL.md) — phases, status protocol, error recovery, red flags
- [`skills/agent-team-driven-development/implementer-prompt.md`](skills/agent-team-driven-development/implementer-prompt.md) — initial spawn, follow-up, review-feedback templates
- [`skills/agent-team-driven-development/spec-reviewer-prompt.md`](skills/agent-team-driven-development/spec-reviewer-prompt.md) — spec compliance reviewer
- [`skills/agent-team-driven-development/code-quality-reviewer-prompt.md`](skills/agent-team-driven-development/code-quality-reviewer-prompt.md) — code quality reviewer
- [`skills/agent-team-driven-development/journal-snapshot-template.md`](skills/agent-team-driven-development/journal-snapshot-template.md) — end-of-session journal format

## Limitations and expectations

- **Agent Teams is experimental.** The `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` flag exists because the primitives can change without notice. If Anthropic renames or restructures `TeamCreate` / `SendMessage`, this plugin needs a patch release.
- **Session death mid-wave is documented.** Agent Teams doesn't yet support resuming a team session cleanly across Claude Code restarts. The skill's error handling covers the recovery shape, but it's manual.
- **No context-rot auto-refresh.** Agents can't reliably self-detect context degradation; hard-coded refresh thresholds would destroy the continuity benefit of persistent specialists. If you observe a teammate making mistakes inconsistent with their earlier work, prompt a controlled shutdown+respawn via `SendMessage`.
- **Max 3 simultaneous implementers.** More hits diminishing returns from git conflicts and coordination overhead.

## Background

Built May 2026 using Claude Code's Agent Teams primitives, drawing on [bok-'s community swarm skills](https://github.com/obra/superpowers/issues/429) as prior art. See [`docs/design-spec.md`](./docs/design-spec.md) for the full rationale behind the hybrid task-assignment flow, two-stage review asymmetry, S3 lifetime policy, and why this couldn't live in upstream superpowers.

## License

MIT. See [LICENSE](./LICENSE).
