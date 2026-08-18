---
name: "instrument-analytics"
description: "Instrument applications with Amplitude Analytics SDKs — install and initialize the Browser, Node.js, or Python SDK, track events with a clean taxonomy, identify users, enable autocapture and Session Replay, and validate that events arrive. Use when the user wants to add Amplitude to an app, track an event, set user properties, set up session replay, fix events that aren't showing up, or audit existing tracking code."
license: "MIT"
metadata:
  author: "Amplitude"
  version: "1.0.0"
---

# Instrument Amplitude Analytics

## Overview

This skill covers adding and maintaining Amplitude instrumentation in application code: SDK setup, event tracking, user identification, autocapture, Session Replay, and validating the result against the project's live taxonomy via the Amplitude MCP server. The MCP server makes instrumentation work markedly better inside the IDE: before writing tracking code you can check the project's existing event names and properties, so new code extends the taxonomy instead of fragmenting it.

**Key capabilities:**
- Set up the Amplitude SDK in web (TypeScript/JavaScript), Node.js, and Python codebases.
- Write `track` / `identify` calls that match the project's existing naming conventions.
- Enable autocapture and the Session Replay plugin.
- Verify instrumentation end-to-end by querying for the new events.

## Prerequisites Checklist

- [ ] An Amplitude project and its **API key** (Amplitude web app → Settings → Projects → *project* → General). The API key identifies the project and is safe to ship in client code, unlike the secret key, which must never appear in client code.
- [ ] For taxonomy checks and validation: the Amplitude MCP server connected and authenticated (configured by this power).
- [ ] Package manager access to install `@amplitude/*` packages (or `amplitude-analytics` for Python).

## Step-by-Step Guide

### 1. Check the existing taxonomy first

Before adding any tracking call, look at what the project already has:

1. Use the MCP `search` and `get_event_properties` tools to list existing event names and property definitions.
2. Match the established conventions exactly — casing, tense, and separator style (many teams use `Noun Verbed`, e.g. `Song Played`; others use snake_case).
3. Reuse existing events and properties when the new code path is semantically the same action. Only mint a new event name for a genuinely new action, and keep it consistent with the convention.

Also check the repository itself: grep for existing `amplitude.track(` / `track(` calls and follow the local wrapper if the codebase routes analytics through one (many do). Extend the wrapper; don't bypass it.

### 2. Install and initialize the SDK

#### Web / Browser (TypeScript or JavaScript)

```sh
npm install @amplitude/analytics-browser
```

```ts
import * as amplitude from '@amplitude/analytics-browser';

amplitude.init(AMPLITUDE_API_KEY, {
  autocapture: true,
});
```

Initialize once, as early as possible in the app lifecycle (entry point, root layout, or app bootstrap). `autocapture: true` enables automatic tracking of page views, sessions, form interactions, file downloads, and marketing attribution; it can also be an object to toggle individual sources, e.g.:

```ts
amplitude.init(AMPLITUDE_API_KEY, {
  autocapture: {
    pageViews: true,
    sessions: true,
    formInteractions: true,
    fileDownloads: false,
    attribution: true,
    elementInteractions: true,
  },
});
```

#### Node.js (server-side)

```sh
npm install @amplitude/analytics-node
```

```ts
import { init, track } from '@amplitude/analytics-node';

init(AMPLITUDE_API_KEY);

track('Subscription Renewed', { plan: 'pro' }, { user_id: 'user@example.com' });
```

Server-side calls must pass `user_id` (or `device_id`) per event — there is no ambient user context on a server.

#### Python (server-side)

```sh
pip install amplitude-analytics
```

```python
from amplitude import Amplitude, BaseEvent

client = Amplitude(AMPLITUDE_API_KEY)

client.track(BaseEvent(
    event_type="Subscription Renewed",
    user_id="user@example.com",
    event_properties={"plan": "pro"},
))
```

#### Other platforms

Amplitude ships SDKs for iOS (Swift), Android (Kotlin), React Native, Flutter, Unity, and more. The taxonomy and validation guidance in this skill applies to all of them; consult the platform's page under the [Amplitude SDK docs](https://amplitude.com/docs/sdks) for setup specifics rather than guessing API shapes.

### 3. Track events

```ts
amplitude.track('Song Played', {
  songId: 'song-123',
  source: 'playlist',
  durationSeconds: 214,
});
```

Guidelines for every tracking call:

- **Event name = the action**, properties = the details. Don't encode variants in the name (`Button Clicked - Blue` is wrong; use a `variant` property).
- **Keep property values low-cardinality** where analysis will group by them (enums, booleans, small numbers of categories). IDs are fine as properties but poor as group-by dimensions.
- **Never send secrets or sensitive PII** in event properties.
- **Track at the moment the action succeeds**, not when it's attempted, unless failures are explicitly what's being measured (then track both with a `status` property).

### 4. Identify users and set user properties

```ts
// When the user logs in:
amplitude.setUserId('user@example.com');

// Set persistent user properties:
const identify = new amplitude.Identify();
identify.set('plan', 'premium');
identify.add('login_count', 1);
amplitude.identify(identify);

// When the user logs out:
amplitude.reset();
```

Use user properties for facts about the user that persist across events (plan, role, cohort); use event properties for facts about a single action.

### 5. Enable Session Replay (optional)

```sh
npm install @amplitude/plugin-session-replay-browser
```

```ts
import * as amplitude from '@amplitude/analytics-browser';
import { sessionReplayPlugin } from '@amplitude/plugin-session-replay-browser';

amplitude.add(sessionReplayPlugin({ sampleRate: 0.1 }));
amplitude.init(AMPLITUDE_API_KEY, { autocapture: true });
```

Add the plugin **before** calling `init`. Start with a conservative `sampleRate` (e.g. `0.1` = 10% of sessions) and raise it deliberately; replay volume has cost and privacy implications. Session Replay masks sensitive inputs by default — review the masking configuration with the user before increasing capture scope.

### 6. Validate end-to-end

1. Run the app and exercise the instrumented code path.
2. In development, verify delivery locally: watch the network tab for `api2.amplitude.com` requests (or set the SDK `logLevel` to Debug), or check Amplitude's User Lookup / event stream in the web app.
3. After events have had a few minutes to ingest, confirm via MCP: `query_dataset` (or `search` for the event) filtered to the new event name and a recent window.
4. Check the received properties match the intended names and types — a typo'd property silently creates a new one.

## Common Workflows

### Workflow: Add Amplitude to an app that has none
**Goal:** Working baseline instrumentation.

1. Confirm platform and get the API key from the user (put it in the app's existing config/env mechanism, not hardcoded).
2. Install and initialize the SDK with autocapture (web) at the app entry point.
3. Add `setUserId` / `reset` at the auth boundaries.
4. Instrument 3–5 core actions that map to the product's key funnel — not every button.
5. Validate end-to-end (step 6 above).

### Workflow: Track a new event in an already-instrumented app
**Goal:** One new event that fits the taxonomy.

1. Check existing taxonomy via MCP and existing `track` calls in the repo; find the naming convention and any analytics wrapper.
2. Add the call through the wrapper, matching conventions.
3. Validate the event arrives with the expected properties.

### Workflow: Debug "events aren't showing up"
**Goal:** Find where the pipeline breaks.

1. Confirm the tracking code actually executes (breakpoint/log at the call site).
2. Confirm the SDK is initialized before the call, with the right API key for the right project (`get_context` via MCP shows which projects the user expects).
3. Check the network: requests to `api2.amplitude.com` succeeding? Ad blockers and strict CSP commonly block client-side analytics — check the console, and consider a proxy if the team already uses one.
4. Check for typos: event name queried vs. event name sent must match exactly.
5. Remember ingestion latency: newly sent events can take a few minutes to appear in queries.

### Workflow: Audit existing instrumentation
**Goal:** A report on tracking health.

1. Grep the codebase for all tracking calls; list event names and properties actually sent.
2. Pull the project taxonomy via MCP and diff: events in code but unused in charts, near-duplicate names (`Sign Up` vs `signup`), properties with inconsistent types.
3. Report findings with concrete consolidation suggestions; let the user decide before renaming anything (renames break existing charts).

## Best Practices

- **Taxonomy before code.** Five well-named events beat fifty ad-hoc ones. Always check what exists first.
- **One initialization, one place.** Multiple `init` calls or scattered SDK setup cause double-counting and session fragmentation.
- **Route through a wrapper.** A thin `analytics.ts` / `analytics.py` module keeps naming consistent and makes future changes cheap.
- **Use the API key appropriate to the environment** — separate dev/staging/prod Amplitude projects prevent test noise polluting production data.
- **Don't block the product on analytics.** Tracking calls must never throw into product code paths; SDKs queue and retry internally.
- **Respect privacy.** Honor the app's consent mechanism before initializing tracking where required (GDPR/CCPA contexts), and keep PII out of event properties.

## Troubleshooting

### Error: Events attributed to the wrong project
**Cause:** Wrong API key (often a dev key in prod or vice versa).
**Solution:** Check the key source (env var per environment); confirm the target project's key in Amplitude Settings → Projects.

### Issue: Duplicate events
**Cause:** SDK initialized twice (e.g. React strict-mode double-mount running init in a component), or both autocapture and a manual call tracking the same action.
**Solution:** Move `init` to module scope / a guaranteed-once bootstrap; pick either the autocaptured event or the manual event for a given action, not both.

### Issue: Session Replay not recording
**Cause:** Plugin added after `init`, sample rate excluding the session, or missing replay entitlement on the plan.
**Solution:** Ensure `amplitude.add(sessionReplayPlugin(...))` precedes `amplitude.init(...)`; temporarily set `sampleRate: 1` in development; confirm Session Replay is enabled for the org.

### Issue: TypeScript can't find the package
**Cause:** Importing from the wrong package name.
**Solution:** Browser is `@amplitude/analytics-browser`, Node is `@amplitude/analytics-node`; both ship their own type definitions.

## References

- [Amplitude SDK documentation](https://amplitude.com/docs/sdks)
- [Browser SDK 2 documentation](https://amplitude.com/docs/sdks/analytics/browser/browser-sdk-2)
- [Session Replay documentation](https://amplitude.com/docs/session-replay)
- [Data taxonomy playbook](https://amplitude.com/blog/data-taxonomy-playbook)
- Related skill: [query-analytics](../query-analytics/SKILL.md) for validating events with queries
- Related skill: [analyze-experiments](../analyze-experiments/SKILL.md) for experiment exposure instrumentation
