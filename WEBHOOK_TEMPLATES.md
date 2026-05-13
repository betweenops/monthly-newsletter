# Webhook Templates

## Scope

These templates are for calling your LLM from Zapier using `Webhooks by Zapier`.

Use this document if:

- you want direct API control rather than the Zapier OpenAI app
- you want to call OpenAI-compatible APIs
- you want to use an Ask Sage-compatible endpoint with a similar request model

All examples assume:

- model: `gpt-5.4`
- classification is run per email
- draft generation is run once after aggregation

This file should own:

- request URLs
- headers
- example JSON bodies
- parser expectations for webhook responses

Workflow design and Zap sequencing live in [IMPLEMENTATION.md](./IMPLEMENTATION.md) and [ZAPIER_BUILD_SHEET.md](./ZAPIER_BUILD_SHEET.md).

## Webhooks by Zapier Setup

Use:

- App: `Webhooks by Zapier`
- Event: `Custom Request`

Recommended request method:

- `POST`

Recommended payload type:

- `json`

## OpenAI Classification Request

### URL

```text
https://api.openai.com/v1/responses
```

### Headers

```json
{
  "Authorization": "Bearer {{OPENAI_API_KEY}}",
  "Content-Type": "application/json"
}
```

Store the API key in Zapier connection fields or secrets if available. Do not hardcode it into a shared prompt field.

### Body

```json
{
  "model": "gpt-5.4",
  "temperature": 0.2,
  "text": {
    "format": {
      "type": "json_schema",
      "name": "newsletter_candidate_review",
      "strict": true,
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "message_id": {
            "type": "string"
          },
          "classification": {
            "type": "string",
            "enum": ["INCLUDE", "MAYBE", "EXCLUDE"]
          },
          "one_sentence_summary": {
            "type": "string"
          },
          "reason": {
            "type": "string"
          },
          "missing_detail_for_human_review": {
            "type": "string"
          },
          "extracted_facts": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "event_name": { "type": "string" },
              "organization": { "type": "string" },
              "date": { "type": "string" },
              "location": { "type": "string" },
              "call_to_action": { "type": "string" }
            },
            "required": ["event_name", "organization", "date", "location", "call_to_action"]
          }
        },
        "required": [
          "message_id",
          "classification",
          "one_sentence_summary",
          "reason",
          "missing_detail_for_human_review",
          "extracted_facts"
        ]
      }
    }
  },
  "input": [
    {
      "role": "system",
      "content": [
        {
          "type": "input_text",
          "text": "You are reviewing one candidate local update for a monthly community newsletter.\n\nRules:\nOnly mark INCLUDE if the item is:\n- local or regionally relevant\n- positive or useful\n- non-political\n- non-divisive\n- family-friendly\n- appropriate for a broad audience\n- supported by the source text\n\nMark EXCLUDE if it includes or suggests:\n- politics\n- crime\n- tragedy\n- emergencies\n- lawsuits\n- scandals\n- culture-war topics\n- controversial public issues\n- private or sensitive information\n- speculation\n- negative or fear-based framing\n\nIf the source is somewhat promising but unclear or incomplete, mark MAYBE.\n\nDo not invent facts.\nUse only the provided source content."
        }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "Candidate item:\n\nmessage_id: {{message_id}}\nfrom: {{from}}\nsubject: {{subject}}\ndate: {{date}}\nsnippet: {{snippet}}\nbody_excerpt: {{body_excerpt}}"
        }
      ]
    }
  ]
}
```

### Expected response parsing

In the OpenAI `responses` format, the structured output is typically available in the response body. In Zapier, inspect the live sample and extract the returned JSON text from the output field that contains the model result.

If your response comes back as text, parse it with `Formatter`.

## OpenAI Draft Generation Request

### URL

```text
https://api.openai.com/v1/responses
```

### Headers

```json
{
  "Authorization": "Bearer {{OPENAI_API_KEY}}",
  "Content-Type": "application/json"
}
```

### Body

```json
{
  "model": "gpt-5.4",
  "temperature": 0.4,
  "input": [
    {
      "role": "system",
      "content": [
        {
          "type": "input_text",
          "text": "You are writing a warm, practical, optimistic monthly local newsletter draft.\n\nUse only the APPROVED items provided below.\nDo not invent or infer missing facts.\nIf a detail is uncertain, omit it.\nDo not include political, divisive, negative, tragic, scandal-driven, or fear-based content.\n\nAudience:\n- broad local audience\n- family-friendly\n- community-oriented\n\nTone:\n- warm\n- practical\n- human\n- lightly upbeat\n- not salesy\n- not corporate\n- not generic\n\nOutput exactly in this structure:\n\nSUBJECT OPTIONS\n- option 1\n- option 2\n- option 3\n\nPREHEADER OPTIONS\n- option 1\n- option 2\n\nNEWSLETTER DRAFT\n[Write the full newsletter draft with:\n- short intro\n- Local Highlights section\n- Worth Knowing section if supported\n- short closing]\n\nINTERNAL REVIEW NOTES\n- Included items used\n- Maybe items omitted or needing review\n- Any missing facts or uncertainty"
        }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "Approved items:\n{{BUNDLED_INCLUDE_ITEMS_JSON}}\n\nMaybe items for reviewer context:\n{{BUNDLED_MAYBE_ITEMS_JSON}}\n\nOptional CRM audience/location context:\n{{CRM_CONTEXT}}"
        }
      ]
    }
  ]
}
```

### Expected output

This request can return plain text rather than strict JSON because the content is meant for direct email and Mailchimp draft creation.

## Ask Sage-Compatible Classification Request

Because Ask Sage deployments vary, use this as a generic OpenAI-compatible pattern and adjust the URL, auth header, and exact field names to match your environment.

### URL

```text
{{ASK_SAGE_OR_COMPATIBLE_BASE_URL}}
```

Example only:

```text
https://your-endpoint.example.com/v1/chat/completions
```

### Headers

```json
{
  "Authorization": "Bearer {{ASK_SAGE_API_KEY}}",
  "Content-Type": "application/json"
}
```

### Body

```json
{
  "model": "gpt-5.4",
  "temperature": 0.2,
  "response_format": {
    "type": "json_object"
  },
  "messages": [
    {
      "role": "system",
      "content": "You are reviewing one candidate local update for a monthly community newsletter.\n\nRules:\nOnly mark INCLUDE if the item is:\n- local or regionally relevant\n- positive or useful\n- non-political\n- non-divisive\n- family-friendly\n- appropriate for a broad audience\n- supported by the source text\n\nMark EXCLUDE if it includes or suggests:\n- politics\n- crime\n- tragedy\n- emergencies\n- lawsuits\n- scandals\n- culture-war topics\n- controversial public issues\n- private or sensitive information\n- speculation\n- negative or fear-based framing\n\nIf the source is somewhat promising but unclear or incomplete, mark MAYBE.\n\nDo not invent facts.\nUse only the provided source content.\n\nReturn strict JSON only in this exact shape:\n{\n  \"message_id\": \"string\",\n  \"classification\": \"INCLUDE | MAYBE | EXCLUDE\",\n  \"one_sentence_summary\": \"string\",\n  \"reason\": \"string\",\n  \"missing_detail_for_human_review\": \"string\",\n  \"extracted_facts\": {\n    \"event_name\": \"string\",\n    \"organization\": \"string\",\n    \"date\": \"string\",\n    \"location\": \"string\",\n    \"call_to_action\": \"string\"\n  }\n}"
    },
    {
      "role": "user",
      "content": "Candidate item:\n\nmessage_id: {{message_id}}\nfrom: {{from}}\nsubject: {{subject}}\ndate: {{date}}\nsnippet: {{snippet}}\nbody_excerpt: {{body_excerpt}}"
    }
  ]
}
```

### Parsing

For OpenAI-compatible chat completion APIs, extract:

- `choices[0].message.content`

Then parse JSON in Zapier.

## Ask Sage-Compatible Draft Request

### Body

```json
{
  "model": "gpt-5.4",
  "temperature": 0.4,
  "messages": [
    {
      "role": "system",
      "content": "You are writing a warm, practical, optimistic monthly local newsletter draft.\n\nUse only the APPROVED items provided below.\nDo not invent or infer missing facts.\nIf a detail is uncertain, omit it.\nDo not include political, divisive, negative, tragic, scandal-driven, or fear-based content.\n\nAudience:\n- broad local audience\n- family-friendly\n- community-oriented\n\nTone:\n- warm\n- practical\n- human\n- lightly upbeat\n- not salesy\n- not corporate\n- not generic\n\nOutput exactly in this structure:\n\nSUBJECT OPTIONS\n- option 1\n- option 2\n- option 3\n\nPREHEADER OPTIONS\n- option 1\n- option 2\n\nNEWSLETTER DRAFT\n[Write the full newsletter draft with:\n- short intro\n- Local Highlights section\n- Worth Knowing section if supported\n- short closing]\n\nINTERNAL REVIEW NOTES\n- Included items used\n- Maybe items omitted or needing review\n- Any missing facts or uncertainty"
    },
    {
      "role": "user",
      "content": "Approved items:\n{{BUNDLED_INCLUDE_ITEMS_JSON}}\n\nMaybe items for reviewer context:\n{{BUNDLED_MAYBE_ITEMS_JSON}}\n\nOptional CRM audience/location context:\n{{CRM_CONTEXT}}"
    }
  ]
}
```

## Recommended Zapier Parser Pattern

For classification calls:

1. Webhook step returns JSON or text
2. If text, add `Formatter` step to isolate the model content
3. Add `Formatter` or `Utilities` to parse JSON
4. Map parsed fields into storage

For draft generation:

1. Webhook returns long text
2. Store raw text directly
3. Optionally use Formatter to split out subject options and preheader options if your downstream app needs separate fields

## Splitting Subject, Preheader, and Draft

If the draft output is plain text with sections, use stable headings:

- `SUBJECT OPTIONS`
- `PREHEADER OPTIONS`
- `NEWSLETTER DRAFT`
- `INTERNAL REVIEW NOTES`

Then use Zapier `Text` formatter steps to split on those headings.

## Error Handling

### If the model returns invalid JSON

- Lower temperature to `0.1`
- Reinforce `strict JSON only`
- Strip markdown code fences before parsing
- Retry once with the same payload

### If the request times out

- Reduce body excerpt size
- Reduce candidate batch size
- Prefer one-email-per-call classification

### If the draft sounds too generic

- Provide fewer but stronger approved items
- Add location context from CRM
- Keep the tone guidance unchanged

## Security Notes

- Keep API keys in Zapier secrets or connection settings
- Do not email raw secrets or full request logs
- Keep classification payloads limited to the minimum text needed
