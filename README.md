# claude-fleet

Orchestrated agent team plugin for Claude Code. Turns an implementation plan into working code through coordinated phases — architecture, design, API contracts, parallel implementation, code review, devil's advocate challenges, and plan compliance auditing.

## What It Does

After you create an implementation plan (via `/plan` or any other method), `/fleet` kicks off a coordinated team:

```
Plan ──→ Solution Architect ──→ UI/UX Designer ║ API Contract Designer
            (decides team)              ↓                  ↓
                                Frontend Engineer(s) ║ Backend Engineer(s)
                                        ↓                  ↓
                                  Code Reviewer + Devil's Advocate (per stream)
                                                ↓
                                    Plan Compliance Auditor
```

Every agent's output is reviewed. Every review is challenged. When a review finds CRITICAL issues, the original author is re-spawned for up to 2 bounded fix cycles before the skill asks the user to decide. The final auditor verifies the build matches the plan and runs the tests scoped to the modules the engineers actually touched.

## Agent Roles

| Role | When Active | What They Do |
|------|-------------|-------------|
| **Solution Architect** | Always | Analyzes plan, decomposes into work streams, assigns to roles |
| **UI/UX Designer** | Frontend work | Component hierarchy, layout, states, accessibility |
| **API Contract Designer** | API work | Endpoint specs, request/response schemas, error handling |
| **Frontend Engineer** | Frontend work | Implements components from design specs + API contracts |
| **Backend Engineer** | Backend work | Implements server logic, DB, endpoints (multiple for modular work) |
| **Code Reviewer** | Per implementation | Reviews code for quality, security, correctness |
| **Devil's Advocate** | Per agent output | Challenges assumptions, finds weaknesses, pokes holes |
| **Plan Compliance Auditor** | Always (last) | Verifies everything built matches the original plan |

## Installation

### Option 1: Add as marketplace (recommended)

If this plugin is published to a GitHub repository:

1. Add to your Claude Code settings (`~/.claude/settings.json`):

```json
{
  "extraKnownMarketplaces": {
    "claude-fleet": {
      "source": {
        "source": "github",
        "repo": "YOUR_USERNAME/claude-fleet"
      }
    }
  }
}
```

2. Install the plugin:

```
/plugin install claude-fleet@claude-fleet
```

3. Enable it in settings if not auto-enabled.

### Option 2: Manual installation

Clone the repo and copy into your Claude Code directories:

```bash
# Clone
git clone https://github.com/YOUR_USERNAME/claude-fleet.git

# Copy agents
cp claude-fleet/agents/*.md ~/.claude/agents/

# Copy skill
cp -r claude-fleet/skills/fleet ~/.claude/skills/fleet
```

### Option 3: Local plugin (development)

Point Claude Code at the local directory:

```json
{
  "extraKnownMarketplaces": {
    "claude-fleet": {
      "source": {
        "source": "local",
        "path": "/path/to/claude-fleet"
      }
    }
  }
}
```

## Prerequisites

- **Claude Code** with agent/skill support
- **Optional: global `code-reviewer` agent** at `~/.claude/agents/code-reviewer.md`. Ships with [Everything Claude Code](https://github.com/affaan-m/everything-claude-code). Fleet probes for this file in Phase 0 and records a `HAS_GLOBAL_CODE_REVIEWER` flag. If absent, Phase 4 code review is skipped automatically and only the devil's advocate runs against each implementation — you'll see the skip noted in the final report.

## Usage

### Basic flow

```
# 1. Create a plan
/plan Add real-time notifications when markets resolve

# 2. Approve the plan
yes

# 3. Execute with fleet
/fleet
```

### With an existing plan file

```
/fleet plan.md
```

### What happens

1. **Phase 0** — Fleet writes your plan verbatim to `.fleet/plan.md`, probes for the global code-reviewer, and asks you to confirm the default base team.
2. **Phase 1** — The Solution Architect reads the plan and codebase, then writes `.fleet/architecture.md` including a `## Team Composition` section that names every implementation stream (e.g., `frontend-notifications`, `backend-auth`). Devil's advocate challenges the architecture. You then approve the proposed team with an estimated cost tier.
3. **Phase 2** — UI/UX Designer and API Contract Designer run in parallel, each challenged by devil's advocate. The API designer cross-checks endpoints against the architecture and flags any deviations.
4. **Phase 3** — Frontend and backend engineers implement their assigned streams in parallel (batches of up to 4). Each engineer practices strict TDD and writes an **implementation manifest** to `.fleet/implementations/<stream-name>.md` listing every file created, every file modified, every test added, deviations, and open questions.
5. **Phase 4** — For each stream, a code reviewer (if available) and a devil's advocate review the manifest-scoped files in parallel (batches of up to 6).
6. **Phase 5** — The Plan Compliance Auditor derives the set of touched directories from the manifests and runs the test suite scoped to those directories, then verifies every plan requirement was implemented and tested.

At any review step, if CRITICAL findings are found, fleet runs up to 2 bounded fix cycles (re-spawning the original author with the review as input, then re-running the reviewers at versioned paths like `-v2.md`, `-v3.md`). After 2 rounds, remaining CRITICAL issues are surfaced to you to revise, accept, or abort.

### Artifacts

All inter-agent communication goes through `.fleet/` in the project root:

```
.fleet/
├── plan.md                                  # Verbatim plan, written in Phase 0
├── architecture.md                          # Solution architect output (incl. Team Composition)
├── designs.md                               # UI/UX designer output
├── api-contracts.md                         # API contract designer output (incl. Deviations from Architecture)
├── implementations/
│   ├── frontend-<stream>.md                 # One manifest per FE engineer
│   └── backend-<stream>.md                  # One manifest per BE engineer
├── reviews/
│   ├── architecture-challenge-v1.md         # -v1, -v2, ... one file per fix round
│   ├── designs-challenge-v1.md
│   ├── api-contracts-challenge-v1.md
│   ├── <stream-name>-code-review-v1.md
│   └── <stream-name>-challenge-v1.md
└── compliance-report.md                     # Final audit
```

Source artifacts (architecture.md, designs.md, api-contracts.md, implementations/*.md) are rewritten in place on revision and carry a `## Revision Log` section. Review files are append-only — each fix round produces a new `-v<N>.md` file.

`.fleet/` is gitignored by default.

## Cost Considerations

- **Thinking roles** (Architect, Devil's Advocate, Compliance) use **Opus** for depth
- **Implementation roles** (Engineers, Designers) use **Sonnet** for speed and cost
- **Batched parallelism**: Phase 2 runs up to 2 designers, Phase 3 up to 4 engineers per batch, Phase 4 up to 6 reviewers per batch
- **Estimated invocations** (baseline, no fix cycles): `2 + 2*D + E + E*(1+C) + 1`
  where `D` = activated designers, `E` = implementation engineers, `C` = 1 if global code-reviewer is installed else 0
- **Typical runs**: FE + 2 BE modules with both designers and the code-reviewer ≈ 16 baseline invocations, up to ~20 with one fix cycle
- Fleet prints a cost-tier estimate (LOW / MEDIUM / HIGH / VERY HIGH) at the Phase 1 approval step so you can decide before spending

## License

MIT
