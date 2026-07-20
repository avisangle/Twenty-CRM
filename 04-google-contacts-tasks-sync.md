# PRD: Google Contacts & Google Tasks Sync for Twenty CRM

**App slug:** `google-contacts-tasks-sync`
**Type:** Twenty App (SDK) — Functions (OAuth + scheduled + database event) + Front-component (optional)
**Effort:** Low-medium (4-6 days for v1, contacts + tasks as two phases)
**External dependencies:** Google Cloud project, OAuth 2.0 client, People API + Tasks API enabled

---

## 1. What this is

Two-way sync between Twenty and Google's Contacts (People API) and Tasks
(Tasks API) — Twenty `person` records stay aligned with a user's Google
Contacts, and Twenty `task` records can optionally push to a Google Tasks
list so they show up on phone/calendar apps outside Twenty.

## 2. Why this idea — what the research showed

The curated integrations wishlist
([Discussion #16735](https://github.com/twentyhq/twenty/discussions/16735))
explicitly calls out this as an odd, unaddressed gap: Twenty already has
native Gmail and Google Calendar integration, but **Google Contacts sync**
and **Google Tasks sync** are separately listed as open requests
([#14985](https://github.com/twentyhq/twenty/discussions/14985) and
[#14040](https://github.com/twentyhq/twenty/discussions/14040)) with no
native or community solution found. This is lower-drama than WhatsApp (no
huge upvote count, no urgency), but it's a clean, well-documented,
stable-API gap that nobody else appears to have picked up — good
portfolio/credibility piece precisely because the underlying Google APIs
are boring and reliable rather than novel.

## 3. Goals / success criteria

- New/updated Twenty `person` records with an email match sync to the
  user's Google Contacts (create or update).
- New Google Contacts (outside Twenty) can be pulled in as new `person`
  records on a schedule.
- Twenty `task` records optionally mirror to a dedicated Google Tasks list
  so they appear in Google's task UI and any connected calendar/phone
  widget.
- No duplicate contacts/tasks created on repeated syncs (idempotent by
  external ID).

## 4. Architecture overview

```
OAuth (one-time, per workspace or per user)
   │
   ▼
person.created / person.updated (database event) ──▶ pushContactToGoogle
                                                            │
                                                            ▼
                                            People API: people.createContact /
                                            people.updateContact

Scheduled function (every N min) ──▶ pullGoogleContacts
                                            │
                                            ▼
                              People API: people.connections.list
                              (paginated, delta via syncToken)
                                            │
                                            ▼
                        coreApiClient.createOne/updateOne('person', ...)

task.created (database event) ──▶ pushTaskToGoogle
                                        │
                                        ▼
                          Tasks API: tasks.insert (into a dedicated "Twenty" tasklist)
```

## 5. Data model

Add fields to standard `person`:

| Field | Type | Notes |
|---|---|---|
| `googleContactResourceName` | TEXT | People API `resourceName` (e.g. `people/c12345`), for idempotent updates |
| `googleContactsSyncToken` | TEXT | store per-workspace, not per-person — see §7 |

Add fields to standard `task`:

| Field | Type | Notes |
|---|---|---|
| `googleTaskId` | TEXT | Tasks API task ID, for idempotent updates |
| `googleTaskListId` | TEXT | which Google tasklist it lives in |

## 6. Auth

Both APIs use standard Google OAuth 2.0. Scopes needed:

- People API: `https://www.googleapis.com/auth/contacts` (read/write) —
  this is a **sensitive scope** requiring Google's app verification process
  before it can be used by anyone outside test users; `contacts.readonly`
  is non-sensitive if you only need the pull direction for v1.
- Tasks API: `https://www.googleapis.com/auth/tasks` (read/write), or
  `tasks.readonly` for read-only.

Store the refresh token as a `secret`-type app setting; refresh access
tokens on each function invocation (they're short-lived).

## 7. Functions

### a) `pushContactToGoogle` — database event trigger

```ts
export const pushContactToGoogle = new TwentyFunction({
  name: 'pushContactToGoogle',
  triggers: [
    { type: 'databaseEvent', objectName: 'person', action: 'created' },
    { type: 'databaseEvent', objectName: 'person', action: 'updated',
      updatedFields: ['firstName', 'lastName', 'email', 'phone'] },
  ],
  handler: async (input, context) => {
    // If person.googleContactResourceName exists → people.updateContact
    // Else → people.createContact, then store the returned resourceName back on the person
  },
});
```

People API endpoints (v1, `https://people.googleapis.com/v1/`):
- Create: `POST /people:createContact`
- Update: `PATCH /people/{resourceName}:updateContact` (requires
  `updatePersonFields` query param listing which fields changed, and the
  contact's current `etag` to avoid overwriting concurrent edits)

### b) `pullGoogleContacts` — scheduled trigger

```ts
export const pullGoogleContacts = new TwentyFunction({
  name: 'pullGoogleContacts',
  triggers: [{ type: 'schedule', cron: '*/30 * * * *' }],
  handler: async (input, context) => {
    // GET /v1/people/me/connections?personFields=names,emailAddresses,phoneNumbers
    //   &syncToken=<stored token>  (omit on first run for a full sync)
    // Store the returned nextSyncToken for the next incremental run
    // For each connection: find-or-create matching person by email
  },
});
```

Use the People API's **sync token** mechanism (`requestSyncToken: true` on
the first call, then pass the returned token on subsequent calls) rather
than re-pulling the full contact list every run — this is the documented,
efficient pattern for ongoing sync rather than a one-off import.

### c) `pushTaskToGoogle` — database event trigger

```ts
export const pushTaskToGoogle = new TwentyFunction({
  name: 'pushTaskToGoogle',
  triggers: [{ type: 'databaseEvent', objectName: 'task', action: 'created' }],
  handler: async (input, context) => {
    // tasks.insert({ tasklist: <TWENTY_TASKLIST_ID>, requestBody: { title, notes, due } })
    // store returned task.id back on the Twenty task as googleTaskId
  },
});
```

Tasks API base: `https://tasks.googleapis.com/tasks/v1/`. Key endpoints:
`GET /users/@me/lists` (find or create a dedicated "Twenty" tasklist once,
store its ID as an app setting), `POST /lists/{tasklistId}/tasks` to
create.

## 8. Roles & permissions

Manifest needs read/write on `person` and `task`. All Google credentials
(`client_id`, `client_secret`, `refresh_token`) as `secret`-type settings.

## 9. Build plan for Claude Code

1. Create a Google Cloud project, enable People API and Tasks API, create
   an OAuth 2.0 client (web application type), configure the consent
   screen.
2. Decide v1 scope: read-only pull (non-sensitive scope, ships same day,
   no Google verification needed) vs. full read/write (sensitive scope,
   needs Google's verification review — budget for that lead time
   separately from the coding work).
3. `npx create-twenty-app google-contacts-tasks-sync`.
4. Build the OAuth handshake as a route-trigger function (`/google/oauth/callback`)
   that exchanges the code for tokens and stores the refresh token.
5. Add the `googleContactResourceName`/`googleTaskId` fields.
6. Build `pullGoogleContacts` first (read-only, lowest risk, fastest to
   demo) — get the sync-token pagination working correctly before adding
   push.
7. Build `pushContactToGoogle`.
8. Build the Tasks-side functions.
9. Test both directions against a real Google account with a handful of
   contacts/tasks.

## 10. Testing plan

- Test the sync-token flow explicitly: first run (full sync, no token) →
  add a contact in Google → second run (incremental, with token) → confirm
  only the new contact is pulled, not the whole list again.
- Test conflict handling: edit the same contact in both Twenty and Google
  between syncs, confirm the `etag` check prevents a silent overwrite (or
  document the "last write wins" behavior if that's the chosen tradeoff).
- Test task push with tasks that have due dates vs. tasks without — Google
  Tasks API treats `due` as optional.
- Confirm idempotency: re-running `pullGoogleContacts` twice in a row with
  no new Google-side changes creates zero duplicate `person` records.

## 11. Risks / open questions

- **Google app verification**: any scope beyond `contacts.readonly` /
  `tasks.readonly` requires Google's OAuth verification process for
  production use with real (non-test) users — this can take days to
  weeks and needs a privacy policy URL, homepage, and justification video
  for sensitive scopes. Scope v1 to read-only if the client needs
  something live quickly.
- **Conflict resolution is genuinely unsolved** in this PRD — decide with
  the client whether Twenty or Google is the source of truth per field,
  or accept last-write-wins for v1 and revisit if it causes problems.
- **Rate limits**: both APIs have standard per-project quotas; fine for a
  single small-business workspace, worth checking quota dashboards before
  syncing a workspace with thousands of contacts.

## Monetization

No platform-native payment rails in Twenty — invoice the client directly
(see this PRD set's general monetization research).

This is the **weakest monetization case of the six**. Demand exists but is
low-urgency — nobody in the research was blocked or upset about this gap,
just mildly inconvenienced. Don't price it as a standalone deliverable;
fold it into a broader Twenty setup/customization retainer as a small
bundled feature, or treat it as a portfolio item that demonstrates clean
OAuth-integration work rather than something to lead a sales conversation
with.

## 12. References

- [Discussion #14985 — Add Google Contacts sync](https://github.com/twentyhq/twenty/discussions/14985)
- [Discussion #14040 — Add Google Tasks sync to Tasks module](https://github.com/twentyhq/twenty/discussions/14040)
- [Discussion #16735 — Native integrations wishlist](https://github.com/twentyhq/twenty/discussions/16735)
- Google People API reference: `developers.google.com/people/api/rest`
- Google Tasks API reference: `developers.google.com/workspace/tasks/reference/rest`
- Google OAuth scopes reference: `developers.google.com/identity/protocols/oauth2/scopes`
