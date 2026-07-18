# Sage Gutenberg Block WP Toolkit — Claude Code Marketplace

Internal marketplace for shared Claude Code config across Encircle Technologies WordPress projects.

## What you get

- **Front-end rules** (Tailwind tokens, markup structure, a11y, perf) — kicks in automatically, nothing to type
- **Sage/Blade/ACF patterns** + Gutenberg block dev know-how
- **WP performance diagnostics** (autoload, cron, DB, object cache)
- **Figma MCP server** bundled — auto-wired on install, each user logs in with their own Figma account
- **Slash commands** to scaffold blocks fast:
  - `/sage-wp-toolkit:make-acorn-block` — dynamic Gutenberg block from raw HTML
  - `/sage-wp-toolkit:figma-to-block` — pixel-perfect block from a Figma URL (needs Figma MCP connected)
  - `/sage-wp-toolkit:add-branding-section` — new settings section on the branding admin page

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

## Contributors

- [Nimesh Gorfad](https://github.com/nimeshgorfad)

## Not included here

`.claude/settings.local.json` (permission allowlist) is intentionally excluded — it's local/session
config, not shareable, and the source project's copy had a hardcoded Figma API token that needs
rotating. Never commit personal access tokens into shared config; use Figma's MCP auth flow instead.
