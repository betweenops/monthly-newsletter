# Monthly Newsletter Automation

Zapier-based MVP for generating a monthly local community newsletter draft from Gmail using an LLM, with human approval required before anything is sent.

## Overview

This project is scoped as a simple automation, not a custom backend.

The workflow:

1. Runs once per month in Zapier
2. Searches Gmail for recent local/community updates
3. Uses an LLM to classify each candidate as `INCLUDE`, `MAYBE`, or `EXCLUDE`
4. Generates a newsletter draft from approved items only
5. Creates a draft in Mailchimp or HubSpot
6. Emails the owner a review summary
7. Requires manual approval before send

## Goal

Create a warm, practical, community-focused monthly newsletter for CRM contacts using recent inbox content while excluding anything political, divisive, negative, sensitive, or speculative.

## MVP Stack

- `Zapier` for scheduling and orchestration
- `Gmail` for candidate content
- `CRM` for audience/source-of-truth data
- `OpenAI` or Ask Sage-compatible API using `gpt-5.4`
- `Mailchimp` or `HubSpot` for draft creation
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
- Classify one email at a time for stability and auditability
- Generate a draft only, never auto-send
- Keep a human review step in every run

## Project Docs

- [Implementation Spec](./IMPLEMENTATION.md) for architecture, design decisions, guardrails, data model, and operating guidance
- [Zapier Build Sheet](./ZAPIER_BUILD_SHEET.md) for the exact Zap sequence, app steps, mappings, filters, and draft-platform setup
- [Webhook Templates](./WEBHOOK_TEMPLATES.md) for copy/paste OpenAI and Ask Sage-compatible request payloads

## Recommended Build Order

1. Create the Gmail label and trusted sender list
2. Build the monthly Zap trigger and Gmail search
3. Add per-email LLM classification
4. Aggregate reviewed items
5. Generate the final newsletter draft
6. Create the Mailchimp or HubSpot draft
7. Send the internal review email
8. Run a manual dry run and tune the filters

## Current Status

This repo is now organized as a small handoff package:

- `README.md` is the entry point
- `IMPLEMENTATION.md` explains what to build and why
- `ZAPIER_BUILD_SHEET.md` explains how to build it in Zapier
- `WEBHOOK_TEMPLATES.md` contains the LLM request bodies

## Next Step

Use [ZAPIER_BUILD_SHEET.md](./ZAPIER_BUILD_SHEET.md) to build the Zap, and use [WEBHOOK_TEMPLATES.md](./WEBHOOK_TEMPLATES.md) if you are calling the LLM through `Webhooks by Zapier`.
