# Decisions

## 2026-07-19 — Build order for the six app PRDs

Ordered by *engagement-winning value*, not standalone app revenue, because Twenty
has no developer payment rails — all six monetize as services.

1. **06 Migration (HubSpot only)** — only idea validated by money changing hands today.
   Best unit economics; least SDK-coupled. Do not build Pipedrive/Salesforce speculatively.
2. **01 Lead scoring** — cheap (2-4d), zero external deps. Value is as a sales demo.
3. **03 PostHog** — upsell inside every #1 engagement (its buyer *is* a HubSpot refugee).
4. **02 WhatsApp** — highest ceiling, highest risk. Start Meta Business verification
   paperwork in parallel with #1 (zero eng cost); reassess position when it clears.
5. **05 E-signature** — PandaDoc first (API key, no JWT).
6. **04 Google Contacts/Tasks** — consider not building. Lowest urgency in the set.

## 2026-07-19 — Monetization: services-led, not freemium-product-led

**Decision:** publish apps free/open (MIT) as distribution + credibility; sell
implementation and retainers. Do not build a paid feature tier initially.

**Why:**
- No platform payment rails; no revenue share; no pricing field in the manifest.
- Freemium needs 10k+ free users / 18-36 months. Twenty's whole ecosystem is 18 apps.
- AGPL SDK bundles into the shipped artifact with no linking exception — a closed-source
  paid tier is legally unresolved, and the npm/marketplace path publishes source anyway.
- The two highest-value PRDs (06, 02) are not freemium-shaped: 06 is run-once per
  engagement, 02 is gated on client-side Meta verification.

**If a paid tier is added later:** make it a *hosted service* the app calls, not a
license check in shipped source. Sidesteps both the AGPL question and the fork risk.

**Deferred, not decided:** anti-piracy/DRM posture. Research on enforcement
effectiveness was retracted as unreliable — no sourced basis either way. Moot while
there is no paid tier; revisit only if one is added.
