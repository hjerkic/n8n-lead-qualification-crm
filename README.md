# AI Lead Qualification & CRM Automation

An n8n workflow that takes an inbound lead, scores it against an explicit
qualification rubric using Claude, writes the result to HubSpot with a full audit
trail, and — for high-priority leads only — drafts a personalized outreach email in
Gmail for a human to review and send.

## The flow

```
Web form → Webhook (shared-secret auth required)
  → Validate required fields                     ── missing/invalid → 400
  → Claude: score the lead against the rubric
      (budget signal, urgency signal, fit signal → 0-100 + reasoning)
  → HubSpot: upsert contact + write an audit-trail note
      (score, signals, reasoning, and the lead's original message)
  → Is it high priority (score >= 70)?
      ├─ No  → done, contact logged in HubSpot for nurture/follow-up later
      └─ Yes → Claude drafts a personalized outreach email
                → Gmail: save as a draft (never auto-sent)
  → Respond to the caller with the outcome
```

Every external call (Claude ×2, HubSpot ×2, Gmail) retries transient failures up to
3 times before giving up, and every failure path returns a clean, sanitized error
response instead of a stack trace or a hung request.

## Why score with reasoning, not a binary flag

The lead is scored on three dimensions:

- **Budget signal** — explicit number/range mentioned, implied by scope, or none.
- **Urgency signal** — immediate, near-term, or purely exploratory.
- **Fit signal** — how well the request matches the target service (n8n / AI agent /
  CRM automation work), weighted most heavily: a great budget for the wrong kind of
  work is still a poor lead.

Claude is called with forced tool-calling so the output is always structured — a
score plus a `reasoning` string, never free text to parse. That reasoning is what
lands in HubSpot as the audit note, so anyone reviewing the contact record sees
*why* it was scored the way it was, not just a number. Score thresholds
(`>=70` high-priority, `40-69` nurture, `<40` log-only) are computed in code, not
left to the model, so the outcome is deterministic and reproducible for a given
score.

## Setup

1. **Import** `workflow.json` into n8n (Workflows → Import from File).
2. **Create credentials** for each of the following, then attach them to the
   matching nodes (n8n will show which nodes are missing a credential):

   | Credential | Type | Used by |
   |---|---|---|
   | Anthropic API key | Header Auth (`x-api-key`) | Score Lead - Claude, Draft Copy - Claude |
   | HubSpot access token | HubSpot App Token | HubSpot - Upsert Contact |
   | HubSpot access token | Bearer Auth | HubSpot - Create Audit Note |
   | Gmail OAuth2 (`gmail.compose` scope only) | Gmail OAuth2 | Gmail - Create Draft |
   | A random shared secret | Header Auth (`X-Webhook-Secret`) | Webhook - Lead Intake |

3. **Add custom contact properties** in HubSpot (Settings → Properties → Contact
   properties): `lead_qualification_score`, `lead_qualification_reasoning`,
   `lead_budget_signal`, `lead_urgency_signal`, `lead_fit_signal`,
   `lead_recommended_action`.
4. **Activate** the workflow. The webhook path is `/webhook/qualify-lead` (POST),
   and every request must include the `X-Webhook-Secret` header matching the
   credential from step 2.

## Example request

```bash
curl -X POST https://your-n8n-instance/webhook/qualify-lead \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Secret: your-shared-secret" \
  -d '{
    "name": "Jordan Lee",
    "email": "jordan@example.com",
    "company": "Acme Co",
    "message": "We need an n8n workflow to qualify inbound leads and push them to HubSpot, budget around $5k, would like to start this month."
  }'
```

```json
{
  "status": "success",
  "email": "jordan@example.com",
  "score": 92,
  "recommended_action": "high_priority_outreach",
  "contact_id": 123456789
}
```

## Security

- The webhook requires a shared-secret header — no request without the correct
  `X-Webhook-Secret` reaches any downstream node, so an exposed URL alone can't
  trigger API spend or CRM writes.
- No credentials, tokens, or secrets are stored in this repo. `workflow.json` is
  exported with all credential references stripped — every node needs its
  credential reattached after import.
- Gmail access is scoped to `gmail.compose` only (drafts), never `send` — outreach
  emails always go through a human before delivery.

## Stack

n8n · Claude API (forced tool-calling for structured output) · HubSpot CRM
(contacts + notes v3 API) · Gmail API (drafts only).
