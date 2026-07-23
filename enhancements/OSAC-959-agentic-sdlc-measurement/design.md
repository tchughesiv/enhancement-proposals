# Design — Agentic SDLC Measurement (OSAC-959)

> Derived from implement planning (`02-plan.md`) pending a formal enhancement proposal.

## Summary

Build a phased measurement framework for the agentic SDLC. Phase 1 delivers an eval harness for PRD and design review skills. Later phases integrate bugfix evals, collect operational metrics from Jira/GitHub, and surface trends via Org Pulse and weekly reports.

## Motivation

### Goals

- Measure whether AI-assisted PRD and design reviews are accurate and improving
- Establish repeatable eval infrastructure before building MTTR/velocity dashboards
- Integrate with existing bugfix eval harness (OSAC-516) without duplicating it
- Eventually prove agentic SDLC delivers measurable improvement over time

### Non-Goals (near-term)

- Replacing the existing FTPR dashboard
- Custom LLM fine-tuning
- Code quality metrics (coverage, complexity)

## Proposal

### Architecture

Planning-phase evals live in **`evals/review/`** inside `osac-workspace` — workspace-native tooling, not a bootstrapped component repo. Review evals run from the workspace root; they consume skills and context already present here plus `enhancement-proposals/` from `./bootstrap.sh`.

Adopt **agent-eval-harness** case/judge patterns from `osac-bugfix-eval` and [agent-eval-harness](https://github.com/opendatahub-io/agent-eval-harness). Do **not** mirror bugfix's `deps/`, `workspace-template/`, or per-case repo SHA pinning — review evals are read-only document exercises.

```
osac-workspace/
  evals/
    README.md                 # prerequisites, how to run all eval types
    review/                   # Phase 1 — planning-phase review evals
      eval-prd-review.yaml
      eval-design-review.yaml
      run-eval.sh
      cases/
        prd/*/
        design/*/
      docs/                   # measurement taxonomy, case schema
      lib/                    # optional: case validation, report aggregation
      results/                # run output (gitignored or baseline committed)
    run-all.sh                # Phase 2 — orchestrates review + external bugfix eval
  .claude/skills/prd-review/  # harness invokes these (already in workspace)
  .claude/skills/design-review/
  enhancement-proposals/      # bootstrapped — reference case documents
```

**Bugfix evals (Phase 2):** remain in external `osac-bugfix-eval` (OSAC-516). Phase 2 links via `evals/run-all.sh` and a report adapter — no duplication, no bootstrap entry.

### Scoring Model

- PRD reviews: 0-2 per dimension, /10 total (`prd-review` skill); PASS ≥7 with no zero on any dimension
- Design reviews: 0-2 per dimension, /8 total (`design-review` skill); PASS ≥5 with no zero on any dimension
- **Primary scoring:** harness-native judges in `eval.yaml` (`check` + LLM `prompt` judges, `thresholds`) using `reference-review.md` + `annotations.yaml` per case
- **Pass criteria:** verdict match, skill PASS/FAIL rules (including zero-dimension auto-fail), critical findings detected (fuzzy match)
- Optional thin Python in `evals/review/lib/` for case validation and report merging only — not a parallel scoring engine

### Output Capture

Review skills emit structured markdown to `artifacts/review-output.md` via skill execution arguments. Harness `outputs.path` must target the **containing directory** (`artifacts`), not the file itself — pointing `outputs.path` at a file silently prevented judges from reading the skill's own output (a pre-existing bug in the pinned harness v1.22.0, fixed during OSAC-2264 implementation). Judges then score the collected `review-output.md` without parsing chat stdout.

### Relationship to EP Review Bot (OSAC-1773)

The production EP Review Bot on `enhancement-proposals` ([OSAC-1773](https://redhat.atlassian.net/browse/OSAC-1773)) runs the *same* `skills/prd-review/SKILL.md` / `skills/design-review/SKILL.md` files this harness grades — confirmed via `enhancement-proposals` commit `c6df563` (OSAC-2815, merged 2026-07-16), which moved the bot's CI runner to clone `osac-workspace` fresh on every run rather than maintain a separately-authored skill. Live example: [enhancement-proposals#121](https://github.com/osac-project/enhancement-proposals/pull/121), which posted both a `prd-review` and a `design-review` bot comment matching each skill's native rubric.

This means Phase 1's harness and the bot share a skill source, not independently-evolving prompts. What still differs is the execution stack: `agent-eval-harness`/`claude-code` here vs. `agentic_ci`/GCP Vertex AI for the bot. The harness remains necessary regardless of this overlap — the bot only performs inference (runs the skill, posts whatever it outputs) with no check against a human-validated baseline. The harness is the accuracy backstop that doesn't otherwise exist. A lower-cost follow-on (out of scope for Phase 1) is a periodic check that the bot's live-posted verdicts agree with the harness's golden-case verdicts, now that same-skill-source is confirmed.

### Phasing

| Phase | Deliverable | Location |
|-------|-------------|----------|
| 1 | PRD + design review eval harness, baseline report | `evals/review/` |
| 2 | Unified reporting with `osac-bugfix-eval` | `evals/run-all.sh` + adapter |
| 3 | Jira + GitHub operational metrics | TBD — likely extend Org Pulse pipelines |
| 4 | Org Pulse trends + weekly Slack reports | Coordinate OSAC-2004 |

### Dependencies

- `agent-eval-harness` Claude Code plugin (`claude plugin install agent-eval-harness@opendatahub-skills`)
- `enhancement-proposals` (via `./bootstrap.sh`)
- `.design/context/` (loaded by review skills)
- OSAC-2004 for dashboard deployment (Phase 4) — coordinate via OSAC-2518; distinct from **UOI** (Konflux DevLake) and OSAC-2007 (already dashboards OSAC-1773's bot scores) — define the delta against both, don't duplicate
- OSAC-516 / `osac-bugfix-eval` for bugfix eval data (Phase 2, external) — lives on a personal fork, no org backup, quiet since 2026-06-03; see Epic 2 Story 2.01
- OSAC-1773 (EP Review Bot) — runs the same review skills in production CI; see "Relationship to EP Review Bot" above

## Test Plan

- Unit tests: case validation and report aggregation in `evals/review/lib/` (if needed)
- Harness judges: validated via manual baseline run and fixture cases
- Integration smoke: `evals/review/run-eval.sh --skip-execute --skip-score` from workspace root
- Full LLM eval: manual baseline run; results in `evals/review/results/baseline/`
- CI: unit tests only; LLM eval runs are local/manual

---

## Provenance

Authored: revise @ design 0.4.0 - 7b6dfe0, workspace OSAC-2264-review-harness-judges @ 6f530dcb
Phases: revise, revise

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.4.0","ai_workflows":"7b6dfe0","source_repo":"6f530dcb","source_repo_branch":"OSAC-2264-review-harness-judges","commits_behind_main":0,"commits_ahead_main":6,"main_ref":"main","phases":["revise","revise"],"authoring_modes":["skill"],"context_changed":false} -->
