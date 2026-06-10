---
name: figma-to-design-md
description: Generate or update a DESIGN.md design-system spec file from a Figma design system file using the Figma MCP. Use this skill whenever the user mentions design.md or DESIGN.md, wants to extract design tokens, variables, or components from Figma into a document, wants to document a design system, or wants prototypes and generated code to match their Figma design system — even if they just paste a Figma URL and ask to "set up design context", "capture my design system", or "make my prototypes look right". Also use it to refresh an existing DESIGN.md after the Figma file changes.
compatibility: Requires a connected Figma MCP server (remote mcp.figma.com or the Figma desktop app's Dev Mode MCP server).
---

# design-md: Figma design system → DESIGN.md

This skill produces a `DESIGN.md` file — a self-contained, plain-text representation of a design system — extracted from a Figma design system file and enriched by interviewing the user about their design decisions.

**Why this file matters:** DESIGN.md will be consumed by AI agents (and humans) that have *no access to Figma*. Whatever doesn't make it into this file is lost to them. The YAML frontmatter tokens are the normative values; the prose carries the intent and judgment that tokens can't express. The end user is often non-technical — the file must let them prototype in Claude and get output that matches the design system as closely as possible, without them knowing what a design token is.

Before doing anything else, read both reference files:

1. `references/design-md-spec.md` — the DESIGN.md format spec (section order, YAML schema, token rules). The output must conform to it exactly.
2. `references/figma-extraction.md` — how to use the Figma MCP tools, including known quirks, observed response formats, and parsing rules. Reading this first will save many failed tool calls.

## Workflow

### Step 1 — Locate the source file

- If the user's message contains a `figma.com/design/...` URL, extract `fileKey` and (if present) `node-id` from it.
- If there's no URL and the connected Figma MCP is the desktop variant (tools take no fileKey parameter and operate on the current selection), ask the user to open the design system file in the Figma desktop app.
- Otherwise, ask the user for a link to their design system file. Any link into the file works — you can enumerate pages from the fileKey alone.

### Step 2 — Map the file

Enumerate the file's pages and classify them (see the extraction reference for the page-listing quirks — the naive listing call is often incomplete):

- **Foundation content**: pages or frames named like Colors, Typography, Type, Spacing, Tokens, Foundations, Styles, Icons. Often several foundation frames live on a single page.
- **Component pages**: pages named after components (Button, Cards, Modal, Nav Bar, …). Many design systems use one page per component.

Show the user the map you found ("I see 24 component pages and foundations for color/type/icons — does that cover everything?") before doing the heavy extraction. This catches missing pages early and costs one message.

### Step 3 — Extract

Follow `references/figma-extraction.md` for tool usage and parsing. In brief:

- **Foundations**: `get_metadata` on foundation frames — swatch labels and type specimens usually encode names and values in the layer names/text. Cross-check against variables.
- **Variables**: `get_variable_defs` on concrete component nodes (never on a page/canvas — it errors). Variables are returned per-node as *only the variables that node uses*, so harvest across many components and merge.
- **Components**: per component page, `get_metadata` to list the component symbols and their variants, then `get_variable_defs` on each symbol. Use `get_design_context` selectively for representative components where you need exact padding, radius, or state styling that variables don't reveal. Use `get_screenshot` (small `maxDimension`, ~512) when you need to *describe* a component's look in prose.
- **Budget your context.** A 25-page system can easily blow the context window if you call `get_design_context` on everything. Metadata + variables are cheap; design context is expensive. If the file has more than ~10 component pages and subagents are available, fan extraction out: one subagent per batch of pages, each reporting back a compact structured summary (component name, variants, tokens used, key dimensions, one-line visual description).

### Step 4 — Draft the tokens

Build the YAML frontmatter per the spec schema. Key rules (details and examples in the extraction reference):

- Normalize Figma variable names to reference-safe token names: lowercase, `/` → `-` (`Space/4` → `space-4`, `Foreground/Primary` → `foreground-primary`). Keep the design system's own vocabulary — don't force-rename to `primary/secondary` if the system says `foreground/halftone`; the spec allows any descriptive keys.
- Unitless Figma numbers are px (`"16"` → `16px`). Round float line-heights to 2 decimals.
- In `components`, use `{token.references}` wherever a value matches a defined token; literals only for one-off values. Define variants as sibling keys (`button-primary`, `button-primary-pressed`).
- Effects (shadows, blurs, glass) don't fit the token schema — define each as a named recipe in the Elevation & Depth prose, precisely (color, offset, blur, spread). When a component's identity depends on an effect, add a custom `effect: <recipe-name>` property to its component entry pointing at that recipe — the spec accepts unknown component properties, and the named link keeps tokens and prose connected.
- Asymmetric padding doesn't fit the spec's single `padding` Dimension. Use custom `paddingX`/`paddingY` (and `gap`) properties — spec-tolerated, unambiguous. Reserve `padding` for uniform padding.

Validate as you go: every reference resolves, every color is valid hex, every dimension has a unit, `colors.primary` (or the system's equivalent) exists.

### Step 5 — Interview the user

This is where the file gains the judgment that Figma can't export. Conduct it in two passes, a few questions at a time — never one giant questionnaire. If an interactive question tool (e.g. AskUserQuestion) is available, use it; offer your best inference from the file as the first option so the user can confirm rather than compose.

**Core set** (always ask, adapted to what you found):

1. **Brand personality & audience** — three adjectives for how the product should feel; who it's for; what emotional response the UI should evoke. (Feeds Overview.)
2. **Color roles & meaning** — what each palette is *for*. Probe semantic colors specifically: "Lime appears in progress components and orange in deficit charts — what do these mean in your product?" (Feeds Colors.)
3. **Typography strategy** — why these typefaces/weights; when each level is used; any rules like "condensed only for display". (Feeds Typography.)
4. **Layout & spacing philosophy** — base unit, density, grids vs. safe-area margins; for iOS apps ask about safe areas and tab/nav conventions. (Feeds Layout.)
5. **Elevation & shape** — how hierarchy is conveyed (shadows, tonal layers, borders, glass); what the corner-radius language means. (Feeds Elevation & Depth + Shapes.)
6. **Do's and don'ts** — the rules they'd shout at a designer breaking them; accessibility commitments; pet peeves. (Feeds Do's and Don'ts.)

**Color modes** (always ask): the Figma MCP silently resolves the default variable mode and offers no way to list modes, so you must ask: "Do your color variables define more than one mode (e.g. light and dark)? You'd see them as columns in Figma's variables panel." If yes: ask which mode should be the *default* for this product, extract the other mode's values via the frame-override flow in the extraction reference (quirk 10), make the default mode's values the normative color tokens, and document the other mode in a `## Color Modes` section placed after Do's and Don'ts (extra sections are spec-safe) with a token-by-token mapping table, the inversion pattern in prose, and an explicit "render in <default> unless asked" rule (also add it to Do's and Don'ts).

**Dynamic follow-ups** (generate from your extraction findings — this is the high-value part):

- Multiple variants of a component → "When should each be used?"
- A custom effect or unusual token (e.g. an opacity-based `halftone` ramp, a glass effect) → "What's the thinking behind this?"
- Gaps → tokens with no apparent use, components with no obvious purpose, missing states (no pressed/disabled variants).
- Ambiguity between similar components (Sheet vs. Modal, Tag vs. Chip) → "How do you decide which to use?"

**Draft mode:** if the user is unavailable or says "just generate it", don't stall. Infer answers from visual evidence (screenshots, naming, component usage) and write the prose anyway, marking inferred-but-unconfirmed claims inline with `<!-- TO CONFIRM: ... -->` comments. Tell the user which items to review.

### Step 6 — Compose DESIGN.md

Write the file per the spec: YAML frontmatter first, then the sections **in spec order** (Overview, Colors, Typography, Layout, Elevation & Depth, Shapes, Components, Do's and Don'ts), all as `##` headings, no duplicates. Weave the interview answers into the prose — the user's reasoning, in clean editorial prose, not Q&A transcripts.

For the **Components** section, give each component its own `###` subsection with: what it is, its variants, when to use it (from the interview), and pointers to its tokens. An agent reading only this section should be able to build a passable version of the component.

Coverage discipline: before writing, list every component from your Step 2 inventory, and as you write tick off two boxes per component — token entry *and* prose subsection. It is easy to drop one component while composing a long file (in testing, a 24-component system lost exactly one subsection this way). Components that exist in Figma but couldn't be extracted (empty pages, broken nodes) still get a subsection noting that.

Prose may use the human color names from Figma ("Gray 900") alongside token names — the tokens are normative, the prose explains.

**Always include a base-text-color rule.** The spec's typography tokens define face/size/weight but carry no color, so a consuming agent that applies a typography token without pairing it with a foreground token falls back to the platform default — black. On a light canvas this fails silently; on a dark-default system it produces invisible text (this exact bug shipped from a generated DESIGN.md once: unstyled headlines vanished into a near-black background). Write an explicit rule in both Typography and Do's and Don'ts: "set the system's primary foreground color as the inherited base text color on the root container; override locally with other foreground tokens." For dark-default systems, call out the failure mode by name.

### Step 7 — Validate and deliver

Run this checklist before saving:

- [ ] Frontmatter parses as YAML; opens and closes with `---`
- [ ] Section order matches the spec; no duplicate `##` headings (a duplicate makes consumers reject the file)
- [ ] Every `{reference}` resolves to a defined token
- [ ] All colors are `#`-prefixed hex; all dimensions have px/em/rem units (or are intentionally unitless numbers like line-height)
- [ ] Every component found in Figma appears in both `components` tokens and the Components prose — check this programmatically (script the cross-check of page inventory vs. `###` headings vs. token keys), not by eye
- [ ] No `<!-- TO CONFIRM -->` markers remain unless the user chose draft mode
- [ ] The file states a base-text-color rule (root inherits the primary foreground token) — mandatory for dark-default systems

Save as `DESIGN.md` at the project root (ask if there's an obvious alternative like a `docs/` convention). Summarize what was extracted (n colors, n type levels, n components), list anything you couldn't extract, and offer a quick review pass: render 2–3 component descriptions back to the user and ask if they match reality.

## Updating an existing DESIGN.md

If a DESIGN.md already exists, read it first. Re-extract from Figma, then *diff*: report what changed (new components, changed values, removed tokens) and ask before overwriting prose the user wrote by hand. Preserve their voice in unchanged sections; only re-interview about the new or changed parts.
