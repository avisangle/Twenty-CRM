# D-TCA — Confirmed GA SDK API shapes (TCA-6 spike) + burst-detector change

**Status:** Accepted (2026-07-20)

Resolves the Lead Scoring PRD's 6 Open Questions against **ground truth** — the installed
`twenty-sdk@2.22.0` / `twenty-client-sdk@2.22.0` TypeScript definitions in `node_modules`
(not docs). Supersedes the PRD "Open Questions" section.

## Decision 1 (design change) — detect bulk import via `createdBy.source`, not a count heuristic

The `databaseEvent` payload has **no** import/batch flag on the envelope. But every record carries
a `createdBy` ACTOR whose `source` is an enum: `'IMPORT' | 'MANUAL' | 'API' | 'WORKFLOW' | 'AGENT'
| 'SYSTEM' | 'WEBHOOK' | 'APPLICATION' | 'EMAIL' | 'CALENDAR'` (`twenty-client-sdk/.../core/generated/schema.ts`).

**Decision:** `scoreOnEvent` routes on `event.properties.after.createdBy.source`:
`'IMPORT'` → enqueue (queue/sweeper); otherwise (`'MANUAL'`/`'API'`) → inline score.
`source` is optional — treat missing as "inline" (safe default).

**Rejected:** the PRD's count-of-recent-creates burst heuristic (BURST_WINDOW/THRESHOLD) — fragile,
adds a query per event, and misclassifies small imports / fast manual entry. The native flag is
reliable and free. **This removes Open Question Q6 (burst tuning) entirely.**

## Decision 2 — CoreApiClient is genql query-builder only; instantiated in-function

`CoreApiClient` exposes only `query(...)`, `mutation(...)`, `uploadFile(...)` — **no**
`findOne`/`findMany`/`createOne`/`updateOne`. The PRD's REST-style calls and `context.coreApiClient`
are wrong: the handler gets no `context` client; create `new CoreApiClient()` (auth token from env:
`TWENTY_APP_ACCESS_TOKEN` / `TWENTY_API_KEY`).

- Read: `client.query({ people: { __args: { first, filter, orderBy }, edges: { node: {...} } } })`
- Write: `client.mutation({ updatePerson: { __args: { id, data: {...} }, id: true } })`
  (also `createPerson`/`createPeople`/`updatePeople`/`updateOpportunity`).
- **Relations:** scalar FK `{ companyId }` OR `{ company: { connect: { where: { id } } } }`.

All logic-function issues (TCA-7/9/10) must use this genql style, not the PRD's snippets.

## Decision 3 — defineView sorts + defineField shapes (for TCA-3, TCA-8)

- **View sort** element: `{ universalIdentifier, fieldMetadataUniversalIdentifier, direction }`,
  where `direction` is `ViewSortDirection.ASC|DESC` (uppercase enum). `ViewKey.INDEX` is the only key.
- **defineField**: `{ type: FieldType.<T>, name, label, objectUniversalIdentifier, description?, icon?,
  isNullable?/defaultValue, options? }`. Enum values: `NUMBER`, `TEXT`, `DATE_TIME`, `SELECT` (exported
  as `FieldType`, alias of `FieldMetadataType`). SELECT `options` = `{ label, value, position, color }`
  where `color` is a `TagColor` union. So `leadScoreStatus` = SELECT with options queued/scored/skipped/failed.

## Decision 4 — activity enrichment source (TCA-7)

Person & Opportunity both expose `timelineActivities`, `noteTargets`, `taskTargets`. Enrichment fetches
recent activity via the `timelineActivities` connection (`orderBy: [{ createdAt: DESC }], first: N`),
optionally joining `note { title bodyV2 createdAt }` / `task` through the target rows.

## Q3 RESOLVED at TCA-9 (live, 2026-07-21) — event name is `person.created` (singular)

Live test (created a Person via GraphQL against the sandbox): the databaseEvent trigger fired on
**`person.created`** — the singular form. `people.created` is NOT used. Pattern is
`<objectNameSingular>.<action>` (`person.created`/`person.updated`/`opportunity.created`/`opportunity.updated`).
Also confirmed: one DB trigger per `defineLogicFunction` (settings are singular) → one thin function
file per (object, action) delegating to a shared handler.

Original (pre-resolution) note retained:

**Q3 event-name pluralization is NOT provable from the installed types** (`eventName: string`, no enum;
CLI example uses placeholder `objectName.created`). Convention + singular `STANDARD_OBJECTS` keys point
to **`person.created` / `opportunity.updated`**, but this MUST be verified against the live workspace at
first `scoreOnEvent` sync. Acceptance note added to TCA-9.
