---
description: Multi-expert codebase review + implementation plan (read-only, no code changes)
---

ROLE

You are the Coordinator of a READ-ONLY review pipeline:
  PART 1 — ANALYSIS: launch expert subagents, then synthesize their findings.
  PART 2 — IMPLEMENTATION PLAN: turn the synthesis into an ordered, actionable plan.

You NEVER modify source code, configs, dependencies, or tests. You only write files under /reviews/ (create it if absent). If your environment can't launch subagents, run each role as a separate, fully completed phase, and don't let one role see another's output until synthesis.

Optional focus from the user: $ARGUMENTS (if empty, review the whole repo).

DEFINITIONS

Severity:
- Critical: exploitable security hole, live data-loss risk, production-down bug, or exposed live secret.
- High: significant correctness/security risk with likely real impact.
- Medium: real but contained, or needs specific conditions to trigger.
- Low: cosmetic, hygiene, dead code, negligible impact.
Blast radius: how much of the system/users/data is affected if the issue triggers or a fix goes wrong (not diff size).
Dead-code confidence: confirmed-dead | likely-dead | possibly-dead (use possibly-dead whenever usage can't be ruled out, including dynamic/reflective/string references).

======================================================================
PART 1 — ANALYSIS
======================================================================

1.1 Scope pass
Read README/ARCHITECTURE docs. Identify languages/frameworks, UI presence, database, background jobs, CI/CD, API surface, and whether it's a monorepo (list packages). Write a 5-10 line summary to /reviews/_scope.md. Every expert reads it instead of re-deriving context.

1.2 Launch expert subagents (in parallel where possible)
Each expert reads only its own domain, writes ONLY to /reviews/[role].md, and modifies nothing else. If a role doesn't apply, its file says so explicitly. No praise; issues and risks only.

Always run: Architect, Code Reviewer, Security Auditor, Test Engineer, Code Hygiene & Dead Code.
Run if applicable per scope: Reliability/SRE, Performance, Database, DevOps, Accessibility + Frontend/UX, API Design.

Finding format:
### [File/Location] — [Critical|High|Medium|Low]
**Finding:** 1-3 concrete sentences
**Why it matters:** real consequence
**Suggested fix:** brief, actionable

Expert criteria:
- Architect: boundary violations, circular deps, coupling, layering leaks, scalability at 10x.
- Code Reviewer: duplicated logic, over-complex functions/files, misleading names, unclear control flow, magic values, inconsistent error handling.
- Security Auditor (max rigor, don't soften): injection, hardcoded secrets, weak input validation, authn/authz gaps, unsafe deserialization, vulnerable dependencies (check lockfiles), IDOR, permissive CORS/webhooks, unscoped secrets to subprocesses, sensitive data in logs, missing rate limits, crypto misuse.
- Test Engineer: untested critical paths, tautological/over-mocked tests, missing edge/error cases, flaky patterns, coverage gaps on high-risk files.
- Code Hygiene & Dead Code: unused vars/functions/exports/imports/dependencies, unreachable code, commented-out code, orphaned files, stale feature flags, near-duplicate implementations, stale TODOs, dead CSS, unused fixtures. Verify with repo-wide usage search before calling anything dead, and tag every finding with a dead-code confidence level.
- Reliability/SRE: unhandled I/O exceptions, missing timeouts/retries, weak logging, no graceful degradation.
- Performance: N+1, unbounded loops on user input, recomputation, missing caching, blocking calls on hot paths.
- Database: missing indexes/constraints, unsafe migrations, unjustified denormalization.
- DevOps: fragile CI, env parity, secrets in CI config, no rollback path.
- Accessibility: semantics/ARIA, keyboard traps, contrast, unlabeled inputs, focus indicators.
- Frontend/UX: design-system consistency, responsive behavior, interaction-state coverage (focus/loading/empty/error), form validation, microcopy consistency, motion/reduced-motion, component duplication, generic AI-design tells. End with a line confirming each sub-area was checked.
- API Design: naming consistency, versioning, error shapes, undocumented behavior, inconsistent auth.

LIVE SECRET EXCEPTION: if any expert finds a live, currently-exposed credential, write it to its file immediately, tell the user right away, and put it prominently at the top of /reviews/_synthesis.md. Do not bury it.

GATE 1: Confirm every role that ran produced a non-empty, correctly formatted file. Redo a failed role once; if it fails again, stop and report.

1.3 Synthesis
Work only from the expert files; don't re-review code.
1. Deduplicate overlapping findings, noting all contributing roles.
2. Surface conflicts between roles explicitly (e.g. Architect says merge, Hygiene says delete); don't silently pick one.
3. Prioritize by severity × blast radius.
4. Account for every source finding: merged (into what) or dropped (why). Never silently drop.
5. Hygiene findings default to Low unless they mask a real correctness/security risk; then tier by real risk. Anything likely-dead/possibly-dead must carry "verify before deletion."

Write /reviews/_synthesis.md: a table (ID | Title | Severity | Source role(s) | One-line summary) with stable IDs (SYN-C1, SYN-H1, SYN-M1, SYN-L1...), followed by full detail for Critical and High items.

GATE 2: Spot-check per-file finding counts against synthesis so nothing is lost.

======================================================================
PART 2 — IMPLEMENTATION PLAN (still no code changes)
======================================================================

Write /reviews/_implementation-plan.md. For EACH finding, in execution order (Critical → High → Medium → Low; within a tier, smallest blast radius first, respecting dependencies), use exactly this structure so the implement command can parse it:

### [SYN-ID] — [title] — [severity]
- **Status:** ready | needs-human | needs-verification
- **Files/locations:** (verified against current code; note if drifted)
- **Approach:** concrete steps, specific enough to execute without re-analysis
- **Tests:** existing tests to run; new regression tests to add (required for Critical/High; not needed for pure dead-code removal)
- **Verification:** how to confirm it's actually fixed
- **Fix risk / blast radius:**
- **Depends on:** other SYN IDs, and files overlapping other findings
- **Question for human:** (needs-human only) phrased so a short answer unblocks it
- **Extra checks required:** (needs-verification only) dynamic/reflective/string refs, templates, config, build output

Status meanings:
- ready: safe to implement as described.
- needs-human: irreversible migration, public API/backward-compat change, auth/authz/payments/secrets handling, or a product/business call.
- needs-verification: likely-dead/possibly-dead deletions.

End the plan with:
1. Suggested batching: commit grouping. Keep Critical/security fixes separate from hygiene/dead-code cleanup; group hygiene deletions by category (imports, orphaned files, dead branches); split anything over ~400 lines of real logic.
2. Open questions for the human: every needs-human item (ID, question, tier).
3. New issues noticed while planning that aren't in the synthesis.
4. Scope gaps: anything not reviewed or coverable (e.g. no tests present, unreviewed packages).

======================================================================
FINAL OUTPUT (in chat)
======================================================================

A short summary, not a dump of the files:
- Any live exposed secret (at the very top)
- Finding counts by severity; roles run/skipped
- Top 5 highest-priority items
- Count of ready / needs-human / needs-verification items
- Every needs-human question
- Paths to /reviews/_synthesis.md and /reviews/_implementation-plan.md
- Next step: run /implement

HARD RULES

- Never modify anything outside /reviews/.
- Never write, apply, or "try out" fixes, even trivial ones.
- Never invent severity tiers or dead-code labels outside the definitions above.
- Never mark something dead without a repo-wide usage search; when unsure, use possibly-dead.
- Never silently drop or downgrade a security finding.
- Never guess on needs-human items; write the question instead.
- Report plainly if any area couldn't be analyzed.
