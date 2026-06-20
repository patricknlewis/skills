# Patrick Lewis — Claude Skills

A small marketplace of [Claude Code](https://code.claude.com) skills I use in my design and product work. Install any of them in two commands.

## Install

In Claude Code, add the marketplace once:

```
/plugin marketplace add patricknlewis/skills
```

Then install the skill you want:

```
/plugin install extract-figma-design-md
```

That's it. Updates arrive automatically when I push new versions — or run `/plugin marketplace update` to refresh on demand.

## Skills

### extract-figma-design-md

Turns any Figma design system into a spec-compliant [`DESIGN.md`](https://github.com/google-labs-code/design.md) — the plain-text, AI-readable source of truth for a design system. It pulls colors, typography, spacing, radii, and every component (with variants) out of Figma via the Figma MCP, interviews you about the decisions Figma can't export (brand voice, semantic color meaning, do's and don'ts, light/dark modes), and writes a clean DESIGN.md you can hand to any AI agent so its prototypes match your system.

Built so a non-designer can prototype against your design system and get output that actually looks right.

**Requires:** a connected [Figma MCP server](https://www.figma.com/blog/introducing-figmas-dev-mode-mcp-server/) (remote `mcp.figma.com` or the Figma desktop app's Dev Mode MCP).

**Run it by saying:** *"Make a DESIGN.md from my Figma file: <url>"* — or just mention design.md and paste a Figma link.

### write-project-summary

Writes a portfolio *project summary* — the single-page writeup that documents one product, sitting between a quick blurb and a full case study (500–900 words). It has two modes: **interview mode** generates a short set of questions when the source material is thin (answer by voice memo, transcript, or text), and **drafting mode** writes the summary once the material exists. It enforces structure (opening, features, strategic decisions, learnings) and a fabrication guard so nothing — metrics, learnings, roles — gets invented from inference.

Built so the summary shows *how a designer thinks*, with the product as evidence, not a press release.

**Run it by saying:** *"Write up [product] for my portfolio"* — or *"draft the [product] page"* / *"help me document this project."*

### critique-from-a-boring-designer

Gives a structured design critique the way a thoughtful senior designer would — not a generic heuristics audit. It runs a short intake (goal, stage, known-rough areas), then works deliberate passes (does it work → is it complete → does it say the right things → is it considered), gating which passes run by design stage so an exploration isn't critiqued like a shipping candidate. Notes name consequences over preferences, are tiered by severity, and capped to the few that matter. Built on Cap Watkins' *The Boring Designer*. Its one hard rule: it **never proposes an alternative design** — it names the problem and leaves the solving to you.

**Run it by saying:** *"Critique this"* / *"review my design"* — and share a screen, a flow, or a flow plus a reference screen.

### name-a-product

Helps name a product the way a designer who's named things well actually does it — start from the job the product does, find a real-world thing that does that job, and check what people *hear* when they say it out loud. It takes a short brief, generates names across two paths (job-to-counterpart and visual-first), tests each against a weighted set of standards (does it do work, does it carry a visual, the wrong-image check, owned-term collisions), and narrows to a shortlist of three to five with the tradeoffs named. Availability is treated as a filter on the shortlist, not a gate at the top. Its hard rules: it **never crowns a winner** and **never draws a logo or mark** — it surfaces the raw material and hands the creative judgment back to you.

**Run it by saying:** *"Help me name [product]"* / *"what should I call this"* / *"this name isn't landing."*

## Notes

- This repo is itself the marketplace; no install step beyond the two commands above.
- Skills install under your `~/.claude/` directory and are namespaced (e.g. `extract-figma-design-md:extract-figma-design-md`) to avoid collisions.

## License

MIT
