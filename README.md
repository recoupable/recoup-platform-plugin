# Recoup Platform Plugin

Agent plugin for working with the [Recoup](https://recoupable.com) platform. Cross-cutting skills that let AI agents interact with the Recoup API, set up accounts, create and manage artists, and work inside sandbox workspaces.

This is the **starting point** for any Recoup integration — install it first, then add domain-specific plugins (research, content, catalogs) as needed.

## Install

### Claude Code (CLI)

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

If you've already added the [Recoup marketplace](https://github.com/recoupable/marketplace):

```bash
/plugin install recoup-platform-plugin@recoup-marketplace
```

## Setup

After installing, run the **getting-started** skill:

> Set up Recoup

This walks you through email verification, API key issuance, org lookup, and persisting credentials to Global instructions — so every future Claude session can reach the Recoup API automatically.

## Skills

| Skill | What it does |
|-------|-------------|
| [getting-started](skills/getting-started) | **First-run onboarding.** Email + PIN verification → API key → org lookup → memory block for Global instructions. Run once per account. |
| [recoup-api](skills/recoup-api) | **API reference + connector actions.** Call any Recoup endpoint (artists, research, content, connectors). Also handles external operations (Google Docs/Drive/Sheets, Gmail, TikTok, Instagram) via shared connections. |
| [create-artist](skills/create-artist) | **End-to-end artist creation.** 8-step chain: create → Spotify match → profile enrichment → Chartmetric research → catalog pull → social discovery → PATCH socials → synthesize knowledge base. Driven from a checklist file for resumability. |
| [artist-workspace](skills/artist-workspace) | **Workspace conventions.** How to create, navigate, and organize artist directories. Scaffolds the `RECOUP.md` checklist, manages context files, songs, and releases. The filesystem guide for agents. |
| [setup-sandbox](skills/setup-sandbox) | **Sandbox initialization.** Fetches orgs and artists via CLI, scaffolds the folder structure. Run once when a new sandbox is provisioned. |

## Layout

```
.claude-plugin/plugin.json   # Claude Code manifest
.codex-plugin/plugin.json    # Codex manifest
.cursor-plugin/plugin.json   # Cursor manifest
skills/
  getting-started/SKILL.md   # First-run onboarding
  recoup-api/SKILL.md        # API reference + connectors
  create-artist/SKILL.md     # 8-step artist creation chain
  artist-workspace/           # Workspace conventions + templates
    SKILL.md
    references/
      artist-template.md
      artist-example.md
      audience-template.md
      release-template.md
  setup-sandbox/SKILL.md     # Sandbox initialization
```

## Required environment

- `RECOUP_API_KEY` or `RECOUP_ACCESS_TOKEN` — obtain via `getting-started` skill or `POST /api/agents/signup`. See [Agents docs](https://developers.recoupable.com/agents).

## Related plugins

| Plugin | Focus |
|--------|-------|
| [recoup-research-plugin](https://github.com/recoupable/recoup-research-plugin) | Artist analytics, audience insights, playlist intelligence, competitive analysis, trend detection |
| [recoup-content-plugin](https://github.com/recoupable/recoup-content-plugin) | Video/image/caption generation, content pipelines |
| [recoup-catalogs-plugin](https://github.com/recoupable/recoup-catalogs-plugin) | Catalog acquisition, deal analysis, royalty normalization, valuation |

## Support

- Email: `support@recoupable.com`
- Website: <https://recoupable.com>
- Docs: <https://developers.recoupable.com>

## License

[Apache-2.0](./LICENSE)
