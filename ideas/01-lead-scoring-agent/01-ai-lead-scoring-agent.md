# PRD: AI Lead Scoring & Enrichment Agent for Twenty CRM

**App slug:** `lead-scoring-agent`
**Type:** Twenty App (SDK) — Agent + Skill + Logic Function
**Effort:** Low-medium (2-4 days for v1)
**External dependencies:** None beyond Twenty's built-in AI credits

---

## 1. What this is

An in-workspace AI agent that runs automatically whenever a new `person` or
`opportunity` record is created (or updated), reads the record plus related
context, and writes back a lead score (0-100) and a short reasoning summary.
Optionally drafts a first outreach message as a second skill.

This is the lowest-risk, highest-polish idea of the six PRDs in this set: it
needs no external API keys, no OAuth, no webhook contract with a third
party — everything happens inside Twenty using the SDK's own AI primitives.

## 2. Why this idea — what the research showed

- Twenty's SDK documentation (`docs.twenty.com/developers/extend/apps/logic/skills-and-agents`)
  uses **a lead-scoring agent as its own canonical example** of
  `defineAgent()` with structured JSON output. That means this isn't a
  novel idea bolted onto the platform — it's the pattern Twenty's own docs
  point developers toward first.
- Skills & Agents are explicitly marked **alpha** in the docs ("the feature
  works but is still evolving") — expect API surface changes before v1.0.
- Twenty's GitHub repo tagline changed to *"The open alternative to
  Salesforce, designed for AI"* and the package list includes
  `twenty-claude-skills` and `twenty-codex-plugin` — the core team is
  investing specifically in AI-agent-shaped extensibility, which is a good
  signal this primitive won't be deprecated.
- Unlike the WhatsApp/PostHog/migration ideas in this set, no GitHub
  Discussion thread was found specifically begging for "lead scoring" —
  this idea is doc-driven and strategy-driven, not demand-driven. Treat it
  as the fastest way to build something real and demoable on the newest
  part of the SDK, not as a validated-market bet.

## 3. Goals / success criteria

- New `person` and `opportunity` records get an automatic score within
  seconds of creation, without any manual action.
- Score and reasoning are visible directly on the record (custom fields +
  optional front-component badge).
- Zero manual re-triggering needed — re-scores on relevant field updates.
- App installs cleanly into any Twenty Cloud or self-hosted workspace via
  `yarn twenty app:sync`.

## 4. Architecture overview

```
person.created / opportunity.created (database event)
        │
        ▼
scoreLeadFunction (TwentyFunction, databaseEvent trigger)
        │  guards against re-scoring loops
        ▼
runAgent({ agentUniversalIdentifier: lead-scorer, prompt: <record context> })
        │
        ▼
Agent (defineAgent, responseFormat: json schema) → { score, summary }
        │
        ▼
coreApiClient.updateOne('person', id, { leadScore, leadScoreSummary })
```

## 5. Data model

Add two custom fields to the standard `person` object (and optionally
`opportunity`):

| Field name | Type | Notes |
|---|---|---|
| `leadScore` | NUMBER | 0-100, written by the agent |
| `leadScoreSummary` | TEXT | 1-2 sentence reasoning, written by the agent |
| `leadScoredAt` | DATE_TIME | timestamp of last scoring run, used to guard loops |

Use the Metadata API client (`metadataApiClient.createField`) or declare
these directly in a `person.object.ts` extension file — the SDK supports
extending standard objects with custom fields, not just creating new
objects.

## 6. Skill definition

```ts
// src/skills/lead-scoring.skill.ts
import { defineSkill } from 'twenty-sdk/define';

export default defineSkill({
  universalIdentifier: '<generate-a-uuid>',
  name: 'lead-scoring',
  label: 'Lead Scoring',
  description: 'Guides the agent through scoring a CRM lead',
  icon: 'IconBrain',
  content: `You are a B2B lead qualification assistant. Given a person or
opportunity record (name, job title, company, company size if known, deal
stage, and any notes), score how likely this lead is to convert, from 0
(no fit / no signal) to 100 (strong fit, clear buying intent). Consider:
seniority of job title, whether a company is attached, deal amount and
stage if present, and recency of last activity. Be conservative with
scores above 80 — reserve those for leads with explicit buying signals.`,
});
```

## 7. Agent definition

```ts
// src/agents/lead-scorer.agent.ts
import { defineAgent } from 'twenty-sdk/define';

export default defineAgent({
  universalIdentifier: '<generate-a-uuid>',
  name: 'lead-scorer',
  label: 'Lead Scorer',
  description: 'Scores CRM leads and explains its reasoning',
  icon: 'IconRobot',
  prompt: 'Score the lead and explain your reasoning in one to two sentences.',
  responseFormat: {
    type: 'json',
    schema: {
      type: 'object',
      properties: {
        score: { type: 'number', description: 'Lead score from 0 to 100' },
        summary: { type: 'string', description: 'Short reasoning for the score' },
      },
      required: ['score', 'summary'],
      additionalProperties: false,
    },
  },
});
```

Note the schema constraint from the docs: **properties must be flat
primitives** (string/number/boolean) — no nested objects or arrays in the
`responseFormat` schema.

## 8. Logic function (trigger + orchestration)

```ts
// src/functions/score-lead.function.ts
import { TwentyFunction, CoreApiClient } from 'twenty-sdk';
import { runAgent } from 'twenty-sdk/logic-function';

export const scoreLeadFunction = new TwentyFunction({
  name: 'scoreLead',
  description: 'Scores a person record using the lead-scorer agent',
  timeoutSeconds: 30, // agent runs can take several seconds — be generous
  triggers: [
    { type: 'databaseEvent', objectName: 'person', action: 'created' },
    {
      type: 'databaseEvent',
      objectName: 'person',
      action: 'updated',
      updatedFields: ['jobTitle', 'companyId'], // NOT leadScore/leadScoreSummary — avoids loops
    },
  ],
  handler: async (input, context) => {
    const { coreApiClient } = context;
    const person = await coreApiClient.findOne('person', input.data.recordId);

    const { result, success, error } = await runAgent({
      agentUniversalIdentifier: '<lead-scorer-uuid>',
      prompt: `Score this lead: ${JSON.stringify(person)}`,
    });

    if (!success) {
      console.error('Lead scoring failed:', error);
      return { success: false, error };
    }

    await coreApiClient.updateOne('person', person.id, {
      leadScore: result.score,
      leadScoreSummary: result.summary,
      leadScoredAt: new Date().toISOString(),
    });

    return { success: true, score: result.score };
  },
});
```

**Loop-avoidance (explicitly called out in Twenty's docs):** the
`updatedFields` scoping above only re-triggers on `jobTitle`/`companyId`
changes — never on the fields the agent itself writes
(`leadScore`, `leadScoreSummary`). This is the documented pattern for
avoiding `runAgent()` from a `*.updated` trigger looping on its own writes.

## 9. Roles & permissions

The app's default role **must** grant the AI permission flag or
`runAgent()` fails with a permission error:

```ts
// src/roles/default-role.ts
import { defineApplicationRole, SystemPermissionFlag } from 'twenty-sdk/define';

export default defineApplicationRole({
  universalIdentifier: '<generate-a-uuid>',
  label: 'Default function role',
  permissionFlagUniversalIdentifiers: [SystemPermissionFlag.AI],
});
```

Manifest permissions block should also declare read/write on `person`
(and `opportunity` if included).

## 10. UI (optional, phase 2)

A small `TwentyFrontComponent` badge rendered on the Person record page
showing the score (color-coded: red <40, yellow 40-70, green >70) and the
summary as a tooltip. Uses `useCoreApi().findOne('person', recordId)`.
Not required for v1 — the fields are visible in the standard record view
without it.

## 11. Build plan for Claude Code

1. `npx create-twenty-app lead-scoring-agent` → scaffold, `yarn twenty auth:login` against a dev/sandbox workspace.
2. Add custom fields to `person` (`leadScore`, `leadScoreSummary`, `leadScoredAt`) via metadata client or object extension.
3. Write the skill (`src/skills/lead-scoring.skill.ts`).
4. Write the agent (`src/agents/lead-scorer.agent.ts`) with the JSON schema.
5. Write the default role granting `SystemPermissionFlag.AI`.
6. Write `scoreLeadFunction` with the two triggers and loop guard.
7. `yarn twenty app:dev` — create a test person, confirm the function fires, confirm the agent returns valid JSON, confirm fields update.
8. Add unit test mocking `runAgent` per the pattern in Twenty's docs (`__tests__/score-lead.test.ts`).
9. (Phase 2) Add the front-component badge.
10. `yarn twenty app:sync` to deploy; `yarn twenty function:logs` to monitor in production.

## 12. Testing plan

- Unit test the function handler with a mocked `runAgent` response (see
  the `createPostCardFunction` test pattern in Twenty's docs for the
  shape).
- Manually create 3-4 test `person` records spanning obviously-cold to
  obviously-hot profiles and sanity-check score spread.
- Test the loop guard by updating `leadScore` directly via API and
  confirming the function does **not** re-fire.
- Test failure path: temporarily revoke the AI permission flag and confirm
  the function returns a clean error rather than crashing.

## 13. Risks / open questions

- **Alpha API**: `defineSkill`/`defineAgent`/`runAgent` signatures may
  change before GA — pin SDK version and re-test on upgrade.
- **AI credit consumption**: each `runAgent()` call consumes workspace AI
  credits; on a high-volume workspace (thousands of new leads/day) this
  could get expensive — consider batching or a volume threshold before
  wiring it to `created` events on a live production workspace.
- **No external demand signal**: unlike the other five ideas, this one
  isn't backed by a specific GitHub Discussion vote count — validate
  interest with a real prospect before investing beyond a portfolio demo.

## Monetization

Twenty has no built-in app marketplace payment system and no official
Stripe/billing partnership — there is no platform-native way to sell this
app or collect recurring revenue through Twenty itself. Any money changes
hands directly between you and the client (invoice, Stripe link, bank
transfer), outside the platform.

This is the **weakest standalone-monetization idea** of the six: it's a
small, fast build that clients will likely expect bundled into a larger
engagement rather than paid for as its own line item. Realistic models:

- **Bundled add-on**: fold it into a broader Twenty setup/customization
  contract as a value-add, not a separately priced deliverable.
- **Capability demo**: use it to win larger implementation work — it's a
  fast, impressive way to show AI-agent competency to a prospect before
  quoting a bigger project (e.g., the WhatsApp or migration builds).
- **Retainer, not the app itself**: if there's recurring revenue here, it's
  from monitoring AI-credit consumption and tuning the scoring prompt over
  time, not from the initial build.

## 14. References

- Skills & Agents docs: `docs.twenty.com/developers/extend/apps/logic/skills-and-agents`
- Custom Apps guide: `docs.twenty.com/developers/extend/apps` (objects, functions, front-components, manifest)
- `twenty-sdk` on npm
- Twenty GitHub repo tagline / `twenty-claude-skills`, `twenty-codex-plugin` packages: `github.com/twentyhq/twenty`
