# Codebase Audit → Plan → Fix → Codify

> Paste below into Claude Code at the repo root.

---

## Task

Audit this repo for foundational problems that compound over time, then plan fixes, execute them, and codify the knowledge so any cold agent or engineer is immediately productive.

**Priorities, strict:** (1) maintainability, (2) operability, (3) performance — order-of-magnitude wins only. Breaking changes allowed; don't soften advice for compatibility.

**Gates:** AUDIT → human triages findings (fix/defer/reject) → PLAN → human approves each plan → FIX → CODIFY. No code during audit/planning. No fix without an approved plan.

**State on disk:** maintain `audit/state.json` + all findings/plans/results in `audit/` as produced. Stages read from disk, not conversation memory. Crashed runs resume from state.

## Model routing

**Fable (you)** = orchestrator: synthesis, prioritization, conflict resolution, deviation rulings. Never read files a subagent can summarize. **Opus** = planning, pre-mortem, adversarial review, anything needing many modules in mind at once. **Sonnet** = deep dives, fix execution, doc drafting. **Haiku** = scans, greps, quote checks. *Opus thinks, Sonnet builds, Haiku fetches, Fable decides.*

Subagents spawn cold — prepend every spawn: priorities, finding schema, quote rule, exclusion list, task rules. Missing preamble = prose essays instead of findings.

## Finding schema (invalid without verbatim quote)

```json
{"id":"F-001","location":"path/file.ext:line",
 "quote":"<verbatim, ≤15 lines, copied BEFORE writing analysis>",
 "what":"","why_it_matters":"","severity":"Critical|High|Medium|Low",
 "confidence":0.0,"verified":false,"survived_adversarial_review":null,
 "fix_direction":"","effort":"S|M|L",
 "category":"architecture|code_health|testing|operability|dependencies|docs|security|performance"}
```

## STAGE 0 — Preconditions (abort if unmet)

- Clean tree, expected branch; record SHA — pin it in report header and every finding (diffs are only valid against a SHA).
- Run test suite **twice**. Red/flaky baseline → fixing the suite is forced roadmap item #1.
- Exclusion list (generated, vendored, lockfiles, migrations, artifacts) → passed to all scanners.
- Inventory and run available deterministic tooling (coverage, dep audit, complexity, git-churn stats, linters); LLM agents interpret tool output, never reproduce what a tool measures exactly.

## STAGE 1 — Audit

1. **Hotspots, dual independent runs (Haiku):** two agents, same instructions, no shared context. Both found → deep dive. One found → deep dive only if Critical/High, else Unverified.
2. **Deep dives (Sonnet):** per hotspot — coupling, hidden state, error handling, testability, blast radius.
3. **Empirical tests (Sonnet):**
   - *Change-trace:* 3 invented feature requests (small/medium/cross-cutting); exact files each must touch; report file/module counts + unrelated code dragged in.
   - *3am incident:* simulate a realistic outage; debug/recover using only what the repo provides; every blind spot = finding.
   - *Onboarding trace:* docs-only — run locally, run tests, ship a one-line change; every tribal-knowledge step = finding.
4. **Pre-mortem (Opus, fresh):** "6 months out this codebase caused a serious outage and a missed deadline in one quarter — write the postmortem explaining how the code made both inevitable." Each causal claim → schema finding or discard.
5. **Verification gate (Haiku, fresh, ~10 findings each):** re-open file, confirm quote verbatim + behavior as described, set confidence. ≥0.9 → eligible; <0.9 → one retry with objection, else Unverified; quote absent → delete finding, increment deletion counter.
6. **Adversarial review (Opus, fresh, repo access):** sole job — kill Critical/High findings; argue wrong / exaggerated / intentional design, citing code. Survives → promote. Rebuttal wins → downgrade/park. Standoff → keep with counter-argument attached.
7. **Synthesis (Fable):** collapse symptoms to root causes (one flaw, N instances = one finding). **Hard cap: 15 main findings**; rest to ranked appendix.

**Scoring anchors** (most real codebases land 4–7): 2 = changes break unrelated features · 4 = fragile, tribal knowledge, tests don't protect · 6 = safe changes with effort · 8 = new hire ships safely week one · 10 = codebase teaches its own conventions. Score architecture, code health, testing, operability, dependencies, docs, security — cite finding IDs + empirical data. Overall grade weighted to priorities 1–2, one-line verdict.

**Report:** exec summary ≤10 sentences · anchored scores · ≤15 findings (Critical first, standoff counter-arguments included) · empirical results · major perf items · leverage-ordered roadmap (effort, inter-item dependencies, breaking-change flags; lead with the 2–3 unlocking changes) · what's good/preserve · appendices: overflow (ranked), unverified, deletion count (hallucination meter), `findings.json` (for diffing future audits).

⛔ Human triage. Fix-marked items only proceed.

## STAGE 2 — Plan (Opus, one agent per item; batch trivially related)

Each plan: **scope** (exact files; explicit out-of-scope) · **approach** + alternatives rejected · **sequencing** (independently verifiable steps) · **breaking changes** — what breaks, who consumes this, and have they been told (real answer, not formality) · **test strategy** — behavior-pinning tests written *before* the change; characterization tests for risky/undertested areas · **rollback** + checkpoint commit boundaries · **risk** — blast radius, worst case, early-warning signs.

Fable cross-reviews all plans: overlapping files → merge or sequence; order = unlocking changes first.

⛔ Human approves/edits/rejects each plan.

## STAGE 3 — Fix (Sonnet, one agent per plan)

- Work on branch `audit/fixes-<date>`; **one PR per fix** (finding ID + plan linked in description) — human reviews diffs, not just plans.
- **Serial execution** in Fable's order, each fix from the previous fix's final commit. Never parallel writes to one repo.
- Pinning tests first; full suite green at every checkpoint; commits reference finding IDs.
- **Deviation rule:** reality contradicts plan → STOP, report to Fable. Minor → Fable approves; structural → back to Opus planner; scope change → back to human.
- No drive-by refactors; mid-fix discoveries become new schema findings for next cycle, not extra edits.
- After each fix, fresh Haiku re-runs the relevant empirical test — improvement must be measured, not claimed.

**Output:** per-item plan-vs-actual + deviations + commits · before/after empirical numbers · updated `findings.json` · new findings · next-audit focus.

## STAGE 4 — Codify (Sonnet drafts from audit/fix artifacts; Fable verifies against final code)

**`CLAUDE.md`** (≤150 lines, link out): commands (build/test/single-test/lint/run/deploy) · architecture in ~10 lines · conventions *actually in force* (from evidence) · gotchas from deep-dive + deviation reports · hard boundaries agents must never touch unasked.

**`DESIGN.md`**: module boundaries + intended dependency direction · key decisions with rationale — **including every adversarial-review standoff** ("looks wrong, intentional because…") so future agents don't re-flag or "fix" deliberate design · known debt = unfixed findings with severity · change guide for the 3 change-trace archetypes.

**`RUNBOOK.md`** (if ops gaps found): logs/metrics locations, health checks, rollback procedure, identified failure modes + recovery.

Rules: describe what *is*; aspirations go under known debt. Every non-obvious claim traces to a finding ID, commit, or test result.

**Validation:** fresh zero-context Haiku re-runs the onboarding trace using only the new docs (run, test, ship one-line change, "where would feature X live?"). Every stumble = doc defect; fix and re-run until clean. Report before/after onboarding diff — the maintainability win, quantified.

## Follow-up

Report ends asking human to mark each finding agree/disagree/unsure; next run re-audits only disputed/unsure with objections injected into verification + adversarial passes. Day-to-day agents append learned gotchas to CLAUDE.md; full pipeline re-runs on a cadence, diffed via `findings.json`.

## Hard rules

No finding without a verbatim quote — **except secrets: location + pattern type only, never the value**. Existing repo docs are claims to verify, not evidence; doc/code mismatch is itself a finding. No code during audit/planning. No fix without an approved plan. No unanchored scores. No symptom-spam. No drive-by refactors. No perf work crowding priorities 1–2. Docs don't ship until a cold agent passes the onboarding trace with them.
