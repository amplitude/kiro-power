---
name: "query-analytics"
description: "Query Amplitude product analytics through the Amplitude MCP server — find and read charts, dashboards, and notebooks, run metric and dataset queries, and search session replays. Use when the user asks about user behavior, active users, conversion, funnels, retention, engagement, event volumes, a specific Amplitude chart or dashboard, or wants to find session recordings."
license: "MIT"
metadata:
  author: "Amplitude"
  version: "1.1.0"
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

Use these names. Several older leaf tools (`search`, `get_charts`, `query_chart`, `query_dataset`, `query_metric`, `get_dashboard`, `get_notebook`, `get_session_replays`, `get_context`) still exist in server code but are hidden when consolidation flags are on. Do not call them.

### Discovery and content
| Tool | Purpose |
|------|---------|
| `get_amplitude_context` | Org, user, and accessible projects (omit `projectId`); one project's settings when `projectId` is set |
| `search_amp_entities` | Find dashboards, charts, notebooks, experiments, and docs by natural-language query |
| `search_amp_data_taxonomy` | Semantic search over events, properties, and property values — never guess names |
| `get_amplitude_charts` | Read saved charts: `include` = `link` (default), `typed`, `definition`, `data` (run the chart), or `guide` |
| `use_amp_dashboards` | Dashboards; `action: "get"` to read layout and chart IDs |
| `use_amp_notebooks` | Notebooks; `action: "get"` to fetch content |
| `get_properties` | Event/user/group/derived property definitions (`propertyType: "event"` + `eventType` for one event) |
| `use_amplitude_metrics` | Metric *definitions* (`action: "get_metrics"`). For metric *values*, use `query_amplitude_data` |

### Query execution
| Tool | Purpose |
|------|---------|
| `query_amplitude_data` | Ad-hoc analytics (segmentation, funnel, retention, sessions, data table). Prefer typed `chart`; raw `definition` only as fallback |
| `render_amplitude_chart` | Render a chart visually from a `chartEditId` returned by `query_amplitude_data` |

### Session replay
| Tool | Purpose |
|------|---------|
| `get_amp_session_replay_info` | Find and inspect replays. `action`: `search` (default), `list`, `events`, `timeline` |
| `render_amp_session_replay` | Play matching replays after search/list when the user wants to see them |

`get_from_url` parses Amplitude URLs into entity IDs.

## Step-by-Step Guide

### 1. Establish context first

On the first analytics request in a session, call `get_amplitude_context` (no `projectId`) to learn the user's organization and accessible projects. When the user has multiple projects and the request is ambiguous, ask which project they mean rather than guessing. Call again with `projectId` for timezone, currency, and project AI context.

### 2. Search before you build

Most questions are already answered by saved content. Before constructing a custom query:

1. Call `search_amp_entities` with keywords from the user's question (e.g. "signup funnel", "weekly retention", "checkout conversion").
2. If a relevant chart or dashboard exists, prefer it — it encodes the team's agreed-upon definition of the metric.
3. Use `get_amplitude_charts` with `include: "definition"` or `"typed"` to read the definition, then `include: "data"` (up to 3 IDs) to fetch numbers. For dashboards, `use_amp_dashboards` with `action: "get"`.

This matters because metric definitions are opinionated: a team's "active user" or "conversion" often carries specific filters that a from-scratch query would miss.

### 3. Run custom queries when saved content doesn't fit

When no saved chart matches, or the user wants a variation (different time range, extra segment, new group-by):

1. Verify event and property names exist before querying. Never invent event names — use `search_amp_data_taxonomy` and `get_properties` to confirm the exact names in the project's taxonomy.
2. Build the query with `query_amplitude_data` (typed `chart` with `kind` `segmentation` | `funnel` | `retention` | `sessions` | `data_table`). Always set `projectId`, a descriptive `name`, and `date_range`.
3. State the time range and filters you used when presenting results, so the user can correct any assumption.
4. To show the chart, call `render_amplitude_chart` with the returned `chartEditId`.

### 4. Present results clearly

- Lead with the direct answer to the question, then supporting numbers.
- Include the time range, project, and any segment filters applied.
- When the result comes from a saved chart, name the chart and link it if a URL is available.
- Offer a natural follow-up (e.g. "Want this broken down by platform?") only when it is genuinely useful.

## Common Workflows

### Workflow: Answer a metrics question ("What were DAU last week?")
**Goal:** Return a trusted number fast.

1. `get_amplitude_context` (if project unknown) → confirm project.
2. `search_amp_entities` for an existing DAU/active-users chart.
3. If found: `get_amplitude_charts` with `include: "data"` and a last-7-days range if the saved chart allows. If not: `query_amplitude_data` on the project's primary activity events (`_active` or the team's activity event).
4. Report the daily values and the weekly trend. For unique counts, use overall unique users — do not sum daily uniques.

### Workflow: Analyze a funnel
**Goal:** Show conversion between steps and where users drop off.

1. `search_amp_entities` for an existing funnel chart matching the flow (e.g. "signup funnel").
2. Read its definition with `get_amplitude_charts` (`include: "typed"` or `"definition"`) to learn the canonical step events.
3. `get_amplitude_charts` `include: "data"`, or `query_amplitude_data` with `kind: "funnel"` for a modified version.
4. Report step-to-step conversion rates and highlight the largest drop-off.

### Workflow: Investigate a metric change ("Why did signups drop on Tuesday?")
**Goal:** Localize the change before explaining it.

1. Query the metric daily around the change window to confirm the drop is real, not noise.
2. Re-run grouped by likely dimensions one at a time: platform, country, device, version, acquisition channel.
3. When one segment explains the change, drill into that segment's events.
4. Optionally use `get_amp_session_replay_info` (`action: "search"`) filtered to affected users/events, then `render_amp_session_replay` if the user wants to watch.

### Workflow: Review a dashboard
**Goal:** Summarize the current state of a team's dashboard.

1. `search_amp_entities` with `entityTypes: ["DASHBOARD"]` for the dashboard by name.
2. `use_amp_dashboards` `action: "get"` to enumerate its charts.
3. `get_amplitude_charts` `include: "data"` for the charts the user cares about (ask before querying every chart on a large dashboard; max 3 IDs per call).
4. Summarize per-chart findings, flagging anything anomalous.

### Workflow: Find session replays
**Goal:** Surface recordings that show a specific behavior.

1. Clarify the criteria: which events, which user segment, what time window.
2. Verify event names via `search_amp_data_taxonomy` / `get_properties`.
3. Call `get_amp_session_replay_info` with `action: "search"` (default) and those filters. Always pass `projectId`.
4. Return matching sessions with enough metadata (time, user, triggering events). If the user wants to see them, call `render_amp_session_replay` in the same turn.

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

### Issue: Tool not found (`search`, `get_charts`, `query_dataset`, …)
**Cause:** Those leaf names are hidden under consolidation flags. The live tools are `search_amp_entities`, `get_amplitude_charts`, `query_amplitude_data`, `use_amp_dashboards`, `get_amp_session_replay_info`.
**Solution:** Retry with the names in the table above.

### Issue: No data returned for a query
**Cause:** Wrong project, wrong event name, or no access to the requested project.
**Solution:**
1. Call `get_amplitude_context` and confirm the project.
2. Verify the exact event name via `search_amp_data_taxonomy` / `get_properties`.
3. Confirm the time range actually contains data (try widening it).

### Issue: EU customer gets connection or auth errors
**Cause:** This power defaults to the US endpoint; EU-residency orgs live on a separate stack.
**Solution:** Edit the power's MCP configuration in Kiro and change the URL to `https://mcp.eu.amplitude.com/mcp`, then reconnect.

### Issue: Results disagree with the Amplitude web app
**Cause:** Different time range, timezone, or segment filters than the saved chart.
**Solution:** Read the chart definition with `get_amplitude_charts` (`include: "definition"`) and re-run with matching parameters; state both configurations to the user.

## References

- [Amplitude MCP documentation](https://amplitude.com/docs/amplitude-ai/amplitude-mcp)
- [Amplitude MCP server guide](https://github.com/amplitude/mcp-server-guide)
- Related skill: [analyze-experiments](../analyze-experiments/SKILL.md) for A/B tests and feature flags
- Related skill: [instrument-analytics](../instrument-analytics/SKILL.md) for adding tracking to code
