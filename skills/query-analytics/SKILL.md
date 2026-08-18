---
name: "query-analytics"
description: "Query Amplitude product analytics through the Amplitude MCP server — find and read charts, dashboards, and notebooks, run metric and dataset queries, and search session replays. Use when the user asks about user behavior, active users, conversion, funnels, retention, engagement, event volumes, a specific Amplitude chart or dashboard, or wants to find session recordings."
license: "MIT"
metadata:
  author: "Amplitude"
  version: "1.0.0"
---

# Query Amplitude Analytics

## Overview

This skill covers answering product-data questions with the Amplitude MCP server: discovering existing content (charts, dashboards, notebooks), reading definitions, executing queries, and searching session replays. Results respect the signed-in user's Amplitude permissions — you can only read projects and content the user can see in the Amplitude web app.

**Key capabilities:**
- Answer questions like "What were daily active users last week?" or "Show me the signup funnel."
- Reuse saved charts and dashboards instead of rebuilding analyses from scratch.
- Run ad-hoc metric and dataset queries with custom time ranges, segments, and filters.
- Find session replays matching specific user or event criteria.

## Prerequisites Checklist

- [ ] The Amplitude MCP server is connected (this power configures it at `https://mcp.amplitude.com/mcp`).
- [ ] The user has completed the OAuth sign-in flow when Kiro first connected to the server.
- [ ] The user's Amplitude account has access to at least one project.
- [ ] EU data residency customers have switched the server URL to `https://mcp.eu.amplitude.com/mcp` (see Troubleshooting).

## Available MCP Tools

### Discovery and content
| Tool | Purpose |
|------|---------|
| `search` | Find dashboards, charts, notebooks, and experiments by natural-language query |
| `get_charts` | Retrieve full chart definitions (events, segments, metrics, chart type) |
| `get_dashboard` | Access dashboard content and layout, including its charts |
| `get_notebook` | Fetch notebook content with embedded charts and analysis |
| `get_context` | Access user and organization information (org, projects, permissions) |
| `get_event_properties` | Explore event property definitions in the taxonomy |

### Query execution
| Tool | Purpose |
|------|---------|
| `query_chart` | Execute a saved chart and return its data |
| `query_metric` | Get data for a specific metric |
| `query_dataset` | Execute a custom analytics query (events, filters, group-bys, time range) |

### Session replay
| Tool | Purpose |
|------|---------|
| `get_session_replays` | Search and filter session recordings by user, event, and time criteria |

## Step-by-Step Guide

### 1. Establish context first

On the first analytics request in a session, call `get_context` to learn the user's organization and accessible projects. When the user has multiple projects and the request is ambiguous, ask which project they mean rather than guessing.

### 2. Search before you build

Most questions are already answered by saved content. Before constructing a custom query:

1. Call `search` with keywords from the user's question (e.g. "signup funnel", "weekly retention", "checkout conversion").
2. If a relevant chart or dashboard exists, prefer it — it encodes the team's agreed-upon definition of the metric.
3. Use `get_charts` or `get_dashboard` to read the definition, then `query_chart` to fetch its data.

This matters because metric definitions are opinionated: a team's "active user" or "conversion" often carries specific filters that a from-scratch query would miss.

### 3. Run custom queries when saved content doesn't fit

When no saved chart matches, or the user wants a variation (different time range, extra segment, new group-by):

1. Verify event and property names exist before querying. Never invent event names — use `search` and `get_event_properties` to confirm the exact names in the project's taxonomy.
2. Build the query with `query_dataset` (event-level analyses) or `query_metric` (defined metrics).
3. State the time range and filters you used when presenting results, so the user can correct any assumption.

### 4. Present results clearly

- Lead with the direct answer to the question, then supporting numbers.
- Include the time range, project, and any segment filters applied.
- When the result comes from a saved chart, name the chart and link it if a URL is available.
- Offer a natural follow-up (e.g. "Want this broken down by platform?") only when it is genuinely useful.

## Common Workflows

### Workflow: Answer a metrics question ("What were DAU last week?")
**Goal:** Return a trusted number fast.

1. `get_context` (if project unknown) → confirm project.
2. `search` for an existing DAU/active-users chart.
3. If found: `query_chart` with the last-7-days range. If not: `query_dataset` on the project's primary activity events.
4. Report the daily values and the weekly trend.

### Workflow: Analyze a funnel
**Goal:** Show conversion between steps and where users drop off.

1. `search` for an existing funnel chart matching the flow (e.g. "signup funnel").
2. Read its definition with `get_charts` to learn the canonical step events.
3. `query_chart` (or `query_dataset` for a modified version).
4. Report step-to-step conversion rates and highlight the largest drop-off.

### Workflow: Investigate a metric change ("Why did signups drop on Tuesday?")
**Goal:** Localize the change before explaining it.

1. Query the metric daily around the change window to confirm the drop is real, not noise.
2. Re-run grouped by likely dimensions one at a time: platform, country, device, version, acquisition channel.
3. When one segment explains the change, drill into that segment's events.
4. Optionally use `get_session_replays` filtered to affected users/events to observe the behavior directly.

### Workflow: Review a dashboard
**Goal:** Summarize the current state of a team's dashboard.

1. `search` for the dashboard by name.
2. `get_dashboard` to enumerate its charts.
3. `query_chart` for each chart the user cares about (ask before querying every chart on a large dashboard).
4. Summarize per-chart findings, flagging anything anomalous.

### Workflow: Find session replays
**Goal:** Surface recordings that show a specific behavior.

1. Clarify the criteria: which events, which user segment, what time window.
2. Verify event names via `search` / `get_event_properties`.
3. Call `get_session_replays` with those filters.
4. Return the matching sessions with enough metadata (time, user, triggering events) for the user to pick which to watch.

## Best Practices

- **Prefer saved definitions over ad-hoc queries.** Saved charts encode the team's metric definitions.
- **Confirm taxonomy before querying.** Event and property names are project-specific; a misspelled event silently returns zeros.
- **Be explicit about time ranges.** Default to a sensible window (last 7 or 30 days) and say which one you used.
- **Break complex analyses into focused questions.** One query per question beats a single sprawling query.
- **Watch for rate limits.** The Amplitude MCP server is under active development and may rate-limit heavy usage; batch questions rather than issuing many redundant queries.
- **Respect data sensitivity.** Query results are product data; don't copy them into code, commits, or files unless the user asks.

## Troubleshooting

### Error: Authentication failed / tools unavailable
**Cause:** OAuth session missing or expired, or the org admin has disabled MCP access.
**Solution:**
1. Reconnect the `amplitude` MCP server in Kiro to re-trigger the OAuth browser flow.
2. Verify the user can sign in at amplitude.com with the same account.
3. If it persists, have the user check with their Amplitude org administrator about MCP server access.

### Issue: No data returned for a query
**Cause:** Wrong project, wrong event name, or no access to the requested project.
**Solution:**
1. Call `get_context` and confirm the project.
2. Verify the exact event name via `search` / `get_event_properties`.
3. Confirm the time range actually contains data (try widening it).

### Issue: EU customer gets connection or auth errors
**Cause:** This power defaults to the US endpoint; EU-residency orgs live on a separate stack.
**Solution:** Edit the power's MCP configuration in Kiro and change the URL to `https://mcp.eu.amplitude.com/mcp`, then reconnect.

### Issue: Results disagree with the Amplitude web app
**Cause:** Different time range, timezone, or segment filters than the saved chart.
**Solution:** Read the chart definition with `get_charts` and re-run with matching parameters; state both configurations to the user.

## References

- [Amplitude MCP documentation](https://amplitude.com/docs/amplitude-ai/amplitude-mcp)
- [Amplitude MCP server guide](https://github.com/amplitude/mcp-server-guide)
- Related skill: [analyze-experiments](../analyze-experiments/SKILL.md) for A/B tests and feature flags
- Related skill: [instrument-analytics](../instrument-analytics/SKILL.md) for adding tracking to code
