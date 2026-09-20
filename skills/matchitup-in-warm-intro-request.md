---
name: Find a Match It Up member and request a warm intro
description: Search Match It Up's Pro/Elite members by natural-language intent, research the best candidate, send a warm intro request card and pick up the accept/decline outcome.
api: openapi/matchitup-in-openapi.yml
base_url: https://matchitup.in
operations:
  - intent_discovery_search_api_protocol_match_search_post
  - research_person_endpoint_api_protocol_research_person_post
  - why_meet_endpoint_api_protocol_why_meet_post
  - create_intro_request_api_protocol_intro_request_post
  - get_signal_inbox_api_agent_notifications_get
  - get_agent_credits_api_protocol_agents__agent_id__credits_get
generated: '2026-09-19'
method: generated
source: https://matchitup.in/api/docs/agent-instructions.md (External Agent -> MIU User Workflows v3.1.0; Decision Hierarchy)
---

# Find a Match It Up member and request a warm intro

This is the flow that reaches real humans, so it is the most rate-limited surface on the platform: on the Dev
Sandbox tier, **20 member searches/day and 5 intro requests/day per agent**. Members must be Pro/Elite and
opted in to external-agent discovery; a request to someone who opted out returns 403.

## Steps

1. **Search by intent** — `intent_discovery_search_api_protocol_match_search_post`
   (`POST /api/protocol/match/search`, `X-API-Key`) with `{"query": "<natural language>", "limit": 5}`.
   Returns name, company, headline, **MU-Pin**, offers summary and profile_url. No PII beyond that.
2. **Qualify before you spend a request** — the provider's own decision hierarchy is relevance (bilateral
   offers/needs overlap), intent match, outcome probability, recency; do not engage below 50% overlap.
   `research_person_endpoint_api_protocol_research_person_post` (`POST /api/protocol/research-person`, by
   name/company/LinkedIn URL) and `why_meet_endpoint_api_protocol_why_meet_post` (`POST /api/protocol/why-meet`)
   produce the web-signal profile and the bilateral "why meet" rationale. Both are 0 credits per the MCP tool list.
3. **Send the intro request** — `create_intro_request_api_protocol_intro_request_post`
   (`POST /api/protocol/intro-request`) with `{"target_mu_pin": "MU-XXXX", "message": "...", "context": "optional"}`.
   The member receives an Accept/Decline card in their inbox. A duplicate pending request returns 409.
4. **Collect the outcome** — the requesting agent gets a Signal Inbox notification on `intro_accept` /
   `intro_decline`: `get_signal_inbox_api_agent_notifications_get` (`GET /api/agent/notifications`), or the
   `intro_request` webhook event. There is no withdraw operation — once sent, the request is the member's to answer.
5. **Budget** — `get_agent_credits_api_protocol_agents__agent_id__credits_get` before a batch; a 402 mid-flow
   means credits are exhausted until `reset_at`.

## Rules that bite

- Intro requests are irreversible and not idempotent; check the Signal Inbox for a pending request before
  resending. 429 (with `Retry-After`) when the daily quota is hit.
- Do not fall back to DMing the member: DMs to Match It Up users go through
  `POST /api/protocol/agents/{agent_id}/dm` (0.25 cr, 1-hour lock for unclaimed agents, 3 DMs/24h per pair
  soft limit) and a blocked recipient returns a silent 403.
- Everything here is the live network — the "Dev Sandbox" is a free tier on production, not a test environment
  (sandbox/matchitup-in-sandbox.yml).
