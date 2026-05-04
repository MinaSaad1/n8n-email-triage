# Architecture

## High-level

```
Gmail Trigger (poll unread, 5 min)
        │
        ▼
Extract Email Fields ─── Set node, normalize email shape
        │
        ▼
Triage + Draft Reply ─── LangChain LLM chain
        │                  Claude Haiku 4.5, single JSON response
        ▼
Parse Triage Response ─── Code node, JSON.parse + merge
        │
        ├────────────────────────────┐
        ▼                            ▼
Create Gmail Draft        Is Urgent? (IF node)
                                     │ true
                                     ▼
                          Slack Urgent Alert
```

The flow is intentionally narrow: one LLM call per email, one draft per email, one optional Slack ping for the high-priority subset. Routing complexity (per-category Slack channels, Airtable logging, auto-archive) is deferred to the user as opt-in branches rather than baked into the template.

## Components

### Gmail Poll Unread (`gmailTrigger`)

Polls every 5 minutes for inbox messages with `readStatus: unread` and `includeSpamTrash: false`. The 5-minute cadence is the practical floor for Gmail polling, anything tighter risks rate limits and produces near-zero latency benefit because Gmail's own delivery isn't instant either.

The trigger does not mark emails as read or apply a label by itself. That means the same unread email will appear on every poll until it's read or labeled. For low-volume inboxes that's fine, the workflow re-creates the draft idempotently. For high-volume inboxes you'll want to add an `addLabel` step after `Create Gmail Draft` that applies `n8n-triaged`, then update the trigger filter to exclude that label.

### Extract Email Fields (`Set` node)

Normalizes the Gmail trigger's payload into a flat shape:

```
{ email_id, from, subject, body, date }
```

`body` falls back through `text -> snippet -> ''` so the prompt always has something to chew on. The id is preserved so a later "mark as triaged" step has the message reference it needs.

### Triage + Draft Reply (`chainLlm` + `lmChatAnthropic`)

A LangChain LLM chain with Claude Haiku 4.5 (`claude-haiku-4-5-20251001`) wired in as the model. The prompt asks for a single JSON object containing:

- `category` (one of nine fixed values)
- `priority` (high / medium / low)
- `one_line_summary` (max 15 words)
- `draft_reply` (2 to 4 sentences, first person, with `[PLACEHOLDER]` markers where personalization is needed)
- `label` (Gmail label suggestion, mirrors category by default)

`maxTokens` is capped at 800. That's enough headroom for a short draft and the surrounding metadata, but small enough to keep cost predictable on inbox spikes.

### Parse Triage Response (`Code` node)

Strips markdown code fences if Claude wraps the JSON in ```` ```json ```` (it sometimes does despite the prompt telling it not to), parses the result, and merges it with the original email fields. On parse failure, returns a safe default object so the rest of the flow doesn't crash:

```js
{ category: 'other', priority: 'medium', one_line_summary: 'Parse error', draft_reply: '', label: 'other' }
```

This is deliberate. A bad model response shouldn't take down the whole workflow.

### Create Gmail Draft (`gmail` node, createDraft)

Writes a draft addressed to `{{ $json.from }}` with subject `Re: {original subject}` and the model's `draft_reply` as the body. Drafts land in the user's Drafts folder, ready for review. Nothing is sent automatically, ever.

### Is Urgent? (`if` node)

Single condition: `priority == 'high'` (case-insensitive). True branch fires Slack, false branch is empty.

### Slack Urgent Alert (`slack` node, post)

Posts a structured message to the configured channel with sender, subject, one-line summary, and category. The channel id is hardcoded as `YOUR_SLACK_CHANNEL_ID` in the template, users replace it during setup.

## Design decisions worth calling out

### Why polling instead of Gmail push

Gmail Pub/Sub push is faster but requires a Google Cloud project, a Pub/Sub topic, and a verified webhook endpoint. For a template that should "just import and run," that setup cost outweighs the latency win. Five-minute polling adds at most 5 minutes of delay to triage, which is irrelevant for everything except `priority: high` (and even then, you're getting an alert you'd otherwise have missed entirely).

If your specific use case needs sub-minute latency, swap the Gmail Trigger for a Webhook node and wire Pub/Sub to it. The rest of the workflow doesn't change.

### Why categorize and draft in a single LLM call

Two separate calls (one for classification, one for drafting) would give better isolation but doubles cost and adds a node. Haiku 4.5 is strong enough to do both in one pass with a structured JSON output. The Code node parses and splits the result downstream.

If you swap to a weaker model later, separate the calls. Until then, one call is the right tradeoff.

### Why no Airtable log in the default template

The original use-case writeup mentioned an Airtable triage board. Building it in forces every user onto Airtable, and the workflow is just as useful without it. The README's Configuration section shows the one-node addition for users who want it.

### Why the Slack ping is only on `priority: high`

Slack is for interrupts. Drafting every email into Gmail is the bulk path, that's where you read in batch. Slack should fire only when something genuinely needs eyes inside the next hour. Routing every category to its own Slack channel is a power-user pattern, not a default.

### Why the workflow doesn't mark emails as read

Marking-as-read on draft creation seems harmless until the workflow errors after the mark and before the draft. Now you've got an email Gmail thinks is read but you've never seen. Default behavior leaves the read state alone, the user opens the draft in Gmail and the parent message gets read alongside it.

For high-volume inboxes where re-triggering on the same email is wasteful, see the Gmail label pattern in the README's Configuration section.

## Performance notes

| Step | Latency expectation |
|---|---|
| Gmail poll | <500 ms per cycle |
| Extract Email Fields | <50 ms |
| Triage + Draft Reply (Haiku 4.5, ~800 tokens) | 1 to 3 sec per email |
| Parse Triage Response | <50 ms |
| Create Gmail Draft | 200 to 800 ms |
| Slack Urgent Alert | 200 to 500 ms |

Total per email: roughly 2 to 5 seconds. A poll that picks up 10 unread emails takes 20 to 50 seconds end-to-end. If your inbox routinely gets 50+ unread per 5-minute window, lengthen the poll interval and add a label-based exclusion filter so you don't re-process the backlog every cycle.

## Observability

- **n8n Executions panel** is the primary debugging surface. Filter by failed executions to find prompt-parse fallbacks or API timeouts.
- The **sticky note inside the workflow** carries setup notes and the mark-as-read caveat. Edit it as you customize.
- For accuracy review, the Code node's output (`category`, `priority`, `one_line_summary`) is exactly what you'd want logged. A 5-second add-on to write each row to a Google Sheet or Airtable gives you a permanent classification record without changing the rest of the flow.

## See also

- [SECURITY.md](SECURITY.md): threat model, prompt injection, scope minimization
- [Catalog architecture principles](https://github.com/MinaSaad1/n8n-ai-agents/blob/main/docs/architecture-principles.md): patterns shared across every template in the collection
