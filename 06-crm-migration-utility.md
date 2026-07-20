# PRD: CRM Migration Utility for Twenty (HubSpot / Pipedrive / Salesforce → Twenty)

**App slug:** `crm-migration-utility`
**Type:** Twenty App (SDK) — route-trigger Functions + batch Core API usage (not a permanently-installed workspace app — more of a scoped tool run per migration engagement)
**Effort:** Medium (1-2 weeks per source CRM, most of it in field-mapping and edge cases, not boilerplate)
**External dependencies:** Source CRM API credentials (HubSpot private app token, Pipedrive API token, or Salesforce OAuth + Bulk API access)

---

## 1. What this is

A batch migration tool that pulls Contacts/Companies/Deals (and
optionally Activities/Notes) from a source CRM via its API and writes them
into Twenty via the Core API client, preserving relationships between
records rather than flattening everything into disconnected CSV rows.

## 2. Why this idea — what the research showed

This isn't validated by a GitHub Discussion vote count — it's validated by
**agencies already charging real money for exactly this work today**:

- The curated integrations wishlist lists Salesforce, Zendesk, HubSpot,
  Zoho, monday.com, Insightly, and Pipedrive under a "CRM" category
  explicitly framed as "mainly for migration to help users move from
  different CRMs to Twenty."
- Independent research into Twenty's partner ecosystem found multiple
  agencies actively selling migration services: one partner (Tectome)
  advertises implementations starting from a few hundred dollars scaling
  to custom pricing based on data volume and integrations, explicitly
  covering migration; another (Buildrhaus) markets itself around moving
  teams off HubSpot/Salesforce/Attio onto self-hosted Twenty. These are
  agencies charging clients for what is largely a data-engineering
  problem — the opportunity here is to **productize** what they currently
  do by hand.
- A separate practical note found in research: HubSpot's own CSV export
  path breaks relationships between objects (contacts, deals, and their
  associations export separately and must be manually re-linked) — the
  real value of an API-based tool over CSV is preserving those
  relationships automatically, which is the part clients actually pay
  agencies to handle correctly.

## 3. Goals / success criteria

- Contacts, Companies, and Deals/Opportunities migrate with their
  relationships intact (a Deal stays linked to the right Contact and
  Company, not just imported as a flat row).
- Custom fields on the source side map to either existing or newly-created
  custom fields in Twenty, based on a mapping config rather than being
  silently dropped.
- Tool is idempotent/resumable — a failed run partway through can be
  re-run without creating duplicates.
- Produces a migration report (counts migrated, counts skipped/failed,
  and why) rather than failing silently.

## 4. Architecture overview

```
Migration config (JSON/YAML): field mappings, object mappings, filters
        │
        ▼
runMigration (route trigger, invoked manually per batch/run)
        │
        ├─▶ Source CRM API: paginated read (contacts → companies → deals,
        │    in that dependency order so relations can resolve)
        │
        ├─▶ De-dupe check (match existing Twenty records by email/domain
        │    before creating, to make re-runs safe)
        │
        ▼
coreApiClient.createMany(...) with relation `connect` to already-migrated
parent records (companies before contacts before deals)
        │
        ▼
migrationLog object: per-record outcome, for the report
```

## 5. Data model

New custom object `migrationLog` (temporary/utility object, useful during
and shortly after the migration, can be archived after):

| Field | Type | Notes |
|---|---|---|
| `sourceCrm` | SELECT | `hubspot` / `pipedrive` / `salesforce` |
| `sourceObjectType` | SELECT | `contact` / `company` / `deal` |
| `sourceId` | TEXT | ID in the source system, for idempotency and traceability |
| `twentyRecordId` | TEXT | resulting Twenty record ID |
| `status` | SELECT | `migrated` / `skipped` / `failed` |
| `errorDetail` | TEXT | populated on failure |

## 6. Source-specific API notes

### HubSpot

- Auth: private app access token (Bearer) — simplest of the three, no
  OAuth flow needed for a single-workspace migration.
- Read via CRM API v3 search endpoints, e.g.
  `POST /crm/v3/objects/contacts/search` with pagination (`after` cursor)
  and an explicit `properties` list (don't rely on defaults — you'll miss
  custom fields).
- **Associations are a separate concern**: contact↔company and
  contact↔deal relationships are not embedded in the basic object read —
  fetch associations explicitly per object or via the Associations API,
  and resolve them into Twenty `RELATION` connects.
- Activities/notes/calls/meetings are **not** covered by CSV export and
  need the Engagements API specifically if the client wants full
  activity history migrated, not just core records — flag this as a
  separate scoping item, it's meaningfully more work than contacts/deals.

### Pipedrive

- Auth: API token (simple) or OAuth for a marketplace-style tool.
- Use API **v2 endpoints where available** — found to consume
  meaningfully fewer rate-limit tokens than v1 equivalents.
- Rate limiting is token-bucket based (a daily budget scaled by plan and
  seat count, with a rolling burst window) — for a large Pipedrive
  account, pace the migration run rather than firing requests as fast as
  possible.

### Salesforce

- Auth: OAuth 2.0 (standard, more setup than the token-based approach in
  the other two).
- For large data volumes, use the **Bulk API v2** rather than the
  standard REST API — it's built for exactly this kind of batch export
  and avoids hitting per-request API limits on large orgs.

## 7. Function

```ts
// src/functions/run-migration.function.ts
export const runMigration = new TwentyFunction({
  name: 'runMigration',
  triggers: [{ type: 'route', method: 'POST', path: '/migration/run' }],
  handler: async (input, context) => {
    const { sourceCrm, objectType, cursor } = input.data;
    const { coreApiClient } = context;

    const { records, nextCursor } = await fetchFromSource(sourceCrm, objectType, cursor);

    for (const record of records) {
      const alreadyMigrated = await coreApiClient.findMany('migrationLog', {
        filter: { sourceCrm: { eq: sourceCrm }, sourceId: { eq: record.id } },
        limit: 1,
      });
      if (alreadyMigrated.length) continue; // idempotent skip

      try {
        const mapped = mapFields(sourceCrm, objectType, record); // per-config mapping
        const created = await coreApiClient.createOne(mapToTwentyObjectType(objectType), mapped);
        await coreApiClient.createOne('migrationLog', {
          sourceCrm, sourceObjectType: objectType, sourceId: record.id,
          twentyRecordId: created.id, status: 'migrated',
        });
      } catch (err) {
        await coreApiClient.createOne('migrationLog', {
          sourceCrm, sourceObjectType: objectType, sourceId: record.id,
          status: 'failed', errorDetail: String(err),
        });
      }
    }

    return { migratedCount: records.length, nextCursor };
  },
});
```

Run order matters: **companies → contacts → deals**, so that when a deal
is created you can resolve its `RELATION` connects to already-migrated
company/contact records rather than creating orphaned deals.

## 8. Roles & permissions

Manifest needs write access to `company`, `person`, `opportunity`, and the
`migrationLog` object. Source-CRM credentials as `secret`-type settings,
scoped per engagement (don't reuse one credential across multiple client
migrations).

## 9. Build plan for Claude Code

1. Start with **HubSpot** as the first source (simplest auth, well
   documented, and matches the migration direction most commonly
   discussed by Twenty's own partner ecosystem).
2. `npx create-twenty-app crm-migration-utility`.
3. Build the `migrationLog` object and idempotency check first — this is
   the piece that makes the tool safe to re-run, build and test it before
   any real migration logic.
4. Build `fetchFromSource` for HubSpot contacts, paginated, with an
   explicit properties list.
5. Build field-mapping config (start with a hardcoded default mapping for
   standard fields, make it overridable per engagement).
6. Build the companies → contacts → deals sequencing with relation
   resolution.
7. Dry-run against a HubSpot sandbox/test account with a small dataset
   (10-20 records) before any real client data.
8. Add a simple report output (counts + failures) at the end of a run.
9. Repeat steps 4-6 for Pipedrive, then Salesforce, only once there's a
   real engagement that needs them — don't build all three speculatively.

## 10. Testing plan

- Test idempotency explicitly: run the same batch twice, confirm the
  second run migrates zero new records and skips cleanly via
  `migrationLog`.
- Test relationship resolution: migrate a company, then a contact
  belonging to it, then a deal belonging to that contact — confirm the
  deal's `RELATION` fields actually connect rather than creating
  duplicate/orphaned parent records.
- Test partial-failure handling: intentionally feed one malformed record
  in a batch, confirm it logs as `failed` with a useful `errorDetail`
  rather than aborting the whole batch.
- Test custom-field mapping with at least one non-standard field per
  source CRM to confirm the mapping config actually reaches Twenty.

## 11. Risks / open questions

- **This is a data-mapping problem more than a coding problem** — the
  bulk of real engagement time will go into per-client field mapping and
  edge cases (multiple emails per contact, custom pipeline stages,
  duplicate detection), not into the boilerplate above. Scope client
  quotes accordingly.
- **Activities/engagement history is a separate, larger scope** from core
  object migration — decide explicitly per engagement whether it's
  in-scope, since it requires different (and rate-limit-heavier) API
  calls on most source CRMs.
- **Rate limits differ meaningfully by source** — Salesforce Bulk API,
  HubSpot's search endpoint pagination, and Pipedrive's token-bucket
  system all need different pacing logic; don't assume one throttling
  strategy works for all three.
- **Competitive context**: several agencies (Tectome, W3villa, Buildrhaus,
  01GROWTH) already sell migration services manually — this tool's value
  is in speed/reliability for you as the implementer, or as a
  productized self-serve offering, not as a wholly novel market.

## Monetization

No platform-native payment rails in Twenty — invoice the client directly
(see this PRD set's general monetization research).

This is the **clearest, already-proven revenue model of the six** —
agencies in Twenty's own partner ecosystem are charging real money for
this exact work today, manually. Price **per source CRM + data volume**,
consistent with what was found in research (implementation partners
quoting a few hundred dollars for straightforward, smaller-volume jobs,
scaling to custom pricing for larger or more complex ones). This idea also
has the best unit economics of the set: once built and tested against one
source CRM, each additional migration client is mostly re-running the tool
with new credentials and a client-specific field-mapping config, not a
rebuild — the tool itself becomes reusable infrastructure rather than a
one-off script, which is where the real leverage is if you take on
multiple migration engagements.

## 12. References

- [Discussion #16735 — Native integrations wishlist](https://github.com/twentyhq/twenty/discussions/16735) (CRM migration section)
- Twenty partners directory: `twenty.com/partners/list` (existing migration-service agencies)
- HubSpot CRM API v3 (objects/search): `developers.hubspot.com`
- Pipedrive API reference (v2 endpoints, token-bucket rate limits): `developers.pipedrive.com`
- Salesforce Bulk API v2: `developer.salesforce.com`
- Twenty Custom Apps guide: `docs.twenty.com/developers/extend/apps`
