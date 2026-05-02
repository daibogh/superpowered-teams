## Superpowered Teams integration with brainstorming

User instruction (highest priority, overrides skill content): After `superpowers:brainstorming` completes its final step, invoke `superpowered-teams:writing-plans-for-teams` (plugin skill) instead of `superpowers:writing-plans`. `writing-plans-for-teams` runs a team fitness check and falls back to `superpowers:writing-plans` when team execution doesn't fit — so plans that should be serial still use the standard flow.

This overrides brainstorming's "ONLY writing-plans" language.
