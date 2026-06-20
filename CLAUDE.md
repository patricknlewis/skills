# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This repo **is** a Claude Code plugin marketplace. There is no application code, build step, test suite, or dependencies — every "plugin" is a directory of Markdown (a `SKILL.md` plus optional reference docs) that Claude Code loads as an agent skill. Work here is authoring and maintaining skills, not writing software.

Users install from it with:
```
/plugin marketplace add patricknlewis/skills
/plugin install <plugin-name>
```

## Structure & wiring

Three files must stay in sync for a plugin to install and load correctly:

1. `.claude-plugin/marketplace.json` — the marketplace manifest. Every plugin needs an entry here (`name`, `source` path, `description`, `version`, `author`). A plugin missing from this list is invisible to the marketplace even if its directory exists.
2. `plugins/<name>/.claude-plugin/plugin.json` — the per-plugin manifest (`name`, `version`, `description`, `author`, `homepage`, `repository`, `keywords`, `license`). The `name` and `version` here must match the marketplace entry. `repository` is a plain string URL, not an object.
3. `plugins/<name>/skills/<skill-name>/SKILL.md` — the skill itself. By convention the plugin name, the skills subdirectory name, and the SKILL.md `name:` frontmatter field are all identical.

A skill is invoked at runtime as `<plugin-name>:<skill-name>` (namespaced to avoid collisions).

## The SKILL.md contract

`SKILL.md` opens with YAML frontmatter whose `description` is the **routing logic** — Claude reads it to decide whether to trigger the skill. Descriptions here are deliberately long and list concrete trigger phrases ("critique this," "write up [product] for my portfolio," paste a Figma URL). When editing a skill's behavior, keep the description's triggers aligned with what the body actually does. Some skills also carry a `compatibility:` field declaring requirements (e.g. a connected Figma MCP server).

Skills may ship reference docs under `references/` (see `extract-figma-design-md`). The skill body instructs Claude to read those before acting; they hold format specs and tool-usage quirks kept out of the main flow.

Each skill encodes an opinionated process with explicit guardrails — e.g. `write-project-summary`'s fabrication guard (never invent metrics/roles/learnings) and `critique-from-a-boring-designer`'s hard rule (never propose an alternative design). Preserve these constraints when editing; they are the point of the skill, not incidental.

## Adding a plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json` and `plugins/<name>/skills/<name>/SKILL.md`.
2. Add the matching entry to `.claude-plugin/marketplace.json`.
3. Add a `### <name>` section to `README.md` describing what it does and how to invoke it.

## Conventions

- Keep `README.md`, the marketplace manifest, and each plugin manifest descriptive of the *same* behavior — they are the three public-facing descriptions of every skill.
- Prose style in skills uses an em-dash-heavy, explanatory voice; match it when editing.
- License is MIT across the board.
