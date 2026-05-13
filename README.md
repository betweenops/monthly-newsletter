# Monthly Newsletter Automation

Zapier-based MVP for generating a monthly local community newsletter draft from Gmail and attaching it to a CRM-defined audience, with human approval required before anything is sent.

## Overview

This project is scoped as a simple Zapier automation, not a custom backend.

The workflow:

1. Runs once per month in Zapier
2. Searches Gmail for recent local/community updates
3. Optionally uses an AI step in Zapier to classify each candidate as `INCLUDE`, `MAYBE`, or `EXCLUDE`
4. Fills a newsletter draft template from approved items
5. Creates a draft in Mailchimp or HubSpot
6. Uses the CRM-defined audience/list for recipients
7. Emails the owner a review summary
8. Requires manual approval before send

## Goal

Create a warm, practical, community-focused monthly newsletter for CRM contacts using recent inbox content while excluding anything political, divisive, negative, sensitive, or speculative.

## MVP Stack

- `Zapier` for scheduling and orchestration
- `Gmail` for candidate content
- `CRM` for audience/source-of-truth data
- `Mailchimp` or `HubSpot` for draft creation and recipient audience
- optional `AI by Zapier`, `OpenAI`, or similar Zapier AI step for classification/draft help
- `Gmail` for internal review notification

## Content Rules

Include:

- local community events
- local business openings or milestones
- nonprofit and volunteer opportunities
- seasonal reminders
- positive school/community achievements
- family-friendly local resources and activities

Exclude:

- politics
- crime
- tragedy
- emergencies
- scandals
- lawsuits
- culture-war topics
- gossip
- private or sensitive information
- speculative or fear-based content

## Key Decisions

- Keep this as a Zapier-first MVP
- Prefer a curated Gmail label such as `newsletter-source`
- Keep the first version buildable without direct API calls
- Use AI inside Zapier only if the newsletter content needs help with classification or drafting
- Generate a draft only, never auto-send
- Keep a human review step in every run

## Project Docs

- [Implementation Spec](./IMPLEMENTATION.md) for architecture, design decisions, guardrails, data model, and operating guidance
- [Zapier Build Sheet](./ZAPIER_BUILD_SHEET.md) for the exact Zap sequence, app steps, mappings, filters, and draft-platform setup
- [Webhook Templates](./WEBHOOK_TEMPLATES.md) for optional advanced webhook/API usage if Zapier app steps are not enough

## Recommended Build Order

1. Create the Gmail label and trusted sender list
2. Build the monthly Zap trigger and Gmail search
3. Pull the audience/list from the CRM or email platform
4. Fill the draft template from the selected Gmail items
5. Add an AI step only if needed for cleanup, classification, or writing help
6. Create the Mailchimp or HubSpot draft
7. Send the internal review email
8. Run a manual dry run and tune the filters

## Current Status

This repo is now organized as a small handoff package:

- `README.md` is the entry point
- `IMPLEMENTATION.md` explains what to build and why
- `ZAPIER_BUILD_SHEET.md` explains how to build it in Zapier
- `WEBHOOK_TEMPLATES.md` contains optional LLM request bodies for an advanced webhook path
- `examples/` contains sample candidate, review, and bundle payloads for dry runs
- `templates/` contains a starter HTML template for Mailchimp campaign content

## Next Step

Use [ZAPIER_BUILD_SHEET.md](./ZAPIER_BUILD_SHEET.md) for the default Zapier-native build. Only use [WEBHOOK_TEMPLATES.md](./WEBHOOK_TEMPLATES.md) if you decide later that Zapier's built-in app steps are too limited.
