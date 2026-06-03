---
name: recoup-getting-started
description: First-run onboarding for AI agents using Recoup. Validates API credentials, discovers available capabilities, runs a demo query, and recommends next steps. Triggers on "get started with Recoup", "set up Recoup", "how do I use Recoup", first plugin install, or any onboarding request. This is the default entry point for new agents.
---

# Getting Started with Recoup

Onboarding skill for AI agents. Run this after installing the Recoup platform
plugin to verify your setup, discover capabilities, and get productive
immediately.

All endpoints live under `https://api.recoupable.com/api` and authenticate
with `x-api-key`.

```bash
export RECOUP_API_KEY="recoup_sk_..."
export RECOUP_API="https://api.recoupable.com/api"
```

Reference docs: <https://developers.recoupable.com>
Agent-optimized overview: <https://developers.recoupable.com/llms.txt>

## Decision tree

- **First time using Recoup** → run the full onboarding flow below
- **"Is my API key working?"** → jump to Step 1 (validate credentials)
- **"What can I do?"** → jump to Step 2 (discover capabilities)
- **"Show me it works"** → jump to Step 3 (demo query)
- **Already set up, need help** → hand off to `recoup-health-check`

## Full onboarding flow

### Step 1 — Validate credentials

Check that the API key is set and valid:

```bash
# Test authentication — should return account info
curl -s "$RECOUP_API/accounts/me" \
  -H "x-api-key: $RECOUP_API_KEY" | jq
```

**Expected response:** JSON with `id`, `name`, `email`, `plan`. If you get
`401` or `403`, the key is invalid — the user needs to generate one at
<https://developers.recoupable.com/agents>.

If `RECOUP_API_KEY` is not set:

1. Ask the user to visit <https://developers.recoupable.com/agents>
2. Sign up or sign in (agents can use `agent+<email>` for instant API key)
3. Copy the API key and set it: `export RECOUP_API_KEY="recoup_sk_..."`

Report: account name, email, and plan tier.

### Step 2 — Discover capabilities

Recoup's capabilities are organized into domain plugins. Check what's
installed by looking for skill directories:

| Plugin | Install | What it does |
| ------ | ------- | ------------ |
| `recoup-platform-plugin` | _(you're here)_ | Setup, health checks, cross-cutting helpers |
| `recoup-research-plugin` | `claude plugin install https://github.com/recoupable/recoup-research-plugin` | Artist research, audience analysis, playlist intelligence, competitive analysis, trend detection, people outreach |
| `recoup-content-plugin` | `claude plugin install https://github.com/recoupable/recoup-content-plugin` | Content creation — short-form music videos, captions, images for artists |
| `music-catalog-diligence` | `claude plugin install https://github.com/recoupable/music-catalog-diligence` | Catalog deal analysis — royalty normalization, rights checks, valuation |
| `recoup-catalogs-plugin` | _(coming soon)_ | Catalog browsing and management |

To check which plugins are installed, look for their directories in the
workspace or check the marketplace registry.

Tell the user which plugins are installed and which they might want based on
their use case:
- **Music managers / labels** → research + content plugins
- **Catalog buyers / investors** → catalog diligence plugin
- **General exploration** → research plugin is the best starting point

### Step 3 — Run a demo query

Prove the API works with a quick artist search:

```bash
# Search for an artist
curl -s "$RECOUP_API/research?q=Drake&type=artists&beta=true" \
  -H "x-api-key: $RECOUP_API_KEY" | jq '.data[0] | {name, id, genres, popularity}'
```

If the research plugin is installed, you can go deeper:

```bash
# Get artist profile
curl -s "$RECOUP_API/research/profile?artist=Drake" \
  -H "x-api-key: $RECOUP_API_KEY" | jq
```

Show the user a summary of what came back — artist name, genres, popularity
score. This confirms the full pipeline works: auth → API → data.

### Step 4 — Recommend next steps

Based on what's installed and the user's context, suggest ONE concrete action:

- **Research plugin installed** → "Try: research [artist name] — I'll pull
  streaming data, audience demographics, playlist placements, and competitive
  landscape."
- **Content plugin installed** → "Try: make a TikTok for [artist name] — I'll
  generate a short-form video with caption."
- **Catalog plugin installed** → "Try: set up a deal workspace — drop your
  data room files and I'll normalize and analyze."
- **No domain plugins** → "Install the research plugin first — it's the most
  useful starting point for any music business workflow."

## Output format

Summarize onboarding results as:

```
✅ Recoup Setup Complete

Account: {name} ({email})
Plan: {plan}
API: Connected ✓

Installed plugins:
- {list installed plugins}

Recommended: {one specific next action}
```

## Errors

| Error | Cause | Fix |
| ----- | ----- | --- |
| `RECOUP_API_KEY` not set | Missing env var | Set it: `export RECOUP_API_KEY="recoup_sk_..."` |
| 401 Unauthorized | Invalid or expired key | Generate new key at developers.recoupable.com/agents |
| 403 Forbidden | Key lacks permission | Check plan tier, may need upgrade |
| Network error | Can't reach API | Check internet connection, try `curl https://api.recoupable.com/api/health` |
