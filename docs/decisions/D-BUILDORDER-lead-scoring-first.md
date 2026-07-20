# D-BUILDORDER — Build 01 Lead Scoring before 06 Migration

**Status:** Accepted (2026-07-20)

`decisions.md` (frozen) ranks 06 Migration #1 by revenue and 01 Lead Scoring #2.
This record flips the **build** order while keeping that **priority** order.

**Decision:** Build 01 Lead Scoring first; defer 06 until a real migration client exists.

**Why (the non-obvious constraint):** 06 cannot be *finished* without a real client.
Its PRD (§11) states the boilerplate is small and the real work is per-client field
mapping, custom pipeline stages, and dedup edge cases against real source data. Built
against a sandbox it's a skeleton that won't survive real client data — so 06 is blocked
on an engagement existing. Confirmed with user: no client lined up yet.

01 has zero blockers (no external API, no OAuth, no client), is finishable in 2-4 days,
and its only listed risk (alpha SDK) is stale — SDK is GA at 2.22.0 per progress.md.
It becomes the capability demo that wins the 06 engagement.

**Rejected:** Build 06 first (per decisions.md ranking) — rejected because it stalls
without a client and real data; effort spent on speculative field mappings is wasted.

**Revisit when:** a migration client/prospect signs — then 06 starts against real data.
