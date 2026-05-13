# Zapier Build Sheet

## Scope

This document translates the MVP design into literal Zapier configuration guidance:

- app-by-app step sequence
- exact field mapping guidance
- recommended filters and storage keys
- Mailchimp draft setup
- HubSpot draft setup notes

Assumption: the build uses one main monthly Zap and keeps send approval manual.

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
- `Webhook` if your CRM requires API access

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

Use simple line-item names because they are easier to map in later webhook steps.

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

## Step 6: Prepare Candidate JSON

App: `Formatter by Zapier`  
Event: `Text` or `Utilities`

Create a single text blob or JSON string to send to the LLM.

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

If Zapier makes it awkward to construct JSON with quotes, plain text input is fine as long as the prompt explicitly asks for strict JSON output.

## Step 7: LLM Classification Call

Use one of these:

- `OpenAI` app in Zapier
- `Webhooks by Zapier`
- Ask Sage-compatible HTTP call via `Webhooks by Zapier`

Purpose:

- classify one candidate as `INCLUDE`, `MAYBE`, or `EXCLUDE`

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

## Step 8: Parse Model Output

App: `Formatter by Zapier`  
Event: `Utilities`

Use this step to parse the JSON returned by the model.

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

## Step 9: Store Reviewed Results

App options:

- `Storage by Zapier`
- `Digest by Zapier`

Recommended choice:

- `Storage by Zapier` if you want structured retrieval
- `Digest by Zapier` if you want a simpler MVP and can work with a text bundle

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

### Digest by Zapier pattern

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
  "source_subject": "{{subject}}"
}
```

## Step 10: Retrieve the Monthly Bundle

After the loop completes, retrieve the stored bundle for final drafting.

App:

- `Storage by Zapier` get value
- or `Digest by Zapier` release digest

You want three logical groups:

- `INCLUDE` items
- `MAYBE` items
- `EXCLUDE` items

If Zapier makes grouping difficult, send the full reviewed bundle to the second LLM and instruct it to use only `INCLUDE` items in the draft while reporting `MAYBE` and `EXCLUDE` in internal notes.

## Step 11: Generate Newsletter Draft

App:

- `OpenAI`
- or `Webhooks by Zapier`

Settings:

- Model: `gpt-5.4`
- Temperature: `0.4`

Inputs:

- reviewed bundle
- optional CRM context
- current month if you want the intro to reference it

Expected output:

- `subject options`
- `preheader options`
- `newsletter draft`
- `internal review notes`

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

If your Mailchimp action expects one full HTML field, either:

- build a very simple HTML wrapper in Formatter
- or store the LLM output as plain text and paste it into a simple template later

For the MVP, simplicity is more important than polished HTML.

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
- One classification call per email is safer than one giant batch.
- Save all classifications if possible so you can audit the monthly run later.

## Recommended MVP Defaults

- Gmail search query: `label:newsletter-source newer_than:45d`
- Max Gmail results: `25`
- Classification temperature: `0.2`
- Drafting temperature: `0.4`
- Destination platform: `Mailchimp`
- Send behavior: manual only
