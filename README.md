# AI Ticket Triage Agent

A multi-stage AI agent that reads raw, unstructured support tickets, decides whether they're safe to auto-resolve, and either drafts a response or escalates to a human with full context — with a live dashboard tracking whether the automation is actually being trusted.

## The problem

Support teams get a mixed inbox: simple, repetitive questions with a known answer sitting right next to urgent, ambiguous, or emotionally charged tickets that genuinely need a person. Without a system to sort them, every ticket costs the same amount of attention, and urgent issues can sit behind routine ones in the queue.

Separately, most AI automation pilots inside companies don't fail because the model is bad — they fail because nobody builds the layer that proves the AI is actually working after deployment, so leadership loses confidence and the pilot quietly dies. This project addresses both: it automates the triage decision, and it measures whether that decision is trustworthy.

## How it works

```
Raw ticket
   │
   ▼
┌─────────────────┐
│ 1. Classify      │  Claude reads the ticket → category, urgency, sentiment
└────────┬─────────┘
         ▼
┌─────────────────┐
│ 2. Retrieve/Draft│  Claude checks it against a small FAQ knowledge base →
│                  │  decides: does this FAQ actually resolve the question,
│                  │  or just share a topic with it?
└────────┬─────────┘
         ▼
┌─────────────────┐
│ 3. Route         │  Pure logic, no AI:
│  (no AI here)    │  low-risk + FAQ match + not urgent  → auto-send
│                  │  anything else                      → escalate to human
└────────┬─────────┘
         ▼
   Logged to an audit trail (every run, both branches)
         ▼
   Live Power BI dashboard
```

**Stage 1 — Classify.** Claude reads the raw ticket and returns structured JSON: category, urgency, sentiment, and a one-sentence summary. Temperature is set low (0.2) since this needs to be consistent, not creative.

**Stage 2 — Retrieve & Draft.** Claude checks the ticket against five FAQ topics (password reset, billing/payment, data export, feature access by plan, subscription changes). It's instructed to distinguish *topical* overlap from the FAQ actually *resolving the specific question asked* — a distinction that mattered in practice (see [Design decisions](#key-design-decision-risk-tiered-not-blanket-automation) below). Output includes whether it matched, the action to take, and either a customer-facing draft or an internal handoff summary for a human.

**Stage 3 — Route.** Deliberately has zero AI in it — just a conditional check on the `action` field from Stage 2. Auto-respond cases get sent via email automatically. Escalations post a Slack alert with full context. Every run, regardless of outcome, gets logged.

## Key design decision: risk-tiered, not blanket, automation

The system does not try to automate everything it can. Only tickets that are *both* low-risk (cleanly FAQ-matched) *and* non-urgent get sent without review. Anything ambiguous, urgent, or outside the FAQ's real scope goes to a human — even if it superficially matched a topic.

This came from a real bug caught during testing: a ticket asking *why* a payment was declined initially matched the "update your payment method" FAQ topically, but that FAQ doesn't actually answer "why." The matching logic was tightened so a match requires the FAQ to substantively resolve the specific question, not just share a subject.

This reflects current thinking in AI agent deployment — optimizing for accuracy over raw automation rate, since inflated auto-resolution with poor accuracy is a documented failure mode that increases repeat contacts and erodes trust.

## Testing infrastructure

A second workflow generates synthetic test tickets using Claude itself, with an explicit quota in the prompt (e.g., a fixed number of low/medium/high-urgency tickets, split further by FAQ-match vs. no-match) rather than a vague "add some variety" instruction — which in practice produced skewed, unrepresentative batches. This guarantees every branch of the routing logic gets exercised on each test run. The batch runs automatically through the full agent via an n8n Loop Over Items + Execute Sub-workflow pattern — one click, no manual data entry.

## Results

From a 72-ticket test batch:

| Metric | Result |
|---|---|
| Auto-resolution rate | 36% |
| Estimated manual triage time saved | ~6 hours |
| Tickets escalated with full context | 64% |

The auto-resolution rate was not tuned upward artificially — accuracy was prioritized over volume.

## Dashboard

Built in Power BI, connected live to the same Google Sheets audit trail the agent writes to (not a static export) — running more tickets through the agent and refreshing the dashboard reflects the new data immediately.

Shows: total volume, auto-resolve vs. escalate split, category and urgency breakdowns, and — the key piece — a status breakdown of what happened to escalated tickets after the fact (resolved / in progress / pending). That last part is what turns this from "an AI tool" into "evidence the AI tool is trusted and used," which is the part most AI pilots skip.

*(Add a dashboard screenshot here.)*

## Tech stack

- **Claude API** (Claude Haiku 4.5) — classification and drafting, structured JSON output
- **n8n** — workflow orchestration, sub-workflow/loop patterns, webhook triggers
- **Google Sheets** — live audit-trail data layer
- **Power BI** — live-connected dashboard, DAX measures, Power Query cleaning
- **Slack** — escalation alerts via webhook
- **Gmail** — automated send for low-risk resolutions

## What this looks like in a real deployment

This prototype uses a manual/batch trigger and Gmail/Sheets as stand-ins for a real company's systems. In production, the entry point would be a live inbox trigger or a webhook from whatever helpdesk platform the company already uses (Zendesk, Freshdesk, Intercom, Jira Service Management) — all of which fire an event the moment a ticket is created. Auto-responses and escalations would write back through that platform's API instead of a personal email account. The reasoning core (classify → retrieve/draft → route) doesn't change between prototype and production — only the "front door" and "front door out" swap for the client's real systems.

## Lessons from building this

- Vague prompt instructions ("include a mix") don't reliably produce balanced test data — explicit quotas do.
- LLM output that's asked for raw JSON can still arrive wrapped in markdown fences; defensive parsing (strip fences, validate, fall back gracefully) is worth building in from the start, not after the first failure.
- The most valuable design decision in the whole project wasn't a prompt or a workflow node — it was deciding which class of tickets the AI should *never* act on alone.
