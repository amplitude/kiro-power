# Amplitude power for Kiro

A [Kiro](https://kiro.dev) power that brings [Amplitude](https://amplitude.com) into your IDE. Ask the Kiro agent about your product data, dig into A/B test results and feature flags, search session replays, and instrument your code with Amplitude SDKs — all grounded in your project's live event taxonomy.

The power connects Kiro to Amplitude's official hosted MCP server and ships three Agent Skills that teach the agent how to use it well.

## What you can do

- **Query analytics** — "What were daily active users last week?", "Show me the signup funnel", "Why did conversions drop on Tuesday?" The agent finds your saved charts and dashboards or runs custom queries.
- **Analyze experiments and flags** — "What are the results of the homepage redesign test?", "What does the `new-onboarding-flow` flag in this file control?", "Which feature flags can we clean up?"
- **Instrument your code** — "Add Amplitude tracking to the checkout flow", "Set up session replay", "Why aren't my events showing up?" The agent checks your existing taxonomy first so new events fit your naming conventions.

## Install

### From the Kiro Powers marketplace

Once listed, open the Powers panel in Kiro, find **Amplitude**, and click install.

### From this GitHub repo

In Kiro, open the Powers panel → **Add power from GitHub** → paste this repository's URL.

### From a local checkout

```sh
git clone https://github.com/amplitude/kiro-power.git
```

Then in Kiro: Powers panel → **Add power from Local Path** → select the cloned directory.

## Authentication

The Amplitude MCP server uses **OAuth 2.0** — no API keys or tokens to configure. The first time Kiro connects, your browser opens to sign in to Amplitude and authorize access. The connection respects your existing Amplitude permissions: you can only access the projects and data you can see in the Amplitude web app.

Amplitude MCP is available to all Amplitude customers, including the Free plan.

### EU data residency

This power defaults to the US endpoint (`https://mcp.amplitude.com/mcp`). If your organization is on Amplitude's EU data center, edit the power's MCP server configuration in Kiro and change the URL to:

```
https://mcp.eu.amplitude.com/mcp
```

## Repository layout

This power follows the [Agent Plugins v1.0.0 specification](https://agent-plugins.org/), so it is portable to any conformant client.

```
.
├── plugin.json                        # power manifest
├── mcp.json                           # Amplitude remote MCP server configuration
└── skills/
    ├── query-analytics/               # charts, dashboards, metrics, funnels, session replay search
    │   └── SKILL.md
    ├── analyze-experiments/           # A/B test results, significance, feature flags
    │   └── SKILL.md
    └── instrument-analytics/          # SDK setup, event tracking, taxonomy, session replay
        └── SKILL.md
```

## Requirements

- An Amplitude account with access to at least one organization/project ([sign up free](https://amplitude.com/get-started))
- Kiro with Powers support

## Notes

- The Amplitude MCP server is under active development; some capabilities may change and heavy usage may be rate limited.
- Data queried through the MCP server is processed by the AI model driving your agent session. Review your organization's policies on AI data processing, and note that Amplitude org administrators can control MCP server access.

## Support

- [Amplitude MCP documentation](https://amplitude.com/docs/amplitude-ai/amplitude-mcp)
- [Amplitude community](https://community.amplitude.com/)
- [Amplitude support](https://support.amplitude.com/)
- Issues with this power: use this repository's issue tracker

## License

[MIT](LICENSE)
