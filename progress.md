# Progress

## 2026-07-19 — PRD analysis + monetization research

**Analysed** six Twenty CRM app PRDs (01-06); recommended build order:
06 Migration (HubSpot) → 01 Lead scoring → 03 PostHog → 02 WhatsApp → 05 E-sign → 04 Google.

**Researched** Twenty marketplace/listing viability. Key findings:

- Marketplace **shipped 2026-07-08** (twenty.com/apps). PRDs' "no marketplace" claim is stale.
- SDK is **GA at 2.22.0**, not alpha. PRDs' alpha-risk sections are stale.
- Public directory is **vetted-only** (3 first-party apps). Self-service listing PR #22621 still open.
- **No developer payment rails.** `chargeCredits()` meters workspace credits, not app revenue.
- Ecosystem = **18 npm packages** w/ `twenty-app` keyword. All third-party ones are MIT.
- SDK is **AGPL-3.0, no linking exception**, statically bundled into shipped artifact.
  Closed-source paid tier is legally ambiguous — unresolved.
- Freemium plugin economics weak: 1-3% conversion, needs 10k+ free users / 18-36 months.

## 2026-07-20 — Build-order decision

Confirmed no migration client lined up. **Decided: build 01 Lead Scoring first**, defer
06 Migration until a real client + real data exists (06 can't be finished on a sandbox).
See `docs/decisions/D-BUILDORDER-lead-scoring-first.md`.

Read all six PRDs. **Build-now ranking** (finishable today × validated demand × fit —
distinct from the revenue ranking in decisions.md):
1. 03 PostHog — named first-party blocker, 3-5d, same buyer as 06. (fix HogQL injection)
2. 01 Lead Scoring — fastest/zero-dep, shiniest SDK, but no demand signal.
3. 05 E-sign PandaDoc — clean build, mid demand.
4. 02 WhatsApp — top demand but blocked on Meta verification (start paperwork in parallel).
5. 06 Migration — only money-validated, but can't finish without a client (the *sell*).
6. 04 Google — weakest demand + monetization; consider not building.

## 2026-07-20 — /prd for 01 Lead Scoring → draft PRD written

User chose to implement **01 Lead Scoring** as a **real sales tool for a real rep** (not a
demo). Ran interactive /prd; verified GA SDK API via Context7. Wrote
`docs/prd/lead-scoring-agent.md` (draft, pending approval).

Locked decisions: score **both** person + opportunity; default list view sorted by score
(native re-sort kept); **hybrid** execution — single manual create scored inline (seconds),
bulk import → queue drained by cron sweeper (~1 min); **fullest enrichment** (record +
company + recent activity); cost guards (no-signal skip, cooldown, per-run cap, burst→queue).
Effort revised 2-4d → 5-8d. GA API correction: `defineLogicFunction` +
databaseEvent/cron trigger settings (old alpha `TwentyFunction` code was stale).
6 API shapes flagged in Open Questions to verify at scaffold.

Published to Confluence (space SD): hub "Twenty CRM — App Ideas" (page 720897) with the
Lead Scoring PRD as child page 753665. Local `docs/prd/lead-scoring-agent.md` remains source
of truth; other 5 PRDs will slot in as sibling child pages.

## 2026-07-20 — /breakdown launched (first post-overhaul run)

Jira: no Twenty CRM project existed (only GR/KAN/NEHA/PPA/YA — all unrelated). User created
**project TCA "Twenty CRM Apps"** (id 10165, next-gen software, Epic→Story→Task→Subtask).
Confirmed via API after a short propagation delay.

Verified capability limits (user challenged): the Atlassian MCP **cannot create a Jira project
or Confluence space** — only issues (in an existing project) and pages (in an existing space).
Terminology: Jira = *projects*, Confluence = *spaces*.

Ran `/breakdown docs/prd/lead-scoring-agent.md` in a general-purpose **subagent**, pinned to
project TCA only, v1 scope only, Open-Questions → spike issues. Awaiting its issue tree to
audit against the PRD (watching for wrong-project leakage, invented scope, broken deps,
missing AC, skill regressions).

Reorg: moved the six idea briefs into per-idea folders under `ideas/<n>-<slug>/` (git mv,
staged not committed); added `ideas/README.md` index. Left `docs/prd/lead-scoring-agent.md`
in place (subagent reading it). Root now: decisions.md, progress.md, docs/, ideas/.

## 2026-07-20 — /breakdown completed → 10 issues in TCA

Decomposed Lead Scoring PRD (v1 only) into **Epic TCA-1 + 9 children**, all in project TCA
(id 10165), correctly parented, Blocks-linked, level-labelled. No leakage to other projects.

Levels: L1 TCA-2 (scaffold) → L2 TCA-3 fields / TCA-4 agent / TCA-5 role / TCA-6 SPIKE (Task) →
L3 TCA-7 enrichment / TCA-8 views → L4 TCA-9 scoreOnEvent / TCA-10 drainQueue.
Open Questions: Q1–Q5 → TCA-6 spike; Q6 (burst tuning) → TCA-9. Phase 2 / out-of-scope excluded.
Skill ran clean — no errors, no wrong-project attempts, no circular deps. Start: `/tdd-prep TCA-2`.

Independently verified via `project = TCA` JQL: 10 issues, all parented to TCA-1, correct
types/level-labels, zero leakage. First-run abnormalities to note: (1) breakdown's interactive
approval gates were silently bypassed in the subagent — skill has no non-interactive/auto-approve
affordance; (2) skill's preferred TDD Confluence child-page was skipped (design embedded in Epic);
(3) `createJiraIssue.parent` param is mis-documented ("for subtasks" but sets Epic parent).

## 2026-07-20 — App code repo created

Per "own sibling repo" decision: created **~/AI/Personal/lead-scoring-agent/** (git init, main)
with README (links to PRD, Confluence 753665, Jira TCA-1..10) + node .gitignore. This planning
repo stays docs-only. Scaffolding (create-twenty-app = TCA-2) deferred: **no Twenty sandbox yet**
— step zero is spinning one up, then `auth:login`.

## 2026-07-20 — App scaffolded + synced (TCA-2 done)

Sandbox: https://lead-scoring.twenty.com. Toolchain blocker found: SDK 2.22.0 requires Node
^24.5 + Yarn 4; machine had Node 22 (at /usr/local/bin, shadowing a Homebrew Node 25). Installed
`brew node@24` (v24.18.0, keg-only) — use via `PATH=/opt/homebrew/opt/node@24/bin:$PATH`.

Scaffolded via `create-twenty-app@2.22.0` into ~/AI/Personal/lead-scoring-agent (yarn4, node-modules
linker, .nvmrc 24.5.0, own git initial commit). OAuth to the sandbox succeeded during scaffold; the
scaffold's step-6 sync failed transiently but `yarn twenty dev --once` then synced cleanly (6 objects
added). App live in workspace. README updated with planning links (uncommitted).

## 2026-07-20 — TCA-6 spike done (GA API shapes from installed types)

Subagent read installed twenty-sdk/twenty-client-sdk types. Findings → `docs/decisions/D-TCA-ga-api-shapes.md`.
Key wins: (1) **native `createdBy.source` ACTOR flag** (`IMPORT`/`MANUAL`/`API`) replaces the
count-based burst detector — simpler/reliable TCA-9, kills Q6; (2) CoreApiClient is **genql-only**
(`query`/`mutation`, `new CoreApiClient()` from env — PRD's `findOne`/`createOne`/`context` were wrong);
(3) view sorts use `fieldMetadataUniversalIdentifier` + `ViewSortDirection.DESC`, `ViewKey.INDEX`;
(4) activity via `timelineActivities`/`noteTargets`/`taskTargets`; (5) `defineField` shape confirmed for TCA-3.
Open: Q3 event-name pluralization (person.created vs people.created) — verify live at TCA-9.

## 2026-07-20 — TCA-3 done (custom fields synced)

8 `defineField` files under `src/fields/` (person + opportunity × leadScore/Summary/ScoredAt/Status),
referencing `STANDARD_OBJECT_UNIVERSAL_IDENTIFIERS.{person,opportunity}.universalIdentifier`; UUIDs in
`src/constants/universal-identifiers.ts`. `yarn twenty dev --once` → 8 added, ✓ synced live. Committed +
pushed (app repo main c768b66). Both repos now on GitHub: avisangle/Twenty-CRM + avisangle/lead-scoring-agent
(private). TCA-3 → Done.

Known: 2 pre-existing typecheck errors in scaffold `src/__tests__/schema.integration-test.ts`
(`created.createNote` possibly undefined) — not ours; fix pending.

## 2026-07-20 — TCA-5 + TCA-4 done (subagent)

**TCA-5:** added `permissionFlagUniversalIdentifiers: [SystemPermissionFlag.AI]` to `default-role.ts`.
**TCA-4:** `src/agents/lead-scorer.agent.ts` (defineAgent, flat `{score,summary}` responseFormat) +
`src/lib/build-prompt.ts` (person/opportunity prompt builders) + unit test (4/4 pass). Also fixed the
2 pre-existing scaffold typecheck errors → `yarn typecheck` now 0 errors. `twenty dev --once` synced
(agent + rolePermissionFlag). Commits f42afff (TCA-5) + 3793b8b (TCA-4), pushed app repo main.
Design note: agent schema is flat-primitives-only (no min/max) — 0-100 bound lives in field description.
Both → Done.

## 2026-07-21 — TCA-7 + TCA-8 done (subagent)

**TCA-7:** `src/lib/enrich.ts` — genql `CoreApiClient().query` fetching record + linked company +
recent `timelineActivities` (first 10, DESC). Field names verified vs generated schema. Mocked-client
unit test. build-prompt.ts extended minimally. Commit 3895e04.
**TCA-8:** `src/views/{person,opportunity}-leads.view.ts` — INDEX views sorted by leadScore DESC.
Sync plan: 4 add / 1 change (own component) / 0 destroy — **no standard view touched**. Commit 9274d73.
typecheck 0, unit 7/7. Both → Done.

Caveats to track: (a) standard Company has no size/industry/employees — enrichment uses
name/domain/annualRevenue/address (Phase-2: add custom Company fields if scoring needs more);
(b) whether the app INDEX view is the nav *default* is server-side — **verify/set in UI**
(the one PRD "default sort" AC needing a live check).

Remaining v1: TCA-9 scoreOnEvent, TCA-10 drainQueue (both now unblocked).

## 2026-07-21 — TCA-9 + TCA-10 done → v1 dev COMPLETE (live-verified)

**TCA-9:** one DB trigger per function (SDK limit) → 4 thin trigger files → shared `src/lib/score-on-event.ts`
(no-signal→SKIPPED, createdBy.source IMPORT→QUEUED, else inline `scoreRecord`). Loop-safe (update triggers
watch business fields only). Commit 2f7c597.
**TCA-10:** `drain-queue.function.ts` cron `* * * * *` → drains QUEUED (cap 20/type) via shared `scoreRecord`.
FAILED terminal for v1 (no retry-count field — bounded-retry = follow-up). Commit 75328cf.
typecheck 0, unit 14/14, sync ✓ (5 logic functions).

**✅ Q3 resolved live:** trigger fires on `person.created` (singular). Test person → SCORED, leadScore=45
in ~20s, then cleaned up. Decision record updated.

Follow-ups filed/pending: TCA-11 (custom Company fields, Phase 2). Possible: retry-count field for bounded
retry. **Manual acceptance still open:** (a) confirm/set the leads view as UI default (sort-by-score AC);
(b) scoring-quality gate — sanity-check score spread on cold→hot leads with a real pipeline.
All 9 stories (TCA-2..TCA-10) + spike Done. Both repos pushed.

## 2026-07-22 — Quality gate PASSED

Created 4 test people (title-only, cold→hot), auto-scored live, then deleted (0 remaining, confirmed):
Student Intern=8, Marketing Coordinator=22, Director of Sales=45, CRO=60. Strictly ascending, no
inversions — scorer behaving correctly. ~4 AI credits. Insight: title-only tops at 60 (agent is
conservative on sparse data) → richer scores need company/activity enrichment (TCA-11 + real data).
Op note: local ~/.twenty tokens had expired (~40h) and were refreshed; deployed functions use
server-side auth so production scoring is unaffected.

**v1 DONE + verified.** Only manual item left: confirm/set the leads view as UI default (sort-by-score AC).

## 2026-07-22 — UI-review fixes: visible columns + backfill (real-data gap)

User checked the frontend and saw no scores. Two real gaps found (lesson saved to memory):
1. **Existing records unscored** — event-driven scoring never ran on pre-install records; no backfill existed.
   → **TCA-12**: `backfill.function.ts` (POST /backfill) + `backfill.ts` (enqueue null-status → drainQueue scores).
   Idempotent. Live: scored all 11 existing records (People 45-62, Opps 55-94). Commit 204aff0.
2. **Score columns hidden** — views had sorts but no visible `fields`. → TCA-8 fix: added `fields`
   (name/leadScore/leadScoreStatus + signal cols). Commit 5bdaf7d.
typecheck 0, unit 16/16, sync ✓ (10 viewFields + backfill fn). TCA-12 → Done.
Caveat: /backfill HTTP route URL unconfirmed (405 on candidates); backfill ran via direct enqueue —
confirm production route URL as follow-up. Lesson: verify on real/existing data + real UI, not synthetic.

**Open items**
- Confirm with Twenty team whether apps are derivative works of AGPL core.
- Confirm Twenty's native WhatsApp roadmap timing before scheduling 02.
- Trademark consent needed before marketing "for Twenty CRM" (ToS is broad).

User confirmed live in UI: scores visible + sorted DESC in the "by Lead Score" views on real data. v1 shipped.

## 2026-07-22 — Incident: new-record scoring FAILED = AI credits exhausted (trial ended)

New Person scored FAILED. Diagnosed (subagent, via replicating the runAgent /metadata call):
**"Credits exhausted"** — workspace AI credits used up (user confirmed Twenty Cloud trial ended).
NOT a code bug; enrichment fine; systemic (re-trigger also failed, 0 credits consumed). Our agent
has no `modelId` → workspace-default model, metered by credits. See bug.md.
Fix: restore credits (upgrade Twenty Cloud plan) OR self-host Twenty with own LLM key (no meter).
Filed **TCA-13**: capture runAgent error + credit-aware re-queue (FAILED records aren't re-scored by
backfill since it filters NULL-status).

## 2026-07-22 — TCA-13 done via tdd→implement→review (merged + deployed)

Full skill workflow in subagents: `tdd-prep` (failing tests) → `implement` (PR #1, 22/22 unit, /simplify)
→ **independent** review (fresh subagent) → PASS → squash-merged to main (67d086b) → **deployed**
(`twenty dev --once`, 6 functions updated in-place). Live behavior now: credit-exhaustion → QUEUED
(auto-retries when credits return, rejected at credit check so no credit spent); genuine error → FAILED
with reason in `leadScoreSummary` + `console.error`; backfill re-processes FAILED. Decision record D-TCA-13.
Review minor findings (non-blocking): credit detection is string-match `/credits?\s+exhausted/i` (fragile
if SDK rewords, but degrades to operator-recoverable); no bounded retry-count yet → TCA-10.
Skill quirks flagged (overhauled skills): (1) new worktree had no node_modules (node-modules linker —
implementer symlinked); (2) initial file writes hit main not the worktree (self-corrected; main verified
clean). Review-side worktree teardown + squash-merge worked correctly.
Still blocked on AI credits (trial ended). Optional: re-queue the 2 FAILED records so they auto-score
when credits return.
