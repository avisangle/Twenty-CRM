# Bugs

## 2026-07-22 — New-record scoring fails: "Credits exhausted"

**Symptom:** newly-created Person ("Tester", company Anthropic) → `leadScoreStatus=FAILED`, no score.

**Root cause:** workspace **AI credits exhausted**. `runAgent` POSTs a `RunAgent` mutation to
`{TWENTY_API_URL}/metadata`; it returns `"Credits exhausted"` / "upgrade your plan". Systemic — a
valid re-trigger test also FAILED; **no credits consumed** (rejected at the credit check). NOT a code
bug; enrichment/query fine. Our agent has no `modelId` pinned → uses the workspace-default AI model,
metered by AI credits.

**Fix (workspace-side, not code):** top up / upgrade the workspace AI credits (Twenty Settings → AI /
plan; workspace `bf296e46-...`). Then re-score the FAILED records (needs a reset since backfill only
picks `NULL`-status — see TCA-13).

**Code weakness (diagnosability):** `src/lib/score-record.ts` swallows the runAgent error (empty
`catch`, discards `error`), so FAILED records carry no hint and logs are empty. → **TCA-13**: capture
error into `leadScoreSummary`/log; distinguish credit-exhaustion (transient → re-QUEUE) from terminal
failure; let backfill/retry re-process FAILED.

## 2026-10-31 — Activity signal silently empty on Twenty >= ~2.35 (published v0.1.0)

**Symptom:** on current servers `recentActivity` is always empty, so every lead is scored without its
timeline activity. No error: the server ignores unknown GraphQL fields and returns `name: null`.

**Root cause:** Twenty #24568/#24738 (Aug 2026) replaced `timelineActivity.name` with a timeline
activity type; the event name now lives in `timelineActivityTypeSnapshot.name` (e.g. `noteLinked`,
`recordUpdated`). `src/lib/enrich.ts` still selects `name`. Unit tests mock the old shape, so they
stayed green. Found while upgrading to SDK 2.54.3 (typed client flagged `name` as `unknown`).

**Fix:** select `timelineActivityTypeSnapshot` and read `.name`; ships in the SDK 2.54 upgrade (v0.2.0).

## 2026-10-31 — App repo CI/CD red on main since 2026-07-27

CI fails at Typecheck (implicit-any on `CoreApiClient` results, because the published client is an
untyped stub until `app apply` generates it). CD fails at Deploy because `TWENTY_DEPLOY_URL` is still
the `http://localhost:2020` template placeholder and no `TWENTY_DEPLOY_API_KEY` secret exists.
Typecheck fixed in the SDK 2.54 upgrade; CD needs a real deploy target or should be removed.

## 2026-10-31 — Publish fails with npm 12: "npm pack returned an invalid package description"

**Symptom:** publish workflow run 41230584002 (tag v0.2.0) failed at `twenty app publish`.

**Root cause:** the template's `npm install -g npm@latest` now installs npm 12.2.1, whose
`npm pack --json` returns an object keyed by package name instead of a one-element array. Twenty CLI
2.54.3 checks `isArray(result) && result.length === 1`. Not reported upstream yet.

**Fix:** pin `npm@11` in `publish.yml` (still >= 11.5.1, which trusted publishing needs). App repo PR #4.
Unpin once a Twenty CLI release accepts the npm 12 shape.

## 2026-11-01 — Install refused on Twenty Cloud: "Custom roles require the Organization plan"

**Symptom:** installing 0.2.0 from the marketplace fails with `[role] Custom roles require the
Organization plan (Lead Scoring Agent default function role)` and `[rolePermissionFlag] Role not found`.

**Root cause:** Twenty #29014 (2026-10-28) gates role creation, app installs included, behind the
CUSTOM_ROLES entitlement when a role is assignable to users or API keys. App manifests default both
flags to `true`; `src/default-role.ts` left them unset. The permission-flag error cascades from the
rejected role. Self-hosted servers grant the entitlement, so local tests could not catch it.

**Fix:** set `canBeAssignedToUsers` and `canBeAssignedToApiKeys` to `false` (v0.2.1, app PR #5).

## 2026-11-01 — Existing records never scored after a marketplace install

**Symptom:** on lead-scoring.twenty.com after installing 0.2.1, new and edited people score
(Test User: 40, Scored), but the 5 seed people with companies stay blank.

**Root cause:** the backfill (`src/logic-functions/backfill.function.ts`) is only exposed as an
authenticated HTTP route. Nothing calls it on install, the cron sweeper only drains QUEUED records,
and there is no UI control, so a UI-only user cannot score pre-existing records. README claims a
backfill.

**Fix:** TCA-16 (app PR #8, 0.3.0): post-install hook (also on upgrade) runs the backfill, which
now marks no-signal records SKIPPED; the app page has a "Score unscored and failed records" button.

## 2026-11-01 — lead-scoring cron will exhaust Twenty's shared app rate limit as installs grow

**Symptom (latent):** none yet at 1 install. Twenty allows 500 API requests per minute per app
registration, shared by every workspace that installed it, for cron work
(docs.twenty.com/developers/extend/apps/getting-started/good-practices). Requests over it get 429.

**Root cause:** `drain-queue.function.ts` runs `* * * * *` in every workspace at the same minute
and makes at least 2 queries per tick even when the queue is empty, plus several requests per
scored record. Around 250 installs the idle ticks alone use the whole budget.

**Fix:** not started. Docs pattern: the cron only enqueues a job with a random `delayMs`, skip
work when nothing is queued (e.g. a KV "queue non-empty" flag set by the event trigger), and
run the drain less often.
