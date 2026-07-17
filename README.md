# Sage Gutenberg Block WP Toolkit — Claude Code Marketplace

Internal marketplace for shared Claude Code config across Encircle Technologies WordPress projects.

## Structure

```
sage-gutenberg-block-wp-toolkit/
├── .claude-plugin/marketplace.json   # lists all plugins in this marketplace
├── plugins/
│   └── sage-wp-toolkit/              # skills + commands for Sage theme projects
└── templates/
    └── CLAUDE.md.template            # per-project CLAUDE.md starting point (not a plugin — copy manually)
```

## One-time setup (per team member)

```
/plugin marketplace add https://github.com/crea8ivedev/sage-gutenberg-block-wp-toolkit.git
```

## Per-project setup

```
/plugin install sage-wp-toolkit@sage-gutenberg-block-wp-toolkit
```

Then copy `templates/CLAUDE.md.template` into the new project's root as `CLAUDE.md` and fill in the `<PLACEHOLDER>` values.

## Updating

When the toolkit changes (new rule, new command), bump the version in
`plugins/sage-wp-toolkit/.claude-plugin/plugin.json`, commit, push. Team members run:

```
/plugin marketplace update sage-gutenberg-block-wp-toolkit
```

## Adding a new plugin later

Add a new folder under `plugins/`, give it its own `.claude-plugin/plugin.json`, then add an entry to
`.claude-plugin/marketplace.json`'s `plugins` array pointing `source` at `./plugins/<name>`.

## Not included here

`.claude/settings.local.json` (permission allowlist) is intentionally excluded — it's local/session
config, not shareable, and the source project's copy had a hardcoded Figma API token that needs
rotating. Never commit personal access tokens into shared config; use Figma's MCP auth flow instead.
