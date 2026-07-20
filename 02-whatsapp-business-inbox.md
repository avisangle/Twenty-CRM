# PRD: WhatsApp Business Inbox App for Twenty CRM

**App slug:** `whatsapp-inbox`
**Type:** Twenty App (SDK) — Object + Functions (webhook + route) + Front-component
**Effort:** Medium-high (2-3 weeks for a solid v1, plus Meta approval lead time)
**External dependencies:** Meta WhatsApp Cloud API (Business verification required for production)

---

## 1. What this is

A Twenty app that syncs WhatsApp Business conversations onto Person/Company
records: inbound messages arrive via a Meta webhook and are written to a
custom `WhatsAppMessage` object linked to the matching Person; a
front-component renders the thread on the record page with a reply box that
sends via the Cloud API.

## 2. Why this idea — what the research showed

This is the **single most-requested feature in Twenty's entire GitHub
Discussions history**:

- [Discussion #7296 "Whatsapp Business sync"](https://github.com/twentyhq/twenty/discussions/7296) — opened Sep 2024, **77 upvotes**, the highest-voted item found in this research, still open as of mid-2026, 26 participants.
- Demand is explicitly geographic and recurring across separate commenters: "whatsapp is ubiquitous in India"; a Brazil-based commenter described growing demand especially among micro/small/medium businesses whose sales mostly run through the channel; commenters from Mexico and general Latin America made the same point — WhatsApp is described as the primary sales channel, ahead of email, in these markets.
- A maintainer (`Bonapara`) called it "a great idea" on day one; a community member (`laforetcan`) announced in Oct 2025 they'd started building it for their own business and would try to keep it generic; a maintainer pointed builders to `twenty-cli` (beta) as the way to start.
- **As of early 2026 it still had not shipped natively.** A maintainer stated WhatsApp integration is sequenced *after* mail-attachment support and in-app notifications land, "in stages," and separately told one builder to wait until the app SDK stabilizes toward v1.0 before building it properly — a real caution on timing, not just a formality.
- The curated integrations wishlist ([Discussion #16735](https://github.com/twentyhq/twenty/discussions/16735)) classifies WhatsApp as one that *should* eventually be a **native** integration (not a lightweight plugin) specifically because of API complexity and expected request volume — meaning even Twenty's own maintainers see this as a substantial build, not a weekend plugin.
- The core team indicated internally that **native integration work (as a category) starts Q3/Q4 2026** — i.e., roughly now. This is simultaneously the best and the last easy window to ship a third-party version before the vendor's own team potentially ships something that competes with or obsoletes it.

## 3. Goals / success criteria

- Inbound WhatsApp messages appear on the correct Person record within
  seconds, matched by phone number.
- Users can reply from inside Twenty using approved message templates
  (Meta requires templates to *initiate* a conversation outside the 24-hour
  customer-service window — see §7).
- No message loss on webhook retries (Meta retries up to 7 days on
  non-200 responses — idempotency is mandatory, see §8).
- New inbound contacts with unrecognized phone numbers create a new
  `person` record rather than being dropped.

## 4. Architecture overview

```
Meta WhatsApp Cloud API
   │  GET  (webhook verification, one-time)
   │  POST (inbound message / status webhook)
   ▼
receiveWhatsAppMessage (TwentyFunction, webhook trigger)
   │  verify X-Hub-Signature-256 (HMAC-SHA256), respond 200 fast
   │  match/create person by phone number
   ▼
coreApiClient.createOne('whatsAppMessage', {...})
   │
   ▼
Front-component on Person record: thread view + reply box
   │  on send →
   ▼
sendWhatsAppMessage (TwentyFunction, route trigger)
   │  POST to Meta Graph API /messages endpoint
   ▼
coreApiClient.createOne('whatsAppMessage', { direction: 'outbound', ... })
```

## 5. Data model

New custom object `whatsAppMessage`:

| Field | Type | Notes |
|---|---|---|
| `waMessageId` | TEXT | Meta's message ID, used for idempotency (unique) |
| `direction` | SELECT | `inbound` / `outbound` |
| `body` | TEXT | message text |
| `messageType` | SELECT | `text`, `image`, `document`, `template`, etc. |
| `mediaUrl` | URL | resolved media link, if applicable |
| `status` | SELECT | `sent`, `delivered`, `read`, `failed` (from status webhooks) |
| `timestamp` | DATE_TIME | from Meta payload |
| `person` | RELATION → person | matched/created by phone number |
| `rawPayload` | TEXT | store the raw JSON for debugging/audit (optional) |

Add a `whatsappPhone` or reuse the standard `phone` field on `person` as
the match key — normalize to E.164 format on both sides.

## 6. Functions

### a) `receiveWhatsAppMessage` — webhook trigger

Handles both the one-time GET verification challenge and ongoing POST
events. Meta's webhook contract:

- **Verification (GET):** Meta calls your endpoint with
  `hub.mode`, `hub.verify_token`, `hub.challenge` query params. Your
  handler must echo back `hub.challenge` if `hub.verify_token` matches a
  secret you configured in the app's settings. This must happen before
  Meta will let you save the webhook URL in the App Dashboard.
- **Events (POST):** JSON payload under `entry[].changes[].value.messages[]`
  (inbound messages) or `.statuses[]` (delivery/read status updates).
  Verify the `X-Hub-Signature-256` header (HMAC-SHA256 of the raw body,
  keyed with your app secret) before trusting the payload.
- **Response time matters:** Meta expects a fast 200 response; slow or
  non-200 responses trigger retries with decreasing frequency for up to
  7 days, which can produce duplicate deliveries — **dedupe on
  `waMessageId`** before inserting.
- Payloads can be up to 3MB.

```ts
// src/functions/receive-whatsapp-message.function.ts
export const receiveWhatsAppMessage = new TwentyFunction({
  name: 'receiveWhatsAppMessage',
  triggers: [{ type: 'webhook', name: 'whatsapp-inbound' }],
  handler: async (input, context) => {
    // 1. If GET verification request: return hub.challenge if token matches
    // 2. Verify X-Hub-Signature-256
    // 3. Parse entry[].changes[].value.messages[]
    // 4. For each message: check waMessageId not already stored (idempotency)
    // 5. Find or create person by phone number
    // 6. coreApiClient.createOne('whatsAppMessage', {...})
    // 7. Return 200 immediately; do heavy work async if needed
  },
});
```

### b) `sendWhatsAppMessage` — route trigger

```ts
// src/functions/send-whatsapp-message.function.ts
export const sendWhatsAppMessage = new TwentyFunction({
  name: 'sendWhatsAppMessage',
  triggers: [{ type: 'route', method: 'POST', path: '/whatsapp/send' }],
  handler: async (input, context) => {
    // POST to https://graph.facebook.com/v<version>/<phone_number_id>/messages
    // Authorization: Bearer <WHATSAPP_ACCESS_TOKEN> (from app settings, secret type)
    // Body: { messaging_product: "whatsapp", to, type, template|text: {...} }
    // On success, coreApiClient.createOne('whatsAppMessage', { direction: 'outbound', ... })
  },
});
```

## 7. Critical Meta platform constraints (must design around these)

- **Template requirement:** you can only *initiate* a conversation (i.e.,
  message a customer who hasn't messaged you in the last 24 hours) using a
  Meta-pre-approved message template. Free-form replies are only allowed
  within a 24-hour window after the customer's last inbound message. Build
  the reply UI to reflect this — show a "send template" flow when outside
  the window.
- **Business verification:** production access (beyond a handful of test
  numbers) requires Meta Business verification and a dedicated phone
  number — this is a real lead-time item (days to weeks), not a
  same-day API key.
- **Two integration paths:** direct to Meta (more setup, full control) or
  via an official Business Solution Provider / BSP (faster onboarding,
  BSP takes a cut or passes through Meta's per-conversation cost). Decide
  this with the client before scoping hours.
- **Pricing model:** Meta charges per-24-hour conversation initiated, not
  per message — relevant if you're advising a client on cost.

## 8. Idempotency & reliability requirements

- Unique constraint (application-level, since Twenty's object fields don't
  enforce DB uniqueness the same way) on `waMessageId` before insert.
- Webhook handler must return 200 within a few seconds even under load —
  offload slow work (media download, person matching against a large
  dataset) rather than blocking the response.
- Support **webhook overrides** (per-phone-number endpoints) if the
  target client has multiple WhatsApp Business numbers.

## 9. UI

Front-component on the Person record: chronological thread (reuse the
visual pattern of Twenty's native email thread if possible), a reply
composer that switches between free-form text (in-window) and template
picker (out-of-window), and a status indicator (sent/delivered/read) per
message using the `statuses[]` webhook data.

## 10. Roles & permissions

Manifest needs read/write on `person` (to create/match contacts) and
read/write on the new `whatsAppMessage` object. Store `WHATSAPP_ACCESS_TOKEN`
and `WHATSAPP_APP_SECRET` as `secret`-type app settings, never in code.

## 11. Build plan for Claude Code

1. Set up a Meta Developer app + WhatsApp product in test mode; get a test
   phone number and temporary access token before writing any code.
2. `npx create-twenty-app whatsapp-inbox`.
3. Define the `whatsAppMessage` object and relation to `person`.
4. Build `receiveWhatsAppMessage`: verification handshake first (get this
   working and confirmed in the Meta dashboard before anything else), then
   message parsing, then idempotent insert.
5. Build `sendWhatsAppMessage` with the Graph API call.
6. Build the front-component thread view + composer.
7. Test end-to-end against the Meta test number: send yourself a message,
   confirm it lands on the right Person; reply from Twenty, confirm
   delivery.
8. Add status-webhook handling (`sent`/`delivered`/`read`).
9. Document the Business-verification / template-approval steps separately
   for whoever owns the client relationship — this is not something code
   can shortcut.

## 12. Testing plan

- Use Meta's sample webhook test payloads (available in their docs)
  before going live.
- Test webhook signature verification with a deliberately-tampered
  payload — must reject.
- Test duplicate delivery (replay the same payload twice) — must not
  create two `whatsAppMessage` records.
- Test the 24-hour window boundary — sending a free-form message just
  outside the window should fail gracefully and prompt for a template.

## 13. Risks / open questions

- **SDK maturity**: a Twenty maintainer explicitly told a community
  builder to wait for a more stable SDK/v1.0 before shipping this — treat
  the current alpha as a real constraint on production-readiness, not
  boilerplate caution.
- **Compliance/contract overhead**: Meta Business verification and
  template approval are outside your control and can stall a client
  timeline by days-to-weeks — set expectations up front.
- **Vendor may ship this natively**: this is explicitly the top item on
  Twenty's own native-integration wishlist; if/when the core team builds
  it, a third-party version needs a differentiation angle (e.g., multi-BSP
  support, or shipping months before the vendor does).
- **Other messengers requested in the same thread** (Telegram, Messenger,
  Matrix) — if the client base is India/LatAm-heavy, WhatsApp alone likely
  covers the vast majority of value; don't scope-creep into a universal
  messaging abstraction for v1.

## Monetization

Twenty has no app-store payment rails (see the general note in this PRD
set's monetization research) — you invoice the client directly, outside
the platform.

This is the **highest-revenue-potential idea** of the six, specifically
*because* it's painful: real, validated demand plus genuine Meta
Business-verification and template-approval overhead means clients will
pay to have someone else own that complexity rather than DIY it. Two
models worth combining:

- **One-time implementation fee + monthly retainer**: charge a fixed fee
  for the initial build (scoped to data volume and whether multi-agent
  inbox support is needed), plus an ongoing retainer for template
  management, delivery monitoring, and handling Meta's periodic API/policy
  changes — this mirrors what implementation partners in Twenty's own
  ecosystem already charge for comparable scoped work.
- **Build once, resell the codebase**: since nothing here is distributed
  through a Twenty marketplace, the app you build is entirely your own IP.
  Once built and hardened for one client, deploying it for the next is
  mostly swapping Meta credentials and phone numbers, not a rebuild —
  this is where the real leverage is, especially given how many separate
  commenters across India, Brazil, and Mexico expressed the same need.

## 14. References

- [Discussion #7296 — WhatsApp Business sync](https://github.com/twentyhq/twenty/discussions/7296) (77 upvotes)
- [Discussion #16735 — Native integrations wishlist](https://github.com/twentyhq/twenty/discussions/16735)
- Meta WhatsApp Cloud API docs: `developers.facebook.com/docs/whatsapp`
- Meta webhooks overview: `developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/overview`
- Twenty Custom Apps guide: `docs.twenty.com/developers/extend/apps`
