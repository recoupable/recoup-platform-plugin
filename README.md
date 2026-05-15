# Recoup Platform Plugin

Agent plugin for working with the [Recoup](https://recoupable.com)
platform. A home for cross-cutting skills, commands, and helpers that
let AI agents interact with Recoup's chat, API, and surrounding
workflows.

This plugin is intentionally a **starter skeleton** — manifests are in
place, but skills, agents, and commands will be added in follow-up
releases. The package is published so the marketplace registry can
point to it; consumers who install it today will get the structure
without any active commands.

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

### Cursor

1. Cursor → Settings → Plugins → **Add custom plugin**.
2. Paste the GitHub URL above.
3. Restart Cursor so `.cursor-plugin/plugin.json` loads.

## Layout

```
.claude-plugin/plugin.json   # Claude Code manifest
.codex-plugin/plugin.json    # Codex manifest
.cursor-plugin/plugin.json   # Cursor manifest
skills/                      # (to be added) reusable agent skills
agents/                      # (to be added) named agent personas
commands/                    # (to be added) slash commands
```

## Roadmap

- Wrap common Recoup chat / API interactions as skills.
- Add commands for triggering Recoup platform tasks from an agent
  session.
- Add hooks for platform-aware safety and validation.

## Support

- Email: `support@recoupable.com`
- Website: <https://recoupable.com>

## License

[Apache-2.0](./LICENSE)
