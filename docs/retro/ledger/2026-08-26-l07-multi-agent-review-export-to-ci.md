# Retro — L07 lab: Multi-Agent Review + Export to CI (parallel worktrees, merged)

**Date:** 2026-08-26
**Mode:** manual (not `workflow-retro`-generated — see note below)
**Run retro'd:** two features built in separate worktrees per the L07 lab
instructions — `feat/multi-agent-review` (specs/13) and `feat/export-to-ci`
(specs/14) — then reconciled and merged into
`feat/07-multi-agent-review-export-to-ci` in this session.

## Why this isn't a `workflow-retro`-skill output

The skill's Step 0 requires an `Agent`/`SendMessage` dispatch history to
retro, and refuses to fabricate one. This session made zero `Agent`/
`SendMessage` calls — all reconciliation work (branch fast-forward, migration
renumbering, merge conflict resolution, verification) was done directly. The
two build sessions that produced the features' code ran earlier, in separate
Claude Code sessions this session has no transcript access to. What follows
is a manually-compiled set of measured facts from git history and one live
product run — exactly the "виміряні величини" (measured quantities) the lab
instructions ask for, but not a per-dispatch token/tool-use breakdown, which
would require data this session does not have.

## Timeline (from git history, both branches' base: `bc00e8b`)

| Branch | Span (commit timestamps) | Commits | Where built |
|---|---|---|---|
| `feat/multi-agent-review` | 2026-08-21 19:23 → 2026-08-25 22:09 | 5 (`f509b18`…`f037a98`) | **Deviation**: built directly on what became `feat/07-multi-agent-review-export-to-ci`, not in the prescribed `../devdigest-review` worktree — that worktree instead held an abandoned, uncommitted WIP dated 2026-08-21 06:40, discovered and discarded this session (see [server/LEARNINGS.md](../../../server/LEARNINGS.md) 2026-08-26 entries) |
| `feat/export-to-ci` | 2026-08-20/21 → 2026-08-25 22:22 | `bcf781d`…`bfcc4dc` + one uncommitted checkpoint completed this session (`bcb3f4d`) | `.worktrees/devdigest-ci`, as prescribed |
| Reconciliation + merge | 2026-08-26 (this session) | `bf84e8e` (migration renumber) → `60ecf66` (merge) → `e5eab16` (post-merge fix) → `1a689a8` (docs) | Main checkout |

**Actual overlap period** between the two branches' active build spans:
2026-08-21 through 2026-08-25 — both were being built concurrently across
that span, per commit timestamps on both branches within the same days.

**Wall-clock from reconciliation start to green merge (this session):** not
independently timed against a clock, but bounded by this session's own tool
call sequence — one fast-forward, one migration renumber+commit, one
`git merge --no-ff` (4 real conflicts: `client/LEARNINGS.md`,
`server/LEARNINGS.md`, `client/src/lib/hooks/index.ts`,
`server/src/db/migrations/meta/_journal.json`), one post-merge typecheck fix,
then full verification (server 82 files/633 tests, client 52 files/279
tests, `pnpm run arch` 0 new violations, `pnpm db:migrate` clean against the
live shared DB) — all green on the first attempt after conflict resolution,
zero retry loops.

**Retries:** 0 failed merge attempts, 0 failed verification runs requiring a
second pass. One real defect required a follow-up fix commit
(`MultiAgentResultsView.tsx`'s stale `RunTraceDrawer` import path — a
cross-file reference the line-based merge could not catch, caught by
`pnpm typecheck` immediately after merging).

## Conflicts

**Merge conflicts:** 4 textual (all resolved by direct concatenation or
renumbering, zero logic changes) + 1 semantic collision requiring real
surgery: both branches independently `pnpm db:generate`d a migration numbered
`0023` against the same shared dev Postgres (`0023_flimsy_nick_fury.sql` vs
`0023_tidy_zaran.sql`+`0024_rare_maestro.sql`), confirmed already applied to
the live DB via `drizzle.__drizzle_migrations` hash lookup before touching
anything. Resolved by renumbering `feat/export-to-ci`'s pair to `0024`/`0025`
and repointing the snapshot's `prevId` chain.

**Product-level conflicts** (Multi-Agent Review's own conflict-detection
feature, exercised as manual verification of specs/13): a live 3-agent run
on PR #491 found **6 conflicts** among 7 total findings across General/
Security/Performance reviewers — e.g. one location where General flagged a
WARNING and both Security and Performance recorded `did not flag`.

## Cost and duration (real, from one live product-level run — PR #491, `deepseek/deepseek-v4-flash` via OpenRouter)

| | 1 agent (General, solo) | 3 agents (batch: General + Security + Performance) |
|---|---|---|
| Wall-clock | ~23s (diff 17ms + intent 9.5s + review ~14s, nothing shared with a batch) | 210.8s total — **not 3×** the solo figure |
| Per-agent duration | — | General 15.4s · Performance 41.6s · Security 210.8s (one outlier LLM call dominates the batch total) |
| Findings | 3 | 4 + 2 + 1 = 7 (6 in conflict) |
| Cost | not captured for the solo run | $0.00165 total across 3 agents |

**Explicit non-conclusion, per the lab's own instruction not to force a 3×
narrative:** the batch's wall-clock was ~9× the solo run's, not 3×, entirely
because one agent's LLM response was an outlier — shared diff/intent
preparation saved time, but did not make the batch's total scale linearly
with agent count either direction.

**Human review time this session:** not independently timed; bounded by one
continuous session covering discovery (finding the worktree/branch
mismatch), reconciliation planning, migration surgery, merge, and
verification — no elapsed-time instrumentation was in place to report an
exact figure.

## Export-to-CI live verification (Parts 3–4, added 2026-08-26 later same session)

The lab's final checklist items 3 and 4 ("тестовий PR проходить рев'ю в
GitHub Actions, а CRITICAL блокує merge через required check"; "CI Runs
отримує автентифікований результат") were verified against a real, freshly
created public repo (`viktormashyka/devdigest-ci-test`, created this session
specifically for this — no course-provided demo repo was available), not
simulated:

- **Export PR** (`#1`, `viktormashyka/devdigest-ci-test`): DevDigest's own
  `POST /agents/:id/export-ci` opened a real PR (branch `devdigest/ci` →
  `main`) containing the generated `.github/workflows/devdigest.yml`, agent
  manifest, and bundled `agent-runner`. Reviewed manually (fork-guard
  `if: github.event.pull_request.head.repo.fork == false`, least-privilege
  `permissions: {contents: read, pull-requests: write}`, external actions
  pinned to full commit SHA, secrets referenced only via `${{ secrets.* }}`)
  before merging.
- **Test PR** (`#2`, same repo, no fork — a hardcoded session secret +
  365-day session TTL introduced in `src/middleware/session.ts`): the
  exported workflow ran for real. `agent-runner` reported
  `findings=1 blockers=1 gateTriggered=true posted=github_review`, exited 1,
  and posted a real `CHANGES_REQUESTED` GitHub review naming the exact
  hardcoded-secret line. Cost **$0.000353411**, duration **16.94s** (from the
  uploaded `devdigest-result.json` artifact, not estimated).
- **Required-check block, proven by attempting the merge, not just reading a
  status badge:** after adding branch protection (`required_status_checks:
  {contexts: ["review"]}`), `gh pr merge 2` failed with `"the base branch
  policy prohibits the merge"` — `mergeStateStatus` was `BLOCKED`.
- **CI Runs ingest, proven with real attribution:** `POST /ci/refresh`
  ingested 3 real GitHub Actions runs; each `ci_runs` row correctly carries
  the real `commit_sha`, `pr_number`, `agent`, `cost_usd`, and `duration_s`
  pulled from the actual workflow artifact — including one incomplete run
  (a diagnostic branch deleted mid-flight) correctly landing as
  `status: "failed", failure_reason: "no_artifact_published"` rather than
  silently disappearing or crashing the ingest.
- **Secrets never appeared in logs or artifacts** — `OPENROUTER_API_KEY`/
  `GITHUB_TOKEN` show as `***` in the Actions log; `devdigest-result.json`
  contains only findings/verdict/cost, no credential material.

**Retries this segment:** 3 failed `export-ci` attempts before success — not
a product defect, an infrastructure/credential gap (next section) — plus one
`kill`-a-stale-server attempt that failed silently (wrong terminal session)
before the human restarted the dev server manually.

## Signals

1. **Handoff efficiency** — not applicable; no `Agent`/`SendMessage` calls in
   this session.
2. **Process deviation discovered, not injected:** `feat/multi-agent-review`
   was built on the wrong branch relative to the lab's own worktree-isolation
   instructions (see Timeline above) — caught only because this session
   independently audited `git -C ../devdigest-review diff --name-only`
   against the lab's own stated verification step, not because anything
   failed loudly.
3. **A clean `git merge` (0 conflicts reported) is not proof of correctness**
   — the `RunTraceDrawer` import break passed the merge silently and was
   only caught by running `pnpm typecheck` as a separate, deliberate
   post-merge step.
4. **A misleading GitHub API error cost three retries before the real cause
   surfaced:** `commitFiles`'s `git.createTree` call failed identically three
   times (initial attempt, after granting Contents+Pull-requests permissions,
   after a full dev-server restart) with GitHub's own error text — "Resource
   not accessible by personal access token" — which reads as a generic
   scope/propagation problem. Direct reproduction via `tsx` running the
   app's own `LocalSecretsProvider`/`OctokitGitHubClient.commitFiles` code
   (not just `curl`) with progressively narrowed file sets (single file →
   full 6-file payload → `.github/workflows/*` alone) isolated the actual
   cause: fine-grained GitHub PATs gate `.github/workflows/**` writes behind
   a **separate "Workflows" repository permission**, distinct from
   "Contents: Read and write" — see `server/LEARNINGS.md` 2026-08-26 entry.

## Recommendations

✅ `Target: root CLAUDE.md`, "Conventions (non-default)" section — state
explicitly that this repo's local Postgres is shared across every git
worktree (one `docker compose` instance, not one per worktree), and that two
worktrees generating migrations independently WILL collide on `idx` if both
touch schema in the same lesson. Justified by signal 2's class of near-miss
and the pre-existing `server/LEARNINGS.md` 2026-08-21/2026-08-26 entries
already describing two separate real occurrences of this exact collision —
recurring enough to belong in the root file every session reads, not just a
module's `LEARNINGS.md`.

✅ `Target: root CLAUDE.md`, "Gotchas" section — add: after merging two
branches whose specs both reference reusing the same shared component from
different call sites, run `pnpm typecheck` before trusting a conflict-free
`git merge` — a clean merge only proves no two branches touched the same
lines, not that cross-file references between them still resolve. Justified
by signal 3.

✅ `Target: specs/14-export-to-ci.md`, "Security" section — add a line: a
fine-grained GitHub PAT used for `export-ci` must be granted **Contents,
Pull requests, AND Workflows** (all "Read and write") — Contents alone lets
every non-`.github/workflows/` file in the export succeed while the workflow
file itself silently fails with a scope error that reads as a general
credential problem, not a missing-permission-category one. Justified by
signal 4 — this cost three real retries (including a full dev-server
restart that turned out to be unnecessary) before isolation via direct
reproduction found the actual cause.

*(No `RECURRING` marker applies — this is the second ledger entry, and its
predecessor, `2026-08-12-workflow-retro-skill.md`, retro'd an unrelated
single-dispatch `implementer` run with no comparable recommendations to
check against.)*
