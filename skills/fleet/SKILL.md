---
name: fleet
description: Orchestrated agent team that executes an implementation plan through coordinated phases. Use after /plan completes. Spawns solution architect, UI/UX designer, API contract designer, frontend/backend engineers, code reviewers, devil's advocates, and a plan compliance auditor.
version: 2.0.0
---

# Fleet — Orchestrated Agent Team Execution

Execute an implementation plan through a coordinated, multi-phase agent team with built-in review, challenge, and compliance checking.

## When to Use

- After `/plan` produces an approved implementation plan
- When the user says "execute", "build it", "kick it off", or "fleet"
- When a plan exists and needs to be turned into working code

## Prerequisites

- A completed implementation plan (from `/plan`, a markdown file, or pasted text)
- A project codebase to work in
- Optional: a global `code-reviewer` agent at `~/.claude/agents/code-reviewer.md`. If present, the global agent is used for Phase 4 code review. If absent, the bundled `fleet-code-reviewer` agent is used as a fallback — code review always runs.

## How It Works

```
Plan → Architect → Design/Contracts → Implementation → Review → Compliance
         ↓              ↓                   ↓             ↓         ↓
     Devil's Adv    Devil's Adv         Devil's Adv    Devil's Adv  Final Report
```

Every agent's output is reviewed. Every review is challenged. Reviews can trigger up to 2 bounded fix cycles before surfacing to the user. The final auditor verifies the build matches the plan.

## Artifact Layout

All inter-agent communication flows through `.fleet/`:

```
.fleet/
├── plan.md                                       # verbatim plan, written in Phase 0
├── architecture.md                               # Phase 1 output (rewritten in place on revision)
├── designs.md                                    # Phase 2 output
├── api-contracts.md                              # Phase 2 output
├── implementations/
│   ├── frontend-<stream>.md                      # one manifest per FE engineer
│   └── backend-<stream>.md                       # one manifest per BE engineer
├── reviews/
│   ├── architecture-challenge-v1.md              # v1, v2, ... per fix round
│   ├── designs-challenge-v1.md
│   ├── api-contracts-challenge-v1.md
│   ├── <stream>-code-review-v1.md
│   └── <stream>-challenge-v1.md
└── compliance-report.md                          # Phase 5 output
```

Revised source artifacts (architecture.md, designs.md, api-contracts.md, implementations/*.md) are rewritten in place and must carry a `## Revision Log` section. Review files are append-only: each fix round produces a new `-v<N>.md` file rather than overwriting the previous review.

---

## Execution Protocol

### Phase 0: Setup

**Step 1: Capture the plan.**
The plan may come from:
- The current conversation (output of a prior `/plan` invocation)
- A file path the user provides
- Text the user pastes

**Step 2: Create the artifact directory and write the plan.**
```bash
mkdir -p .fleet/reviews .fleet/implementations
```
Clean up any previous `.fleet/` from a prior run. Write the captured plan verbatim to `.fleet/plan.md`. Every downstream agent reads the plan from this file — do not inline plan text into prompts.

**Step 3: Probe for the global code-reviewer agent.**
Attempt to Read `~/.claude/agents/code-reviewer.md`. Set an internal variable:
- If the file exists and is readable: `CODE_REVIEWER_AGENT = "code-reviewer"` (use the global agent)
- Otherwise: `CODE_REVIEWER_AGENT = "fleet-code-reviewer"` (use the bundled fallback)

Phase 4 always runs code review — this variable determines which agent is used. Record the result so it can be mentioned in the final report.

**Step 4: Announce the default team and defer detailed composition to Phase 1.**

```
## Fleet Initialized

Plan captured at .fleet/plan.md
Code reviewer: [global ~/.claude/agents/code-reviewer.md detected | using bundled fleet-code-reviewer]

Default team always includes:
- Solution Architect (Phase 1)
- Devil's Advocate (runs after every artifact)
- Plan Compliance Auditor (Phase 5)

The Solution Architect will analyze the plan and codebase in Phase 1, then propose
the full implementation team (designers, engineers, stream counts). You will be
asked to approve the team after Phase 1 completes.

Proceed to Phase 1? (yes / abort)
```

Wait for user approval, then continue. Full team composition is decided by the Solution Architect in Phase 1, not here.

---

### Phase 1: Architecture & Team Composition

**Spawn:** Solution Architect agent.

```
Agent({
  description: "Solution architecture and team composition",
  subagent_type: "solution-architect",
  prompt: "Read .fleet/plan.md as the source of truth for the implementation plan. Analyze the plan and the existing codebase. Write your architecture document to .fleet/architecture.md. Your output MUST include a 'Team Composition' section listing every implementation stream, the role assigned (Frontend Engineer, Backend Engineer, UI/UX Designer, API Contract Designer), and a stable kebab-case stream name that will be used as the filename stem for manifests and reviews (e.g., frontend-notifications, backend-auth, backend-billing)."
})
```

Wait for completion. Read `.fleet/architecture.md` to confirm it was written and contains a `## Team Composition` section.

**Spawn:** Devil's Advocate to review architecture.

```
Agent({
  description: "Challenge architecture",
  subagent_type: "devils-advocate",
  prompt: "Read .fleet/plan.md and .fleet/architecture.md. Challenge the architecture and proposed team composition. Write your review to .fleet/reviews/architecture-challenge-v1.md."
})
```

Wait for completion. Read the review. Apply the **Fix Cycle Protocol** (see below) if CRITICAL findings exist.

**Present the team to the user for approval.**

After any fix cycles are resolved, read the final `## Team Composition` section from `.fleet/architecture.md`. Before presenting, compute a rough cost estimate so the user can make an informed decision.

**Estimated invocation count** (baseline, no fix cycles):

```
baseline = 2                       # Phase 1: architect + devil's advocate
         + 2 * D                   # Phase 2: D designers (each + devil's advocate)
         + E                       # Phase 3: E implementation engineers
         + E * 2                   # Phase 4: E * (devil's advocate + code-reviewer) — code review always runs
         + 1                       # Phase 5: compliance auditor

where:
  D = number of activated designers (0, 1, or 2)
  E = total implementation engineers (frontend + backend streams)
```

Add a fix-cycle buffer of up to `+4` invocations per artifact that hits the fix cycle (worst case: 2 rounds × 2 re-spawns = 4 extra). Assume ~1 artifact hits the fix cycle on average for the range estimate.

**Cost tier** (signal only — actual cost varies with plan complexity and codebase size):

| Invocations | Tier | Notes |
|-------------|------|-------|
| ≤ 10        | LOW     | single-stream work, minimal review |
| 11–18       | MEDIUM  | typical feature — FE + 1-2 BE streams |
| 19–28       | HIGH    | multi-module refactor or wide FE surface |
| > 28        | VERY HIGH | consider splitting the plan |

Present the team:

```
## Fleet Team Composition (from Solution Architect)

| Role | Stream Name | Scope |
|------|-------------|-------|
| UI/UX Designer       | design-ui           | [scope from architecture.md] |
| API Contract Designer| design-api          | [scope from architecture.md] |
| Frontend Engineer    | frontend-<stream>   | [scope] |
| Backend Engineer     | backend-<stream-a>  | [scope] |
| Backend Engineer     | backend-<stream-b>  | [scope] |

Every implementation stream will be reviewed by Devil's Advocate and, if available,
the global code-reviewer.

### Estimated Cost
- Baseline invocations: **[N]** (Phase 1-5, no fix cycles)
- With fix-cycle buffer: **[N] – [N+4]**
- Cost tier: **[LOW | MEDIUM | HIGH | VERY HIGH]**
- Thinking-heavy roles (architect, devil's advocate, compliance) run on opus; implementation roles run on sonnet.
- Code-reviewer: [global agent | bundled fleet-code-reviewer]

Proceed? (yes / modify / abort)
```

Wait for user approval. If the user modifies the team, update `.fleet/architecture.md` accordingly (or re-spawn the architect with the user's feedback) before continuing.

---

### Phase 2: Contracts & Design (Parallel)

Based on the team composition in `.fleet/architecture.md`, spawn applicable designers **in parallel**.

**If UI/UX Designer is activated:**
```
Agent({
  description: "UI/UX design specs",
  subagent_type: "uiux-designer",
  prompt: "Read .fleet/plan.md and .fleet/architecture.md. Produce design specifications for your assigned work stream(s). Write your specs to .fleet/designs.md."
})
```

**If API Contract Designer is activated:**
```
Agent({
  description: "API contract design",
  subagent_type: "api-contract-designer",
  prompt: "Read .fleet/plan.md and .fleet/architecture.md. Produce API contracts for your assigned work stream(s). Write your contracts to .fleet/api-contracts.md. Cross-check every endpoint against the work streams in .fleet/architecture.md — if your contract adds, removes, or reshapes an endpoint relative to the architect's list, document the deviation in the 'Deviations from Architecture' section of your output."
})
```

Wait for all Phase 2 agents to complete.

**Spawn Devil's Advocates in parallel** — one per Phase 2 artifact:

```
Agent({
  description: "Challenge UI designs",
  subagent_type: "devils-advocate",
  prompt: "Read .fleet/plan.md, .fleet/architecture.md, and .fleet/designs.md. Challenge the design spec. Write your review to .fleet/reviews/designs-challenge-v1.md."
})

Agent({
  description: "Challenge API contracts",
  subagent_type: "devils-advocate",
  prompt: "Read .fleet/plan.md, .fleet/architecture.md, and .fleet/api-contracts.md. Challenge the API contracts, especially any 'Deviations from Architecture'. Write your review to .fleet/reviews/api-contracts-challenge-v1.md."
})
```

Apply the **Fix Cycle Protocol** per artifact if CRITICAL findings exist.

---

### Phase 3: Implementation (Parallel, Batched)

Read `.fleet/architecture.md` to get the work stream assignments and stream names. Spawn implementation agents in batches of up to **4 parallel agents**. Dispatch the next batch after the current one completes.

**Frontend Engineer(s):** For each frontend stream:
```
Agent({
  description: "Frontend: <stream-name>",
  subagent_type: "frontend-engineer",
  prompt: "Read .fleet/plan.md, .fleet/architecture.md, .fleet/designs.md, and .fleet/api-contracts.md. Your assigned stream is '<stream-name>' — implement only that stream. When done, write an implementation manifest to .fleet/implementations/<stream-name>.md listing every file created, every file modified, every test added, any deviations from the design/contract specs, and any open questions."
})
```

**Backend Engineer(s):** For each independent backend module:
```
Agent({
  description: "Backend: <stream-name>",
  subagent_type: "backend-engineer",
  prompt: "Read .fleet/plan.md, .fleet/architecture.md, and .fleet/api-contracts.md. Your assigned stream is '<stream-name>' — implement only that module. Do not import from other modules being built in parallel. When done, write an implementation manifest to .fleet/implementations/<stream-name>.md listing every file created, every file modified, every test added, any deviations from the contract, and any open questions."
})
```

Wait for each batch to complete before starting the next. Wait for all implementation agents to complete before advancing to Phase 4.

**Validation:** After all implementation agents complete, verify that `.fleet/implementations/<stream-name>.md` exists for every stream. If any manifest is missing, re-spawn that engineer with an explicit instruction to write the manifest.

---

### Phase 4: Review (Parallel, Batched)

For **each** implementation stream, spawn a review pair. Dispatch reviewers in batches of up to **6 parallel agents** (so up to 3 streams' review pairs at a time). Wait for each batch before dispatching the next.

For each stream, read `.fleet/implementations/<stream-name>.md` to identify the exact files the engineer touched. Pass the manifest path into both reviewer prompts.

In the spawn templates below, `<stream-name>` is a placeholder — substitute the actual kebab-case stream identifier from `.fleet/architecture.md` (e.g., `frontend-notifications`, `backend-auth`) into both the `description` field and every path in the `prompt` field before spawning. Do not pass the literal string `<stream-name>` to the agent.

**Code Reviewer** (always runs — uses `CODE_REVIEWER_AGENT` from Phase 0):
```
Agent({
  description: "Code review: <stream-name>",
  subagent_type: CODE_REVIEWER_AGENT,
  prompt: "Review the code changes for the '<stream-name>' work stream. The implementation manifest at .fleet/implementations/<stream-name>.md lists every file created and modified by this engineer — constrain your review to those files only. Cross-check against:\n- .fleet/api-contracts.md (if API work)\n- .fleet/designs.md (if frontend work)\n- .fleet/architecture.md (overall design)\n\nFocus on correctness, contract compliance, error handling, and performance. Write findings to .fleet/reviews/<stream-name>-code-review-v1.md."
})
```

**Devil's Advocate:**
```
Agent({
  description: "Challenge: <stream-name>",
  subagent_type: "devils-advocate",
  prompt: "Review the implementation for the '<stream-name>' work stream. The implementation manifest at .fleet/implementations/<stream-name>.md lists the files the engineer created and modified — Read those files to understand what was built, then challenge that code specifically on correctness, security, scalability, edge cases, test quality, and (for FE work) accessibility. Also read .fleet/plan.md, .fleet/architecture.md, .fleet/designs.md (if present), and .fleet/api-contracts.md (if present) for spec context. Write your review to .fleet/reviews/<stream-name>-challenge-v1.md."
})
```

Wait for all Phase 4 reviews to complete.

**Apply the Fix Cycle Protocol** per stream if CRITICAL findings exist. The fix cycle re-spawns the original implementation engineer with the review files as input.

**Surface findings:** After all fix cycles conclude, collect all remaining CRITICAL and HIGH findings from the highest-version review files and present them to the user.

---

### Phase 5: Compliance Audit

**Spawn:** Plan Compliance Auditor.

```
Agent({
  description: "Plan compliance audit",
  subagent_type: "plan-compliance-auditor",
  prompt: "Read .fleet/plan.md as the source of truth. Read .fleet/architecture.md, .fleet/designs.md (if present), .fleet/api-contracts.md (if present), and every file in .fleet/implementations/. For reviews, only read the highest-numbered version of each file in .fleet/reviews/ (e.g., if designs-challenge-v1.md and designs-challenge-v2.md both exist, read v2). Derive the set of touched directories from the implementation manifests and scope test runs to those directories where the runner supports it. Write your report to .fleet/compliance-report.md."
})
```

Wait for completion. Read `.fleet/compliance-report.md`.

**Present the final report to the user:**

```
## Fleet Execution Complete

### Compliance: [PASS | PASS WITH NOTES | FAIL]

| Metric | Count |
|--------|-------|
| Requirements done | X/Y |
| Requirements partial | X |
| Requirements missing | X |
| Critical review findings open | X |
| Code-reviewer | [global agent | bundled fleet-code-reviewer] |

### What Was Built
- [Summary from architecture + implementation manifests]

### Open Items
- [Any PARTIAL/MISSING requirements]
- [Unresolved CRITICAL findings]

### Artifacts
- .fleet/plan.md
- .fleet/architecture.md
- .fleet/designs.md
- .fleet/api-contracts.md
- .fleet/implementations/
- .fleet/reviews/
- .fleet/compliance-report.md
```

---

## Fix Cycle Protocol

After every review step (Phase 1, 2, and 4), check for CRITICAL findings in the review file(s) just produced. If any exist, enter a bounded fix cycle:

**Iteration limit: 2 rounds per artifact.**

### Round N (1 ≤ N ≤ 2)

1. **Re-spawn the original author** of the artifact (solution-architect, uiux-designer, api-contract-designer, frontend-engineer, or backend-engineer) with the review file(s) from the current version as additional input. The prompt must instruct the author to:
   - Read the review file(s) (pass the exact paths, e.g., `.fleet/reviews/architecture-challenge-v<N>.md`)
   - Address every CRITICAL finding and, where practical, HIGH findings
   - Rewrite the source artifact in place (`.fleet/architecture.md`, `.fleet/designs.md`, `.fleet/api-contracts.md`, or `.fleet/implementations/<stream>.md`) — not to a new file
   - Append or update a `## Revision Log` section at the bottom of the rewritten artifact with one entry per round, listing each CRITICAL/HIGH finding from the previous review and how it was addressed
2. **Re-run the reviewers** against the revised artifact, writing to the next version suffix:
   - Devil's Advocate writes to `.fleet/reviews/<artifact>-challenge-v<N+1>.md`
   - Code-reviewer (Phase 4 only, if available) writes to `.fleet/reviews/<stream>-code-review-v<N+1>.md`
   - Reviewer prompts must include the previous review path so they can verify each prior finding was addressed
3. If the new review file has no CRITICAL findings, exit the fix cycle for this artifact. Proceed.
4. If CRITICAL findings remain and `N < 2`, increment `N` and repeat.
5. If CRITICAL findings remain after round 2, **stop the cycle** and surface the remaining findings to the user with three options:
   - `revise` — run one more manual fix round (user provides direction)
   - `accept` — proceed with findings documented as open items
   - `abort` — stop the fleet execution

The auditor in Phase 5 must read only the **highest-numbered** review version per artifact — older versions are historical.

### Fix Cycle Spawn Template

Use this concrete template for every re-spawn, substituting the agent name, artifact path, and review version. The example below shows round 2 for the architecture artifact — adapt the `subagent_type`, paths, and version numbers for other artifacts and rounds.

```
// Re-spawn the original author
Agent({
  description: "Fix cycle round 2: architecture",
  subagent_type: "solution-architect",
  prompt: "The architecture at .fleet/architecture.md has CRITICAL findings in .fleet/reviews/architecture-challenge-v1.md. Read .fleet/plan.md, the current architecture, and the review file. Address every CRITICAL finding and, where practical, HIGH findings. Rewrite .fleet/architecture.md in place — do not write to a new path. At the bottom of the rewritten file, append or update a '## Revision Log' section with a 'Round 2' entry listing each CRITICAL/HIGH finding from the review and a one-line description of how you addressed it."
})

// Re-run the reviewer with version bump
Agent({
  description: "Re-challenge architecture after round 2 fix",
  subagent_type: "devils-advocate",
  prompt: "Read .fleet/plan.md and the revised .fleet/architecture.md. Also read the previous review at .fleet/reviews/architecture-challenge-v1.md — verify whether each CRITICAL and HIGH finding from that review has been addressed in the revision. Write your new review to .fleet/reviews/architecture-challenge-v2.md. Include a 'Previous Findings Status' section at the top mapping each prior finding to one of: RESOLVED, PARTIALLY RESOLVED, NOT ADDRESSED."
})
```

For Phase 4 implementation fixes, add the code-reviewer re-spawn (using `CODE_REVIEWER_AGENT`) in parallel with the devil's advocate, also versioned to `<stream-name>-code-review-v<N+1>.md`, and include the previous review path in its prompt.

### Implementation Engineer Fix Cycle Note

For Phase 4 implementation fixes, the re-spawn prompt for frontend-engineer or backend-engineer must include: "Your implementation manifest at `.fleet/implementations/<stream-name>.md` lists the files you own. Fix the issues raised in the review files, update any test or implementation files as needed, and update your manifest's file lists and Revision Log to reflect the changes."

---

## Agent Reference

| Agent | subagent_type | Phase | Tools | Model |
|-------|---------------|-------|-------|-------|
| Solution Architect      | solution-architect      | 1     | Read, Grep, Glob, Write            | opus   |
| UI/UX Designer          | uiux-designer           | 2     | Read, Grep, Glob, Write            | sonnet |
| API Contract Designer   | api-contract-designer   | 2     | Read, Grep, Glob, Write            | sonnet |
| Frontend Engineer       | frontend-engineer       | 3     | Read, Grep, Glob, Write, Edit, Bash| sonnet |
| Backend Engineer        | backend-engineer        | 3     | Read, Grep, Glob, Write, Edit, Bash| sonnet |
| Devil's Advocate        | devils-advocate         | 1-4   | Read, Grep, Glob, Write            | opus   |
| Code Reviewer (global)  | code-reviewer           | 4     | Read, Grep, Glob, Bash             | sonnet |
| Code Reviewer (bundled) | fleet-code-reviewer     | 4     | Read, Grep, Glob, Write            | sonnet |
| Plan Compliance Auditor | plan-compliance-auditor | 5     | Read, Grep, Glob, Bash, Write      | opus   |

## Concurrency Notes

- **Phase 3 (implementation)**: Dispatch engineers in batches of up to **4** parallel agents. Wait for each batch before starting the next.
- **Phase 4 (review)**: Dispatch reviewers in batches of up to **6** parallel agents. Wait for each batch before starting the next.
- **Phase 2 (design)**: Run both designers in parallel (max 2).
- Thinking-heavy roles (Architect, Devil's Advocate, Compliance) use **opus** for depth.
- Implementation roles (FE/BE Engineers, Designers) use **sonnet** for speed and cost.
- Code Reviewer always runs — uses the global `code-reviewer` agent if installed at `~/.claude/agents/code-reviewer.md`, otherwise falls back to the bundled `fleet-code-reviewer`.
- Artifact files in `.fleet/` are the communication channel between phases — no direct inter-agent messaging.

## Cleanup

After the user is satisfied with the results:
```bash
# Keep artifacts for reference
git add .fleet/ && git commit -m "chore: fleet execution artifacts"

# Or clean up
rm -rf .fleet/
```

The `.fleet/` directory is ephemeral by design. Add it to `.gitignore` if you don't want to track artifacts.

## Examples

### Minimal invocation
```
User: /plan Add user notifications when markets resolve
[plan is created and approved]

User: /fleet
[Fleet reads the plan, composes team, asks for approval, executes all phases]
```

### With explicit plan file
```
User: /fleet plan.md
[Fleet reads plan.md as the plan input]
```

### Modifying the team
```
User: /fleet
[Fleet presents team composition]

User: modify: skip UI/UX designer, add a second FE engineer
[Fleet adjusts team and re-presents for approval]
```
