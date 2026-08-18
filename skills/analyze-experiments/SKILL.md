---
name: "analyze-experiments"
description: "Analyze Amplitude experiments and feature flags through the Amplitude MCP server — inspect experiment setups, evaluate A/B test results and statistical significance, and review feature flag configurations and rollouts. Use when the user asks about an A/B test, experiment results, which variant is winning, statistical significance, feature flag status, rollout percentages, or wants to connect a flag key in code to its Amplitude configuration."
license: "MIT"
metadata:
  author: "Amplitude"
  version: "1.0.0"
---

# Analyze Amplitude Experiments and Feature Flags

## Overview

This skill covers working with Amplitude Experiment through the Amplitude MCP server: finding experiments, reading their configuration, evaluating results with statistical rigor, and inspecting feature flags and rollout state. It is particularly useful inside the IDE, where flag keys appear in code and the user wants to know what they control, who sees which variant, and whether a flag is safe to remove.

**Key capabilities:**
- Summarize an experiment's hypothesis, variants, metrics, and current results.
- Report statistical significance honestly, including when results are not yet conclusive.
- Map a feature flag key found in code to its Amplitude configuration and rollout status.
- Identify stale flags that have shipped (100% rolled out or experiment concluded) and can be cleaned up.

## Prerequisites Checklist

- [ ] The Amplitude MCP server is connected and authenticated via OAuth (configured by this power at `https://mcp.amplitude.com/mcp`; EU orgs use `https://mcp.eu.amplitude.com/mcp`).
- [ ] The user's Amplitude org uses Amplitude Experiment (experiments/flags exist to analyze).
- [ ] The user has view access to the relevant Amplitude project.

## Available MCP Tools

| Tool | Purpose |
|------|---------|
| `search` | Find experiments (and related charts/dashboards) by name or topic |
| `get_experiments` | Get detailed experiment information: hypothesis, variants, allocation, metrics, status |
| `query_experiment` | Analyze experiment results and statistical significance |
| `get_flags` | Retrieve feature flag configurations: key, variants, targeting rules, rollout state |
| `get_context` | Access user and organization information (org, projects) |
| `query_dataset` | Run supporting event-level queries (e.g. exposure or downstream-metric checks) |

## Step-by-Step Guide

### 1. Locate the experiment or flag

- From a name or topic: call `search` with the user's phrasing (e.g. "homepage redesign", "checkout flow test").
- From a flag key in code: call `get_flags` and match the key exactly. Flag keys in code are the stable join point between the repository and Amplitude.
- If several candidates match, list them with status (running / decided / rolled out) and ask which one.

### 2. Read the configuration before the results

Call `get_experiments` for the chosen experiment and summarize:

- **Hypothesis and primary metric** — what the experiment is trying to move.
- **Variants and allocation** — including the control, and any mid-flight allocation changes.
- **Targeting** — which users are eligible (segment, platform, percentage).
- **Status and duration** — start date, planned end, current state.

This framing prevents the most common mistake: reporting a lift on a metric the experiment was never powered to detect.

### 3. Query and interpret results

Call `query_experiment` and report:

1. **Primary metric first**: lift per variant vs. control, with confidence intervals when available.
2. **Statistical significance, stated plainly**: "statistically significant" only when the analysis says so. If not significant, say the result is inconclusive — not "trending positive."
3. **Sample size and runtime**: flag experiments that are early (days of data, small exposure counts) and warn against peeking decisions.
4. **Secondary and guardrail metrics**: report regressions on guardrails even when the primary metric wins.

### 4. Recommend, don't overreach

When asked "should we ship it?", summarize the evidence (significance, lift size, guardrails, runtime) and give a recommendation with its caveats. Ship/no-ship is the team's call; your job is to make the statistics legible.

## Common Workflows

### Workflow: "What are the results of experiment X?"
**Goal:** A decision-ready summary.

1. `search` → identify the experiment; `get_experiments` for setup.
2. `query_experiment` for results.
3. Report: primary metric lift per variant, significance, exposure counts, runtime, guardrail status.
4. Conclude with what the evidence supports and what is still uncertain.

### Workflow: "What does this flag in my code do?"
**Goal:** Connect code to configuration.

1. Take the flag key from the code (e.g. `experiment.variant('new-onboarding-flow')`).
2. `get_flags` → find the matching key; report variants, targeting rules, rollout percentage, and status.
3. If the flag backs an experiment, follow with `get_experiments` / `query_experiment` for its results.
4. Show which code paths correspond to which variants.

### Workflow: Flag cleanup audit
**Goal:** Find flags that can be removed from code.

1. Collect flag keys referenced in the codebase (grep for the project's variant-lookup calls).
2. `get_flags` → check each key's status.
3. Flags that are decided/rolled out to 100% (or fully off with a concluded experiment) are cleanup candidates.
4. For each candidate, identify the winning code path to keep and the dead branches to delete. Propose the edits; let the user confirm before changing rollout state or deleting code.

### Workflow: Verify experiment exposure is firing
**Goal:** Confirm an experiment is actually collecting data.

1. `get_experiments` → confirm the experiment is running and note its exposure event.
2. `query_dataset` → check exposure event volume over the experiment window, grouped by variant.
3. Empty or heavily skewed variant volumes indicate an instrumentation or targeting problem — check the SDK integration (see [instrument-analytics](../instrument-analytics/SKILL.md)).

## Best Practices

- **Never declare a winner without significance.** Under-powered "wins" are the fastest way to erode trust in experimentation.
- **Report guardrail regressions unprompted.** A primary-metric win with a broken guardrail is not a win.
- **Distinguish flags from experiments.** A rollout flag has no control group; don't present rollout metrics as causal experiment results.
- **Use exact flag keys.** Keys are case- and punctuation-sensitive; match what's in code verbatim.
- **Flag early peeking.** If the experiment hasn't reached its planned duration or sample, say so before interpreting results.
- **Keep write operations out of scope.** This power reads experiment state; changing allocations, targeting, or rollout percentages should happen in the Amplitude web app by a human.

## Troubleshooting

### Issue: Experiment or flag not found
**Cause:** Wrong project, different key than the code suggests, or no view permission.
**Solution:**
1. `get_context` → confirm the org/project.
2. `search` with alternative names; flags are sometimes named differently from their keys.
3. Have the user confirm they can see the experiment in the Amplitude web app.

### Issue: Results look different from the Amplitude UI
**Cause:** Different analysis window or metric variant.
**Solution:** Compare the queried window and metric against the experiment's configured analysis settings from `get_experiments`, and re-run to match.

### Issue: No exposure data
**Cause:** SDK not sending exposure events, targeting excludes everyone, or the experiment just started.
**Solution:** Run the exposure-verification workflow above; if instrumentation is the problem, switch to the [instrument-analytics](../instrument-analytics/SKILL.md) skill.

## References

- [Amplitude Experiment documentation](https://amplitude.com/docs/feature-experiment)
- [Amplitude MCP documentation](https://amplitude.com/docs/amplitude-ai/amplitude-mcp)
- Related skill: [query-analytics](../query-analytics/SKILL.md) for general metric queries
- Related skill: [instrument-analytics](../instrument-analytics/SKILL.md) for SDK and exposure instrumentation
