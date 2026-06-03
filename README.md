# Recoup Platform Plugin

The starting point for AI agents using [Recoup](https://recoupable.com) — music business infrastructure that works with any AI tool.

Install this plugin first. It validates your setup, discovers capabilities, and gets you productive immediately. Then add domain plugins (research, content, catalog diligence) based on your workflow.

## Skills

| Skill | What it does |
| ----- | ------------ |
| `recoup-getting-started` | First-run onboarding — validates API key, discovers capabilities, runs a demo query, recommends next steps. |
| `recoup-health-check` | Diagnostic checks — API connectivity, auth, credit balance, plugin status, troubleshooting. |

## Install

### Claude Code

```bash
claude plugin install https://github.com/recoupable/recoup-platform-plugin
```

### Claude Cowork

1. Open the plugin marketplace (puzzle-piece icon in the sidebar).
2. Click **Add custom plugin** and paste:
   `https://github.com/recoupable/recoup-platform-plugin`
3. Approve the requested tool permissions.
4. Restart the Cowork session so manifests load.

### Codex

```bash
codex plugin install https://github.com/recoupable/recoup-platform-plugin
```

### Cursor

1. Cursor → Settings → Plugins → **Add custom plugin**.
2. Paste the GitHub URL above.
3. Restart Cursor so `.cursor-plugin/plugin.json` loads.

### Via Marketplace

If you've already added the [Recoup marketplace](https://github.com/recoupable/plugins):

```bash
/plugin install recoup-platform-plugin@recoup-marketplace
```

## Getting started

After install, set your Recoup API key:

```bash
export RECOUP_API_KEY="recoup_sk_..."   # see https://developers.recoupable.com/agents
```

Then ask your agent:

> **Get started with Recoup**

The agent will validate your setup, show you what's available, and suggest your first action.

## Other Recoup plugins

| Plugin | Purpose | Install |
| ------ | ------- | ------- |
| [recoup-research-plugin](https://github.com/recoupable/recoup-research-plugin) | Artist research, audience analysis, playlist intelligence, competitive analysis | `claude plugin install https://github.com/recoupable/recoup-research-plugin` |
| [recoup-content-plugin](https://github.com/recoupable/recoup-content-plugin) | Content creation — short-form music videos, captions, images | `claude plugin install https://github.com/recoupable/recoup-content-plugin` |
| [music-catalog-diligence](https://github.com/recoupable/music-catalog-diligence) | Catalog deal analysis — royalty normalization, rights checks, valuation | `claude plugin install https://github.com/recoupable/music-catalog-diligence` |

## API reference

- Base URL: `https://api.recoupable.com/api`
- Auth: `x-api-key` header
- Docs: <https://developers.recoupable.com>
- Agent overview: <https://developers.recoupable.com/llms.txt>

## License

Apache-2.0
