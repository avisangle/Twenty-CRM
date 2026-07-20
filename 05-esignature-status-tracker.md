# PRD: E-Signature Status Tracker for Twenty CRM (DocuSign / PandaDoc)

**App slug:** `esignature-tracker`
**Type:** Twenty App (SDK) — Object + Functions (webhook + route) + Front-component
**Effort:** Low-medium (4-6 days for one provider; +2-3 days per additional provider)
**External dependencies:** DocuSign developer account (Connect) and/or PandaDoc API key

---

## 1. What this is

Attaches e-signature document status to Twenty Opportunities: when a
contract is sent, viewed, signed, or completed in DocuSign or PandaDoc, a
webhook updates a linked record in Twenty automatically, so sales can see
"contract sent 3 days ago, still unsigned" without opening another tool.

## 2. Why this idea — what the research showed

Listed explicitly in the curated integrations wishlist
([Discussion #16735](https://github.com/twentyhq/twenty/discussions/16735))
under "Documents," described as requested by several users and expected to
land once file handling in Twenty matures. Named candidates: DocuSign,
DocuSeal, PandaDoc; RabbitSign was explicitly ruled out by the list's
maintainer as not viable because its API exposes too little to build
anything meaningful. Demand here is real but was found at lower volume
than WhatsApp or PostHog in this research — position this as a solid,
technically clean build rather than an urgently-contested one.

Of the two realistic candidates, **DocuSign and PandaDoc both have clean,
well-documented webhook systems**, which makes this one of the more
predictable builds in this set — no scraping, no unofficial APIs, no
business-verification bottleneck like WhatsApp.

## 3. Goals / success criteria

- An Opportunity (or Company) can be linked to a sent envelope/document.
- Status updates (`sent`, `viewed`, `signed`, `completed`, `declined`,
  `voided`) reflect in Twenty within seconds of the event happening in the
  provider.
- The completed, signed document (or a link to it) is accessible from the
  Twenty record.

## 4. Architecture overview

```
sendForSignature (route trigger, called from a button/front-component)
   │
   ▼
DocuSign: POST /accounts/{accountId}/envelopes  (create + send envelope,
          with eventNotification.url pointing at our webhook function)
   or
PandaDoc: POST /public/v1/documents            (create + send document)
   │
   ▼
coreApiClient.createOne('signatureRequest', { status: 'sent', ... })


Provider webhook (DocuSign Connect / PandaDoc webhook)
   │
   ▼
receiveSignatureEvent (webhook trigger)
   │  verify payload, map provider status → internal status
   ▼
coreApiClient.updateOne('signatureRequest', ..., { status, completedAt, documentUrl })
```

## 5. Data model

New custom object `signatureRequest`:

| Field | Type | Notes |
|---|---|---|
| `provider` | SELECT | `docusign` / `pandadoc` |
| `externalId` | TEXT | DocuSign `envelopeId` or PandaDoc `document.id` |
| `status` | SELECT | `sent`, `viewed`, `signed`, `completed`, `declined`, `voided` |
| `documentUrl` | URL | link to the (completed) document, once available |
| `sentAt` / `completedAt` | DATE_TIME | timestamps from provider events |
| `opportunity` | RELATION → opportunity | what this document is attached to |

## 6. DocuSign integration details

- **Auth**: OAuth 2.0 (JWT grant recommended for server-to-server / no
  interactive login) against DocuSign's Authentication API; store the
  private key and integration key as `secret` app settings.
- **Sending**: `POST /restapi/v2.1/accounts/{accountId}/envelopes` with an
  `eventNotification` block on the envelope definition — this is how you
  register the webhook **per envelope**, which is simpler than managing a
  global Connect subscription for a first version:

```
eventNotification: {
  url: "<our webhook function URL>",
  requireAcknowledgment: true,
  loggingEnabled: true,
  envelopeEvents: [
    { envelopeEventStatusCode: "sent" },
    { envelopeEventStatusCode: "delivered" },
    { envelopeEventStatusCode: "completed" },
    { envelopeEventStatusCode: "declined" },
    { envelopeEventStatusCode: "voided" },
  ],
}
```

- **Known gotcha found in research**: some DocuSign integrators report
  only receiving `envelope-completed` and not the per-recipient events
  (`recipient-signed`, `recipient-completed`) even when subscribed — if
  the client needs per-signer granularity (e.g. multi-signer contracts),
  test this explicitly early rather than assuming recipient-level events
  will reliably arrive; envelope-level status is the safe baseline to
  design around.
- **Payload delivery**: DocuSign retries webhook delivery on failure;
  design the handler idempotently keyed on `envelopeId` + status.

## 7. PandaDoc integration details

- **Auth**: API key (Bearer token) — simpler than DocuSign's OAuth/JWT
  setup, good candidate to build first if you want the faster demo.
- **Webhooks**: configured in the PandaDoc dashboard (Developer Dashboard
  → Webhooks) or via API, subscribing to events including
  `document_state_changed`; the payload includes the document `id` and new
  `status` (`document.draft`, `document.sent`, `document.viewed`,
  `document.completed`, `document.declined`, etc.).

## 8. Function: `receiveSignatureEvent`

```ts
export const receiveSignatureEvent = new TwentyFunction({
  name: 'receiveSignatureEvent',
  triggers: [{ type: 'webhook', name: 'esignature-events' }],
  handler: async (input, context) => {
    const provider = input.data.provider; // determined by route or payload shape
    const { externalId, status, documentUrl } = mapProviderPayload(provider, input.data);

    const existing = await context.coreApiClient.findMany('signatureRequest', {
      filter: { externalId: { eq: externalId } },
      limit: 1,
    });
    if (!existing.length) return { success: false, error: 'unknown externalId' };

    await context.coreApiClient.updateOne('signatureRequest', existing[0].id, {
      status,
      documentUrl,
      completedAt: status === 'completed' ? new Date().toISOString() : undefined,
    });
    return { success: true };
  },
});
```

## 9. UI

Front-component on the Opportunity record: a status badge (color-coded by
`status`), sent/completed timestamps, and a "View document" link once
`documentUrl` is populated. Optionally a "Send for signature" button that
calls the `sendForSignature` route function directly from the record page.

## 10. Roles & permissions

Manifest needs read/write on `opportunity` and the new `signatureRequest`
object. Provider credentials as `secret`-type settings, scoped separately
per provider if supporting both.

## 11. Build plan for Claude Code

1. Pick one provider for v1 — **PandaDoc is the faster path** (API-key
   auth, no JWT setup) if the goal is a quick working demo; DocuSign if
   the target client base already uses it.
2. `npx create-twenty-app esignature-tracker`.
3. Define the `signatureRequest` object and its relation to `opportunity`.
4. Build `receiveSignatureEvent` first against the chosen provider's
   sandbox/test account — get the webhook verified and receiving real
   test events before building the send flow.
5. Build `sendForSignature` (route trigger) to create/send a
   document/envelope and register the webhook.
6. Build the front-component.
7. Test the full loop: send from Twenty → sign in a test browser session →
   confirm status updates back in Twenty within seconds.
8. (Phase 2) Add the second provider behind the same `signatureRequest`
   object using the `provider` field to branch logic.

## 12. Testing plan

- Test each status transition explicitly (sent → viewed → completed, and
  separately sent → declined) against the provider's sandbox.
- Test webhook idempotency: replay the same event payload twice, confirm
  no duplicate updates or errors.
- Test the "unknown externalId" path (a webhook arrives for a document not
  created through this app) — should fail gracefully, not crash.
- For DocuSign specifically: verify whether recipient-level events are
  actually being received in your test account before promising
  per-signer status to a client.

## 13. Risks / open questions

- **DocuSign recipient-event reliability** — flagged above, confirm before
  committing to a multi-signer feature.
- **File/storage maturity in Twenty**: the wishlist notes this category is
  expected to land "once Files are surfaced" more fully in Twenty's own
  object model — worth checking current file-attachment capabilities in
  the SDK before finalizing how `documentUrl` is stored/displayed (a
  direct external link is the safe fallback if native file attachment
  isn't there yet).
- **Lower validated demand than WhatsApp/PostHog** — good technical
  portfolio piece, but confirm a specific client need before over-building
  both providers.

## Monetization

No platform-native payment rails in Twenty — invoice the client directly
(see this PRD set's general monetization research). Note this is a
different thing from *building payment/e-signature tracking into a
client's CRM* — that's a deliverable you charge for; it's not Twenty
processing money on your behalf either way.

Mid-tier revenue potential: price as a **fixed fee per provider
integrated** (a client wanting both DocuSign and PandaDoc is two scoped
fees, not one, since the auth models and webhook shapes genuinely differ).
Recurring revenue is plausible via a maintenance retainer if the client
wants someone monitoring webhook delivery reliability and handling
provider-side API changes over time — less compelling as a one-and-done
sale on its own.

## 14. References

- [Discussion #16735 — Native integrations wishlist](https://github.com/twentyhq/twenty/discussions/16735) (Documents section)
- DocuSign Connect (webhooks) docs: `developers.docusign.com/platform/webhooks/connect/`
- PandaDoc API docs: `developers.pandadoc.com`
- Twenty Custom Apps guide: `docs.twenty.com/developers/extend/apps`
