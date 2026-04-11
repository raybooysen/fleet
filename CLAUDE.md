# claude-fleet

Claude Code plugin that orchestrates multi-agent team execution of implementation plans.

## Project Structure

```
.claude-plugin/plugin.json    Plugin manifest
agents/                       Agent persona definitions (7 agents)
skills/fleet/SKILL.md         Main orchestration skill
```

## Architecture

The plugin follows a phased execution model:
0. **Phase 0**: Fleet writes the plan to `.fleet/plan.md` and probes for a global code-reviewer at `~/.claude/agents/code-reviewer.md`, setting a `HAS_GLOBAL_CODE_REVIEWER` flag that Phase 4 branches on
1. **Phase 1**: Solution Architect decomposes plan into work streams AND decides team composition (stream names, roles); devil's advocate challenges; user approves the team with an estimated cost tier
2. **Phase 2**: UI/UX Designer + API Contract Designer produce specs (parallel); API designer cross-checks endpoints against architecture and emits a Deviations section
3. **Phase 3**: Frontend + Backend Engineers implement (parallel batches of 4, multiple BE instances); each engineer writes an **implementation manifest** to `.fleet/implementations/<stream-name>.md`
4. **Phase 4**: Code Reviewer (if available) + Devil's Advocate per implementation stream, scoped to the files listed in that stream's manifest (parallel batches of 6)
5. **Phase 5**: Plan Compliance Auditor derives touched directories from manifests, scopes test runs accordingly, and verifies built == planned

Every review step supports a **bounded fix cycle** (up to 2 rounds): CRITICAL findings re-spawn the original author; reviewers re-run against the revised artifact, writing to versioned review files (`-v1.md`, `-v2.md`, ...). After 2 rounds, remaining criticals escalate to the user (revise / accept / abort).

Inter-agent communication uses `.fleet/` artifact directory — no direct agent messaging. `.fleet/plan.md` is the durable plan source of truth read by every agent (never inline the plan into spawn prompts).

## Conventions

- Agent definitions use YAML frontmatter: `name`, `description`, `tools`, `model`
- Thinking-heavy agents (architect, devil's advocate, compliance) use `model: opus`
- Implementation agents (engineers, designers) use `model: sonnet`
- All agents write outputs to `.fleet/` for downstream consumption
- The SKILL.md is the orchestration brain — it spawns agents via `Agent({ subagent_type: "<name>", ... })`. Never inline agent file contents into spawn prompts — this erases the agent's `tools` frontmatter boundary
- Revised source artifacts are rewritten in place with a `## Revision Log` section appended; review files are append-only and versioned via `-v<N>.md` suffix
- Stream names are kebab-case (e.g., `frontend-notifications`, `backend-auth`) and are used as the filename stem for both implementation manifests and review files

## Development

- Keep agent files focused: one role, one persona, one output format
- Every agent must specify exact output file paths in its instructions
- The `tools` frontmatter must include every tool the agent actually uses (it's a JSON array, matching the global code-reviewer format)
- When editing SKILL.md, preserve the `subagent_type` spawn pattern across all phases
- Test changes by running `/fleet` against the sample plan in `tests/fixtures/` and following the checklist in `tests/SMOKE.md`

## Dependencies

- Requires Claude Code with agent/skill support
- Phase 4 code review uses the global `code-reviewer` agent (`~/.claude/agents/code-reviewer.md`)
