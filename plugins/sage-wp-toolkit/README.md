# Sage WP Toolkit

Claude Code plugin for Sage 11 (Blade + Tailwind v4 + Vite + Acorn) WordPress theme projects.

## What's inside

**Skills** (auto-invoked by Claude when the task matches — no need to call them by name):
- `frontend-code-rules` — mandatory HTML/CSS/JS conventions (tokens, breakpoints, markup structure, a11y, perf). Triggers on any HTML/CSS/JS write or edit.
- `wordpress-sage-theme` — Sage/Blade/ACF patterns, theme scaffolding, template hierarchy.
- `wp-block-development` — Gutenberg block internals: attributes, deprecations, inner blocks, dynamic rendering.
- `wp-performance` — diagnostics for autoload options, cron, DB queries, object cache, `wp-cli` profiling.

**Commands** (invoke explicitly):
- `/sage-wp-toolkit:make-acorn-block` — scaffold a dynamic Gutenberg block from raw HTML + notes.
- `/sage-wp-toolkit:figma-to-block` — scaffold a block from a Figma URL via Figma MCP.
- `/sage-wp-toolkit:add-branding-section` — add a new settings section to a branding admin page, following the standard 5-layer pattern.

## Install (per project)

```
/plugin marketplace add https://github.com/crea8ivedev/sage-gutenberg-block-wp-toolkit.git
/plugin install sage-wp-toolkit@sage-gutenberg-block-wp-toolkit
```

## Per-project setup

This plugin ships reusable skills/commands only — it does **not** carry a CLAUDE.md (plugins can't auto-load project-wide guidance). Copy `templates/CLAUDE.md.template` from the marketplace repo into your new project's root as `CLAUDE.md` and fill in the placeholders (`<CLIENT_NAME>`, `<THEME_FOLDER>`, `<BLOCK_NAMESPACE>`, etc.).
