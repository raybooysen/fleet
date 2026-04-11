# Fleet Smoke Test

Manual end-to-end verification that a change to `skills/fleet/SKILL.md` or any agent under `agents/` hasn't broken the plugin. Run after any non-trivial edit, especially edits that touch spawn syntax, artifact paths, phase boundaries, or the fix cycle protocol.

## Prerequisites

- A throwaway test project with some existing code (any stack — the plan fixture is intentionally stack-agnostic in how it describes requirements). A small TypeScript / Next.js / Django / Rails app is ideal.
- The fleet plugin installed in Claude Code (via local marketplace or copied to `~/.claude`).
- Optional: global `code-reviewer` at `~/.claude/agents/code-reviewer.md`. Run the smoke test **twice** — once with the global code-reviewer present, once with it temporarily renamed — to exercise both branches of the Phase 0 probe.

## Setup

1. `cd` into the throwaway project.
2. Confirm a clean working tree: `git status` shows no changes.
3. Confirm no leftover `.fleet/` directory: `ls -d .fleet 2>/dev/null` should return nothing.
4. Copy the fixture plan into place (or pass its path directly to fleet):
   ```
   cp /path/to/claude-fleet/tests/fixtures/plan-notifications.md ./plan.md
   ```

## Run

Invoke fleet with the fixture:

```
/fleet plan.md
```

## Verification Checklist

Walk through each item during or after the run. Every box should be checkable.

### Phase 0

- [ ] `.fleet/plan.md` was created and contains the verbatim fixture contents
- [ ] `.fleet/implementations/` and `.fleet/reviews/` directories exist
- [ ] Fleet printed a `Global code-reviewer:` status line reflecting whether `~/.claude/agents/code-reviewer.md` is present
- [ ] Fleet asked for confirmation before proceeding to Phase 1

### Phase 1

- [ ] Solution Architect ran and wrote `.fleet/architecture.md`
- [ ] `.fleet/architecture.md` contains a `## Team Composition` section naming every implementation stream in kebab-case
- [ ] Stream names follow the pattern `frontend-<slug>` / `backend-<slug>` / `design-ui` / `design-api`
- [ ] Devil's advocate ran and wrote `.fleet/reviews/architecture-challenge-v1.md`
- [ ] Fleet presented the team composition table with an **Estimated Cost** block showing: baseline invocation count, fix-cycle range, cost tier, code-reviewer status
- [ ] Fleet asked for user approval before Phase 2

### Phase 2

- [ ] Both designers ran (in parallel, not sequentially — check the run log)
- [ ] `.fleet/designs.md` exists with at least one component spec including `data-testid` selectors and an `Accessibility` subsection
- [ ] `.fleet/api-contracts.md` exists with all three endpoints from the fixture
- [ ] `.fleet/api-contracts.md` contains a `## Deviations from Architecture` section (even if empty)
- [ ] `.fleet/reviews/designs-challenge-v1.md` and `.fleet/reviews/api-contracts-challenge-v1.md` both exist

### Phase 3

- [ ] One frontend engineer and one backend engineer ran (or more, depending on how the architect decomposed the work)
- [ ] `.fleet/implementations/<stream-name>.md` exists for every stream the architect assigned
- [ ] Each manifest contains sections: Files Created, Files Modified, Tests Added, Deviations, Open Questions, Test Run Evidence
- [ ] The frontend manifest includes an `API Mocking Strategy` section
- [ ] The backend manifest includes an `API Contract Coverage` section listing every endpoint from the contract
- [ ] `git status` shows new source files AND new test files — tests were written alongside implementation
- [ ] Test Run Evidence records the actual command and counts (not a placeholder)

### Phase 4

- [ ] For each stream, `.fleet/reviews/<stream-name>-challenge-v1.md` exists
- [ ] If `HAS_GLOBAL_CODE_REVIEWER` was true: `.fleet/reviews/<stream-name>-code-review-v1.md` also exists
- [ ] If `HAS_GLOBAL_CODE_REVIEWER` was false: fleet printed a skip message, and no code-review files exist
- [ ] Each review references the manifest and only critiques files listed in it

### Phase 5

- [ ] `.fleet/compliance-report.md` exists
- [ ] Report includes: Summary, Verdict (PASS/PASS WITH NOTES/FAIL), Test Results, **Test Scope**, Test Completeness, Accessibility Audit, Requirement Checklist
- [ ] Test Scope section documents which directories were scoped (from implementation manifests)
- [ ] Every acceptance criterion from the fixture plan appears in the Requirement Checklist
- [ ] The final user-facing report shows the `Code-reviewer status` row reflecting the Phase 0 probe result

### Fix Cycle (only if reviews surfaced CRITICAL findings)

- [ ] Fleet re-spawned the original author for the affected artifact
- [ ] The revised artifact has a `## Revision Log` section with a `Round 1` (or higher) entry
- [ ] A new review file exists with `-v2.md` suffix
- [ ] The new review has a `Previous Findings Status` section mapping v1 findings to RESOLVED / PARTIALLY RESOLVED / NOT ADDRESSED
- [ ] After 2 rounds, remaining criticals were surfaced to the user with revise / accept / abort options

## Correctness Spot-Checks

After the run completes, sanity-check the generated code:

- [ ] New tests actually run and at least some pass (`npm test`, `pytest`, or whatever the project uses)
- [ ] `<button>` is used for clickable elements — `grep -r '<div[^>]*onClick' src/` returns nothing notification-related
- [ ] The bell component has a `data-testid` attribute matching one in `.fleet/designs.md`
- [ ] Integration tests hit a real DB (look for imports of the project's test DB helper, not `jest.mock('db')`)

## Cleanup

```
rm -rf .fleet/ plan.md
git restore .
```

## Known Non-Issues

- First Phase 0 `mkdir` may emit a harmless notice about re-creating `.fleet/` if an earlier run left one behind. If cleanup was performed, ignore.
- Devil's advocate may report 0 findings on trivial artifacts — that is acceptable per the agent's rules (don't invent problems).
- Total wall-clock time is dominated by model latency, not tool I/O. A clean run of this fixture typically takes 5–15 minutes depending on parallelism and plan complexity.

## Reporting Regressions

If any checkbox fails, open an issue with:
1. Which checkbox
2. The relevant `.fleet/` artifact (or its absence)
3. The git SHA of `claude-fleet` you tested against
