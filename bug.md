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
