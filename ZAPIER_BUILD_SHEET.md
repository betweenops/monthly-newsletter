# Zapier Build Sheet

## Scope

This document translates the MVP design into literal Zapier configuration guidance:

- app-by-app step sequence
- exact field mapping guidance
- recommended filters and storage keys
- Mailchimp draft setup
- HubSpot draft setup notes

Assumption: the build uses one main monthly Zap and keeps send approval manual.

This build sheet supports two Zapier-native paths:

- `template-only`: select the best Gmail items and place them into a fixed newsletter structure
- `AI-assisted`: use an AI step in Zapier to classify, summarize, or polish content before filling the draft

This file is intentionally operational. It should own:

- exact app sequence
- field mappings
- parser and filter steps
- Mailchimp and HubSpot draft setup

Architecture, guardrails, and design rationale live in [IMPLEMENTATION.md](./IMPLEMENTATION.md).

## Main Zap

Name suggestion:

```text
Monthly Newsletter Draft Builder
```

## Recommended Default

Start with the `template-only` path.

Add an AI step only if:

- the Gmail content is too messy to review manually
- you need help filtering down candidates
- you want cleaner draft copy than a direct template fill can provide

## Step 1: Schedule Trigger

App: `Schedule by Zapier`  
Event: `Every Month`

Recommended values:

- Day of Month: `1`
- Time of Day: `8:00 AM`
- Timezone: `America/New_York`

Output you will use later:

- `Zap Meta Human Now`
- `Zap Meta Timestamp`

## Step 2: Optional CRM Audience Lookup

Use this only if you want the draft email to include segment/location context.

App options:

- `HubSpot`
- your CRM app in Zapier

Recommended fields to pull:

- `newsletter_opt_in`
- `newsletter_status`
- `city`
- `region`
- `audience_segment`

Create a compact text field for later prompt context:

```text
Primary audience region: {{region}}
Primary city: {{city}}
Audience segment: {{audience_segment}}
```

If you do not need personalization yet, skip this step.

Minimum goal for this step:

- identify the audience/list/segment that should receive the draft

## Step 3: Gmail Find Emails

App: `Gmail`  
Event: `Find Emails`

Recommended configuration:

- Search String: `label:newsletter-source newer_than:45d`
- Max Results: `25`
- Include Spam and Trash: `No`

If you need broader coverage later, duplicate the search step with a second query:

```text
label:newsletter-source newer_than:45d ("community" OR "festival" OR "fundraiser" OR "volunteer" OR "grand opening" OR "family event")
```

### Fields to map forward

Map the Gmail outputs you actually see in your Zap account. Usually these are available:

- `ID` or `Message ID`
- `Thread ID`
- `From`
- `Subject`
- `Date`
- `Snippet`
- `Body Plain`
- `Internal Date`

## Step 4: Loop Through Found Emails

App: `Looping by Zapier`  
Event: `Create Loop From Line Items`

Line items to include:

- `message_id` -> Gmail `ID`
- `thread_id` -> Gmail `Thread ID`
- `from` -> Gmail `From`
- `subject` -> Gmail `Subject`
- `date` -> Gmail `Date`
- `snippet` -> Gmail `Snippet`
- `body_plain` -> Gmail `Body Plain`

Use simple line-item names because they are easier to map in later formatter, AI, and draft steps.

## Step 5: Clean the Email Body

App: `Formatter by Zapier`  
Event: `Text`

This may be one formatter step or a small chain of formatter steps depending on how noisy the emails are.

Recommended operations:

1. Trim whitespace
2. Replace repeated line breaks
3. Truncate to safe prompt length

Target output:

- `body_excerpt`

Recommended max length:

- `1500-2500` characters

If the Gmail body is often too messy, use the snippet plus the first `1500` characters of plain text rather than the entire body.

## Step 6: Prepare Candidate Input

App: `Formatter by Zapier`  
Event: `Text` or `Utilities`

Create a compact text blob you can use in either path:

- for a manual/template-first review flow
- for an optional AI classification step

Recommended candidate object:

```json
{
  "message_id": "{{message_id}}",
  "thread_id": "{{thread_id}}",
  "from": "{{from}}",
  "subject": "{{subject}}",
  "date": "{{date}}",
  "snippet": "{{snippet}}",
  "body_excerpt": "{{body_excerpt}}",
  "trusted_sender": "unknown",
  "label_match": true
}
```

If you are not using AI yet, this step can simply prepare a clean text summary for the digest or template-fill steps.

## Step 7: Optional AI Classification or Cleanup

Use one of these:

- `AI by Zapier`
- `OpenAI` app in Zapier
- `Webhooks by Zapier`
- another AI app already supported in your Zapier account

Purpose:

- classify one candidate as `INCLUDE`, `MAYBE`, or `EXCLUDE`
- or return a shorter cleaned summary for human review

Settings:

- Model: `gpt-5.4`
- Temperature: `0.2`

Input fields to include:

- `message_id`
- `from`
- `subject`
- `date`
- `snippet`
- `body_excerpt`

Expected response format:

```json
{
  "message_id": "string",
  "classification": "INCLUDE | MAYBE | EXCLUDE",
  "one_sentence_summary": "string",
  "reason": "string",
  "missing_detail_for_human_review": "string",
  "extracted_facts": {
    "event_name": "string",
    "organization": "string",
    "date": "string",
    "location": "string",
    "call_to_action": "string"
  }
}
```

Skip this step entirely if you stay on the template-only path and review candidates manually.

## Step 8: Parse AI Output

App: `Formatter by Zapier`  
Event: `Utilities`

Use this step to parse the JSON or structured text returned by the AI step.

Fields to extract:

- `message_id`
- `classification`
- `one_sentence_summary`
- `reason`
- `missing_detail_for_human_review`
- `extracted_facts.event_name`
- `extracted_facts.organization`
- `extracted_facts.date`
- `extracted_facts.location`
- `extracted_facts.call_to_action`

If the model occasionally returns wrapped text instead of clean JSON, add a formatter step first to strip code fences.

## Step 9: Store Reviewed or Selected Results

App options:

- `Digest by Zapier`
- `Storage by Zapier`

Recommended MVP default:

- use `Digest by Zapier`

Why:

- it is simpler to append one reviewed item per loop iteration
- it avoids the read-modify-write race you can create when repeatedly updating one shared storage key
- it keeps the first build focused on shipping a draft, not on building a mini data store inside Zapier

Use `Storage by Zapier` only if you have confirmed you need keyed retrieval outside the current run.

### Digest by Zapier pattern

Digest title:

```text
newsletter-reviewed-items
```

Append one compact line per reviewed email:

```text
{
  "message_id": "{{message_id}}",
  "classification": "{{classification}}",
  "summary": "{{one_sentence_summary}}",
  "reason": "{{reason}}",
  "missing_detail": "{{missing_detail_for_human_review}}",
  "event_name": "{{event_name}}",
  "organization": "{{organization}}",
  "date": "{{date}}",
  "location": "{{location}}",
  "call_to_action": "{{call_to_action}}",
  "source_from": "{{from}}",
  "source_subject": "{{subject}}",
  "source_snippet": "{{snippet}}"
}
```

Important:

- configure the digest to append a newline between entries
- keep each entry as one JSON object per line
- this makes the released digest easy to pass to the drafting prompt as an NDJSON-style bundle

### Storage by Zapier pattern

Use a monthly key:

```text
newsletter:{{zap_meta_human_now_month}}
```

If you need a safer explicit key:

```text
newsletter:2026-05
```

Value to append:

```json
{
  "message_id": "{{message_id}}",
  "classification": "{{classification}}",
  "summary": "{{one_sentence_summary}}",
  "reason": "{{reason}}",
  "missing_detail": "{{missing_detail_for_human_review}}",
  "event_name": "{{event_name}}",
  "organization": "{{organization}}",
  "date": "{{date}}",
  "location": "{{location}}",
  "call_to_action": "{{call_to_action}}",
  "source_from": "{{from}}",
  "source_subject": "{{subject}}",
  "source_snippet": "{{snippet}}",
  "source_excerpt": "{{body_excerpt}}"
}
```

## Step 10: Retrieve the Monthly Bundle

After the loop completes, retrieve the stored bundle for final drafting.

App:

- `Digest by Zapier` release digest
- or `Storage by Zapier` get value

You want three logical groups:

- `INCLUDE` items
- `MAYBE` items
- `EXCLUDE` items

Recommended MVP path:

- release the digest as one text block
- pass the full reviewed bundle to the second LLM
- instruct the model to use only `INCLUDE` items in the draft and to list `MAYBE` and `EXCLUDE` items only in internal review notes

This avoids a brittle parser chain in the first iteration.

## Step 11: Fill Newsletter Draft

App:

- `Formatter by Zapier`
- or an AI app in Zapier if you want writing help

Inputs:

- reviewed bundle
- optional CRM context
- current month if you want the intro to reference it

Expected output:

- `subject options`
- `preheader options`
- `newsletter draft`
- `internal review notes`

Template-only path:

- map the best approved items into the HTML placeholders in `templates/mailchimp-newsletter.html`
- write a fixed intro and closing that can be reused each month

AI-assisted path:

- ask the AI step to turn the approved bundle into draft-ready sections
- keep the output structure stable so it still maps cleanly into Mailchimp or HubSpot

Recommended parser path for the AI-assisted version:

1. keep the draft-generation response as plain text
2. split it by the stable headings in [WEBHOOK_TEMPLATES.md](./WEBHOOK_TEMPLATES.md)
3. map only the first subject option and first preheader option into Mailchimp fields
4. keep the full raw draft text in the internal review email

## Step 12: Create Mailchimp Draft

App: `Mailchimp`  
Event: `Create Campaign`

Recommended values:

- Type: `Regular`
- Audience/List: your newsletter list
- Subject Line: first subject option from the LLM
- From Name: your business or personal brand
- Reply-To: your inbox

Save these outputs:

- `Campaign ID`
- `Campaign Web ID`
- `Archive URL` if provided

## Step 13: Set Mailchimp Campaign Content

App: `Mailchimp`  
Event: `Set Campaign Content`

For the MVP, use a simple content block rather than a complex HTML builder.

Recommended content layout:

```html
<h2>Monthly Local Highlights</h2>
<p>{{intro paragraph}}</p>
<h3>Local Highlights</h3>
<p>{{section body}}</p>
<h3>Worth Knowing</h3>
<p>{{section body}}</p>
<p>{{closing}}</p>
```

Recommended first iteration:

- start from [templates/mailchimp-newsletter.html](./templates/mailchimp-newsletter.html)
- replace `{{INTRO_HEADING}}`, `{{INTRO_PARAGRAPH}}`, `{{LOCAL_HIGHLIGHTS_BODY}}`, `{{WORTH_KNOWING_BODY}}`, and `{{CLOSING_PARAGRAPH}}` with Formatter outputs or hand-mapped text sections

If your Mailchimp action expects one full HTML field:

- use the starter template as the wrapper
- map the generated sections into the placeholders

If section splitting is too brittle in your Zap account:

- store the draft as plain text in the review email
- paste the approved content into Mailchimp manually for the first live run

For the first live run, reliability is more important than automation depth.

## Step 14: Send Internal Review Email

App: `Gmail`  
Event: `Send Email`

To:

- your review email

Subject:

```text
Monthly newsletter draft ready for review
```

Email body template:

```text
Campaign ID: {{Campaign ID}}

Subject line options:
{{Subject Options}}

Preheader options:
{{Preheader Options}}

Newsletter draft:
{{Newsletter Draft}}

Internal review notes:
{{Internal Review Notes}}

Approval checklist:
- Confirm all included items are local and broad-audience safe
- Confirm dates, locations, and organization names are correct
- Remove any MAYBE item unless source support is strong enough
- Confirm no politics, tragedy, controversy, or sensitive details appear
- Confirm the campaign is still in draft state
```

## Step 15: Manual Approval and Send

Do not automate this.

Human action:

1. Review the internal email
2. Open the draft in Mailchimp
3. Verify facts against source summaries
4. Remove anything ambiguous
5. Send manually

## HubSpot Swap

If you prefer HubSpot over Mailchimp, keep steps `1-11` the same and replace steps `12-13`.

### HubSpot draft pattern

Use HubSpot to:

- create a marketing email draft
- populate subject and body
- leave the email unpublished or unsent

Suggested mappings:

- Subject -> first generated subject option
- Preview text -> first generated preheader option
- Body -> generated newsletter draft or simple HTML version

Keep review in Gmail exactly the same.

## Recommended Filters

If you want extra protection before storing items:

### Filter A

Continue only if Gmail subject or snippet is not empty.

### Filter B

Continue only if `body_excerpt` has at least `100` characters.

### Filter C

Optional hard block if the subject contains obvious excluded terms:

- `election`
- `crime`
- `shooting`
- `lawsuit`
- `protest`
- `emergency`

This is optional because the model already handles it, but it can reduce noise.

## Operational Notes

- Keep candidate count low at first.
- Do not try to perfectly normalize email HTML in Zapier.
- Prefer simple text extraction and aggressive trimming.
- One AI classification call per email is safer than one giant batch.
- Save all classifications if possible so you can audit the monthly run later.
- Dry-run the webhook and parser chain with the sample files in `examples/` before using live Gmail data.

## Recommended MVP Defaults

- Gmail search query: `label:newsletter-source newer_than:45d`
- Max Gmail results: `25`
- Classification temperature: `0.2`
- Drafting temperature: `0.4`
- Destination platform: `Mailchimp`
- Send behavior: manual only
