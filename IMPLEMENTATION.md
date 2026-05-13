# Monthly Newsletter Automation MVP

## Goal

Build a simple monthly workflow that:

- runs once per month in Zapier
- pulls candidate local updates from Gmail
- uses an LLM to classify items as `INCLUDE`, `MAYBE`, or `EXCLUDE`
- generates a newsletter draft from approved items only
- creates a draft in Mailchimp or HubSpot
- emails the owner a review summary
- never sends automatically

This document is intentionally scoped to a Zapier-first MVP. It does not require AWS, a custom backend, or a database.

## Core Decisions

- Use `Zapier` as the scheduler, glue, and approval-notification layer.
- Use `Gmail` as the source of candidate updates.
- Use a `trusted sender + Gmail label` strategy for quality control.
- Use `gpt-5.4` via OpenAI or an Ask Sage-compatible API for classification and draft generation.
- Use `Mailchimp` first for the MVP because draft campaign creation is straightforward.
- Keep `HubSpot` as an equivalent downstream swap if preferred.
- Require human review before send.

## Architecture

1. `Schedule by Zapier` triggers monthly.
2. `Gmail` searches for candidate messages from the last `30-45` days.
3. `Looping by Zapier` processes each candidate email.
4. `Formatter by Zapier` extracts a compact structured candidate record.
5. `LLM Step 1` classifies and summarizes each candidate.
6. `Storage by Zapier` or `Digest by Zapier` collects the reviewed results.
7. `LLM Step 2` writes the final draft using approved items only.
8. `Mailchimp` or `HubSpot` creates a draft newsletter.
9. `Gmail` emails the owner a review package and approval checklist.
10. The owner manually reviews and sends from the email platform.

## Recommended Data Model

### Gmail Source Strategy

Create a Gmail label named `newsletter-source`.

Apply that label to:

- local chamber emails
- downtown association emails
- library updates
- parks and recreation emails
- school/community newsletters
- nonprofit newsletters
- trusted local business announcements
- family-friendly event roundups

This matters more than prompt tuning. The highest quality MVP is produced by curating the inbox before the LLM sees it.

### CRM Fields

Minimum fields:

- `newsletter_opt_in` boolean
- `newsletter_status` enum: `active`, `paused`, `unsubscribed`
- `city`
- `region`
- `audience_segment`

Useful later:

- `family_focus` boolean
- `topics_of_interest` multi-select
- `preferred_email`
- `last_newsletter_sent_at`

### Candidate Item Shape

Use this object per email before classification:

```json
{
  "message_id": "gmail-message-id",
  "thread_id": "gmail-thread-id",
  "from": "sender@example.org",
  "subject": "Community Spring Festival",
  "date": "2026-04-21T13:05:00Z",
  "snippet": "Join us this Saturday for...",
  "body_excerpt": "Trimmed plain-text email body...",
  "trusted_sender": true,
  "label_match": true
}
```

### Reviewed Item Shape

Use this object after LLM review:

```json
{
  "message_id": "gmail-message-id",
  "classification": "INCLUDE",
  "one_sentence_summary": "The city library is hosting a free family reading event next Saturday.",
  "reason": "This is local, positive, broadly useful, family-friendly, and clearly supported by the source text.",
  "missing_detail_for_human_review": "",
  "extracted_facts": {
    "event_name": "Spring Reading Day",
    "organization": "City Library",
    "date": "Saturday, May 10",
    "location": "Main Branch Library",
    "call_to_action": "Register online"
  },
  "source_from": "library@example.org",
  "source_subject": "Spring Reading Day",
  "source_snippet": "Free books, family activities..."
}
```

## Gmail Search Strategy

Run one or more of these searches in Zapier.

Primary query:

```text
label:newsletter-source newer_than:45d
```

Expanded community query:

```text
label:newsletter-source newer_than:45d ("community" OR "festival" OR "fundraiser" OR "volunteer" OR "grand opening" OR "family event" OR "open house" OR "farmers market")
```

Trusted sender query:

```text
newer_than:45d from:(chamber* OR downtown* OR library* OR parks* OR schools* OR foundation* OR nonprofit*)
```

Useful local reminders query:

```text
newer_than:30d ("summer camp" OR "cleanup" OR "food drive" OR "donation drive" OR "community class" OR "seasonal reminder")
```

Negative topic exclusion query:

```text
newer_than:45d -("election" OR "shooting" OR "arrest" OR "lawsuit" OR "protest" OR "emergency" OR "breaking news" OR "crash")
```

### Recommended Search Order

1. Use the label-only query first.
2. Add the label-plus-keywords query next.
3. Add the trusted-sender query only if the first two are too sparse.
4. Add broad keyword queries last, because they produce more noise.

### Volume Guidance

Start with `20-40` emails per monthly run. More than that increases false positives, token costs, and review friction.

## Workflow Summary

The monthly run should stay simple:

1. Trigger the workflow on a fixed monthly schedule.
2. Search Gmail for labeled and trusted-source messages from the last `30-45` days.
3. Clean each message into a compact candidate record.
4. Classify each candidate with the LLM as `INCLUDE`, `MAYBE`, or `EXCLUDE`.
5. Aggregate the reviewed results for that month.
6. Generate a newsletter draft from approved items only.
7. Create a draft in Mailchimp or HubSpot.
8. Email the owner a review summary and checklist.
9. Require manual approval before send.

For the literal Zap steps and field mappings, use [ZAPIER_BUILD_SHEET.md](./ZAPIER_BUILD_SHEET.md).  
For the exact OpenAI and Ask Sage-compatible request payloads, use [WEBHOOK_TEMPLATES.md](./WEBHOOK_TEMPLATES.md).

## Mailchimp vs HubSpot

### Mailchimp

Use Mailchimp if you want:

- quicker draft creation
- a lighter-weight campaign builder
- the fastest MVP setup

### HubSpot

Use HubSpot if you want:

- tighter CRM alignment
- marketing emails tied more directly to contact properties
- more centralized sales and marketing workflows

Both work for the MVP. Mailchimp is the lower-friction default.

## CRM Integration Pattern

Keep CRM use minimal initially.

Use CRM to:

- define the send audience
- identify opted-in contacts
- segment by city or region if needed
- store newsletter participation status

Do not use CRM as the source of content in MVP. Gmail should remain the source of candidate updates.

## Prompt Ownership

Keep prompt definitions in one place:

- classification and draft-generation request bodies live in [WEBHOOK_TEMPLATES.md](./WEBHOOK_TEMPLATES.md)
- Zap-level field mappings and parser steps live in [ZAPIER_BUILD_SHEET.md](./ZAPIER_BUILD_SHEET.md)

This document should describe the rules and decisions behind those prompts, not duplicate them.

## Guardrails

### Content Guardrails

Only include:

- local community events
- local business openings or milestones
- nonprofit or volunteer opportunities
- seasonal reminders
- school/community achievements if clearly positive and non-political
- local resources
- family-friendly activities
- useful neighborhood updates

Always exclude:

- politics
- crime
- tragedy
- emergencies
- scandals
- lawsuits
- culture-war topics
- controversial activism
- divisive public meetings
- gossip
- private customer information
- speculative claims
- negative or fear-based content

### LLM Guardrails

- Keep temperature low.
- Require strict JSON for the classifier.
- Process one email per classification call.
- Use source-grounding language in every prompt.
- If any key fact is uncertain, mark `MAYBE` or omit.
- Never allow the model to invent dates, venues, or organizations.

### Operational Guardrails

- Never enable auto-send in the MVP.
- Keep a human review email in every run.
- Keep source summaries and classification reasons.
- Limit monthly candidate count.
- Prefer sender allowlists and labels over open-ended keyword ingestion.

## Human Approval Checklist

- Is every included item clearly local or regionally relevant?
- Is every included item positive, practical, or community-useful?
- Did anything political, divisive, tragic, scandal-driven, or fear-based slip through?
- Are the dates, locations, and organization names correct?
- Is every claim supported by the source email?
- Are any `MAYBE` items still too ambiguous to include?
- Does the tone feel warm and practical rather than generic or promotional?
- Is the campaign still in draft state?

## Failure Modes and Mitigations

### Too Many Irrelevant Emails

Mitigation:

- tighten the Gmail label
- add trusted senders
- reduce max results
- shorten the date window to `30d`

### Good Items Missing Important Details

Mitigation:

- mark as `MAYBE`
- include the source link in the review email
- require human confirmation before final send

### Draft Sounds Generic

Mitigation:

- keep the draft prompt strict
- provide only strong approved items
- avoid padding if only a few items qualify
- include light regional context from CRM

### LLM Costs Grow

Mitigation:

- classify one email at a time with trimmed excerpts
- cap monthly volume
- rely more on Gmail labels than broad search queries

## Recommended MVP Defaults

- Frequency: monthly
- Date window: `45d`
- Gmail max candidates: `25`
- Classification model: `gpt-5.4`
- Classification temperature: `0.2`
- Drafting temperature: `0.4`
- Draft destination: `Mailchimp`
- Send mode: manual only

## Documentation Map

- [README.md](./README.md): quick project overview and starting point
- [IMPLEMENTATION.md](./IMPLEMENTATION.md): architecture, design decisions, guardrails, data model, and operating guidance
- [ZAPIER_BUILD_SHEET.md](./ZAPIER_BUILD_SHEET.md): exact Zapier step sequence, mappings, filters, and draft-platform setup
- [WEBHOOK_TEMPLATES.md](./WEBHOOK_TEMPLATES.md): copy/paste webhook requests for OpenAI and Ask Sage-compatible endpoints

## Future Improvements

After the MVP works reliably, consider:

- sender allowlists stored in Airtable or CRM
- duplicate detection across months
- topic scoring by audience segment
- richer HTML templates
- Airtable or Notion approval queue
- location-specific newsletter versions
- performance feedback loop based on opens/clicks
- stronger source extraction for event dates and links

## Recommended Build Order

1. Create the Gmail label and trusted sender list.
2. Build the monthly trigger and Gmail search.
3. Build per-email classification and verify JSON output.
4. Store reviewed items in Zapier storage or digest.
5. Build the draft-generation step using only approved items.
6. Create the Mailchimp draft.
7. Send the review summary email.
8. Run a manual dry run with real inbox content.
9. Tune labels, queries, and prompts based on false positives.

## Final Recommendation

For the first version:

- use `Mailchimp`
- use `newsletter-source` Gmail label
- process `25` recent emails monthly
- classify one message at a time
- generate one draft newsletter
- review manually every month before send

That gives you the simplest system with the lowest implementation risk and the best chance of producing useful output quickly.
