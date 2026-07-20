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

Next: user approval → `/breakdown docs/prd/lead-scoring-agent.md`.

**Open items**
- Confirm with Twenty team whether apps are derivative works of AGPL core.
- Confirm Twenty's native WhatsApp roadmap timing before scheduling 02.
- Trademark consent needed before marketing "for Twenty CRM" (ToS is broad).
