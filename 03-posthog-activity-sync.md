# PRD: PostHog Web Activity Sync for Twenty CRM

**App slug:** `posthog-activity-sync`
**Type:** Twenty App (SDK) — Object + scheduled Function + Front-component
**Effort:** Low-medium (3-5 days for v1)
**External dependencies:** PostHog Personal API Key (self-hosted or PostHog Cloud)

---

## 1. What this is

A scheduled sync that pulls page-view and product-usage events from PostHog
for known contacts (matched by email) and writes them onto a timeline
attached to the Person record in Twenty — giving sales/support the same
"what has this person actually done on our site/product" context that
HubSpot's native tracking provides, without leaving Twenty.

## 2. Why this idea — what the research showed

From the curated integrations wishlist
([Discussion #16735](https://github.com/twentyhq/twenty/discussions/16735)),
a user (`benweissbehavehealth`) who had a direct conversation with
co-founder Felix Malfait flagged PostHog specifically and in strong terms:
they described it as **critical**, and named it as **one of the actual
blockers preventing their team from leaving HubSpot**, because HubSpot's
built-in page-view tracking tied to a contact record has no equivalent in
Twenty today. This is a named, first-party pain point from someone actively
evaluating a HubSpot-to-Twenty migration — not a speculative nice-to-have.

Unlike WhatsApp, this integration has **no regulatory or business-verification
overhead** — PostHog's API is a standard bearer-token REST/HogQL API, which
makes it one of the fastest items on this list to actually ship and
demo.

## 3. Goals / success criteria

- For any Person with a matching email in PostHog, their recent
  page-view/event activity is visible on their Twenty record without
  manual lookup.
- Sync runs on a schedule (not real-time) — acceptable latency is minutes,
  not seconds, since this is context, not a live chat channel.
- Sales/support can see "last active" and a short recent-activity list
  directly on the Person page.

## 4. Architecture overview

```
Scheduled function (cron, e.g. every 15 min)
   │
   ▼
For each person with email + posthogSyncedAt older than threshold:
   │
   ▼
POST https://<posthog-host>/api/projects/:project_id/query/
   { query: { kind: "HogQLQuery",
              query: "select event, timestamp, properties.$current_url
                       from events where person.properties.email = {email}
                       order by timestamp desc limit 20" } }
   │
   ▼
coreApiClient.createMany('webActivity', [...]) + update person.posthogSyncedAt
   │
   ▼
Front-component: recent activity list on Person record
```

## 5. Data model

New custom object `webActivity`:

| Field | Type | Notes |
|---|---|---|
| `eventName` | TEXT | e.g. `$pageview`, `signup_completed` |
| `url` | URL | from `properties.$current_url`, if present |
| `occurredAt` | DATE_TIME | event timestamp from PostHog |
| `person` | RELATION → person | matched by email |
| `rawProperties` | TEXT | JSON blob of other properties (optional, for debugging) |

Add one field to standard `person`:

| Field | Type | Notes |
|---|---|---|
| `posthogSyncedAt` | DATE_TIME | last successful sync, used to avoid redundant queries |

## 6. Function

```ts
// src/functions/sync-posthog-activity.function.ts
import { TwentyFunction, CoreApiClient } from 'twenty-sdk';

export const syncPosthogActivity = new TwentyFunction({
  name: 'syncPosthogActivity',
  triggers: [{ type: 'schedule', cron: '*/15 * * * *' }], // every 15 minutes
  handler: async (input, context) => {
    const { coreApiClient } = context;
    const people = await coreApiClient.findMany('person', {
      filter: { email: { isNotNull: true } },
      limit: 100, // batch size — paginate across runs for large workspaces
    });

    for (const person of people) {
      const res = await fetch(
        `${process.env.POSTHOG_HOST}/api/projects/${process.env.POSTHOG_PROJECT_ID}/query/`,
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${process.env.POSTHOG_PERSONAL_API_KEY}`,
          },
          body: JSON.stringify({
            query: {
              kind: 'HogQLQuery',
              query: `select event, timestamp, properties.$current_url
                       from events
                       where person.properties.email = '${person.email}'
                       order by timestamp desc limit 20`,
            },
          }),
        },
      );
      const { results } = await res.json();
      if (!results?.length) continue;

      await coreApiClient.createMany(
        'webActivity',
        results.map((row) => ({
          eventName: row[0],
          occurredAt: row[1],
          url: row[2],
          person: { connect: person.id },
        })),
      );
      await coreApiClient.updateOne('person', person.id, {
        posthogSyncedAt: new Date().toISOString(),
      });
    }
  },
});
```

Notes on the PostHog API itself:

- Auth is a **Personal API Key** (`phx_...`) sent as a Bearer token — scope
  it to `query:read` only.
- The `/query/` endpoint accepts a `HogQLQuery` (PostHog's SQL dialect) —
  this is the most flexible way to filter events by a person property like
  email, and works identically against PostHog Cloud or a self-hosted
  instance (just change the host).
- Default query cap is 100 rows; you can request up to 50k with an
  explicit `LIMIT` before PostHog recommends pagination — irrelevant for a
  per-person query capped at 20, but relevant if you later build an
  account-level rollup.
- There's also a simpler `GET /api/projects/:project_id/events` endpoint
  with `person_id`/`distinct_id` filters if you want to avoid HogQL
  entirely for v1 — slightly less flexible but faster to implement.

## 7. Dedup strategy

Since this runs on a schedule rather than reacting to a webhook, avoid
re-inserting the same events every 15 minutes: either (a) always query
`where timestamp > person.posthogSyncedAt` so each run only pulls new
events, or (b) dedupe in-app on `(eventName, occurredAt, person)` before
insert. Option (a) is simpler and cheaper — use it for v1.

## 8. UI

Front-component on the Person record: a compact "Recent Activity" list
(icon + event name + relative time + link if `url` present), sorted
newest-first, capped at ~10 visible rows with a "show more" expansion.

## 9. Roles & permissions

Manifest needs read/write on `person` and the new `webActivity` object.
`POSTHOG_PERSONAL_API_KEY` stored as a `secret`-type app setting.

## 10. Build plan for Claude Code

1. `npx create-twenty-app posthog-activity-sync`.
2. Get a PostHog Personal API Key scoped to `query:read` from a test
   project; confirm a manual `curl` against `/query/` works before writing
   any Twenty code.
3. Define the `webActivity` object and the `posthogSyncedAt` field on
   `person`.
4. Write `syncPosthogActivity` with the incremental-timestamp filter.
5. `yarn twenty app:dev`, manually trigger via
   `yarn twenty function:execute -n syncPosthogActivity`, confirm records
   land correctly for a test person with known PostHog activity.
6. Build the front-component.
7. Tune the cron interval and batch size against real data volume before
   deploying to a live workspace.

## 11. Testing plan

- Unit test the HogQL query construction (escaping the email value safely
  — don't string-interpolate untrusted input directly into SQL; use
  parameterization if the SDK/PostHog client supports it, or strict
  sanitization if not).
- Test with a person who has zero PostHog activity — should not error,
  should just skip.
- Test the incremental filter — run twice in a row, confirm no duplicate
  `webActivity` records on the second run.
- Load-test the batch loop against a workspace with hundreds of `person`
  records to confirm the 15-minute cron window is enough time to complete
  a full pass (adjust batch size / add pagination across runs if not).

## 12. Risks / open questions

- **SQL injection surface**: building HogQL query strings by interpolating
  the person's email is a real risk if any email field ever contains
  unexpected characters — sanitize or parameterize before shipping.
- **Rate limits / cost on large workspaces**: querying PostHog once per
  person every 15 minutes doesn't scale gracefully past a few hundred
  contacts — for a larger client, restructure to a single batched HogQL
  query with an `IN (...)` clause across all synced emails rather than
  one request per person.
- **Matching only works by email** — contacts without an email, or whose
  PostHog identity is tied to an anonymous/pre-signup distinct_id, won't
  show activity until identity resolution happens on the PostHog side.

## Monetization

No platform-native payment rails in Twenty — invoice the client directly
(see this PRD set's general monetization research).

This is a well-scoped, lower-effort build compared to WhatsApp — price it
as a **fixed one-time fee** rather than open-ended hours; the clean
bearer-token API and lack of business-verification overhead make the
build time predictable enough to quote confidently. Best positioned as a
**bundled feature inside a HubSpot-to-Twenty migration engagement**
(pairs naturally with the migration-utility idea in this set) rather than
sold as a standalone product — the person who most wants this is,
specifically, someone already migrating off HubSpot.

## 13. References

- [Discussion #16735 — Native integrations wishlist](https://github.com/twentyhq/twenty/discussions/16735) (PostHog callout from `benweissbehavehealth`'s conversation with Felix Malfait)
- PostHog Query API: `posthog.com/docs/api/query`
- PostHog Events API: `posthog.com/docs/api/events` (archive reference)
- Twenty Custom Apps guide: `docs.twenty.com/developers/extend/apps`
