# Figma MCP extraction guide

How to pull a design system out of Figma via the Figma MCP server. Everything here was verified against real MCP responses; the quirks are real behaviors, not speculation.

## Which server variant is connected?

There are two common Figma MCP variants. Check the tool schemas to tell them apart:

| Variant | How to recognize | How to target nodes |
|---|---|---|
| **Remote** (mcp.figma.com) | Tools require `fileKey` and `nodeId` parameters | Extract from URL: `figma.com/design/:fileKey/:name?node-id=55-1219` → fileKey `n84UIty...`, nodeId `55:1219` (convert `-` to `:`) |
| **Desktop** (Dev Mode MCP in Figma app) | Tools take no parameters, or only optional ones | Operates on the user's current selection/open file; ask the user to open the file and select nodes |

The workflow is the same for both; with the desktop variant you ask the user to navigate instead of passing IDs.

## The tools, cheapest first

| Tool | Cost | Returns | Use for |
|---|---|---|---|
| `get_metadata` | Cheap | XML: node IDs, types, names, positions, sizes | File structure, component inventory, swatch/specimen labels |
| `get_variable_defs` | Cheap | JSON map of variable name → value | Token values (colors, type, spacing, effects) |
| `get_screenshot` | Medium | PNG (set `maxDimension` ~512 to save context) | Describing a component's look in prose |
| `get_design_context` | **Expensive** | React+Tailwind reference code | Exact padding/radius/layout/state styling when variables don't reveal it. Set `excludeScreenshot: true` if you've already seen the component |

## Known quirks (important)

1. **Page enumeration is unreliable.** Calling `get_metadata` with no `nodeId` is documented to list top-level pages, but it can return a stale, incomplete list (observed: 1 page listed when 25 exist). Workarounds, in order:
   - Call `get_metadata` with a deliberately invalid nodeId (e.g. `0:0`). The error message includes the *full, accurate* page list ("Top-level pages of the document: …"). Hacky but verified to work; harmless if it stops working.
   - Use the node-id from the user's URL to confirm at least that page exists, and ask the user to name the rest ("right-click a page → Copy link to page").

2. **`get_variable_defs` errors on pages/canvases** ("You currently have nothing selected"). Always pass a concrete node (a component symbol, frame, etc.), never a page ID.

3. **`get_variable_defs` returns only the variables *used by* that node** — not the file's full variable collection. To reconstruct the full token set, harvest from many components and merge. Components collectively touch nearly every token; foundation specimen frames fill the rest.

4. **`search_design_system` only sees published libraries.** Unpublished design system files return empty results. Don't conclude "no tokens" from an empty search.

5. **`get_design_context` may return only metadata for large nodes** (size guard). That's usually fine — prefer extracting from smaller representative components anyway.

6. **Some variables return empty values.** Gradient variables (e.g. `Gradient/Maintenance`) come back as `""` — gradients are invisible to `get_variable_defs`. Note where they're used, describe them from a screenshot if possible, and flag the exact stops as an interview/TO CONFIRM item.

7. **Trust wired variables over specimen labels.** When a foundation swatch label disagrees with the variable a component actually uses (observed: swatch said `#EEFFB8`, the wired `Lime/300` variable resolved `#F2FFCA`), the variable wins — components render from variables; posters go stale. Record the discrepancy and flag it for the user.

8. **Watch for mis-bound variables.** A radius bound to a *spacing* variable (e.g. corner radius wired to `Space/4` instead of `Radius/Large`) is a Figma hygiene issue, not a design intent. Tokenize by intent (`rounded.large`), note the quirk in prose, and mention it to the user — they'll usually want to fix the Figma file.

9. **Empty component pages exist.** A page may be a named placeholder with no children. Document the component as "not yet built" rather than silently skipping it.

10. **Variable modes (light/dark) can't be listed, but frame overrides ARE respected.** `get_variable_defs` resolves variables in the mode of the node you query — which is the file default unless the designer set an explicit mode override on that frame. There is no API to enumerate modes or query "give me mode X", so always ask the user whether the variables panel has more than one mode column. If a second mode exists, harvest its values (verified workflow):
    1. Ask the user to set the variable-mode override to the other mode on a handful of token-rich components in Figma (select frame → Appearance section → mode switcher). Choose components that collectively touch all color tokens: a content card (halftones), a status card (semantic colors), a tag (accent surface pairs), an input (borders, tertiary text), and a sheet/page surface (primary background).
    2. Re-run `get_variable_defs` on those node IDs — same variable names now resolve to the other mode's values.
    3. Diff against the default-mode pool; tokens never exercised by the flipped components (e.g. warning colors) remain unknown — infer from the system's inversion pattern and mark as inferred, or ask.
    Fallbacks: the user screenshots the variables panel, or pastes values.

## Observed response formats and how to parse them

### get_variable_defs

```json
{
  "Foreground/Primary": "#0a0a0a",
  "Halftone/50": "#0a0a0a80",
  "Space/4": "16",
  "Body/Default": "Font(family: \"Mona Sans\", style: Regular, size: 17, weight: 400, lineHeight: 1.2999999523162842, letterSpacing: 0)",
  "glassCard": "Effect(type: GLASS, radius: 20); Effect(type: INNER_SHADOW, color: #FFFFFF1A, offset: (4, 4), radius: 4, spread: 0)"
}
```

Parsing rules:

- **Colors**: 6-digit hex, or 8-digit hex where the last two digits are alpha (`#0a0a0a80` = 50% black). 8-digit hex is valid in DESIGN.md; keep it.
- **Numbers as strings** (`"16"`): pixel values. Emit as `16px` in spacing/dimension tokens.
- **`Font(...)` strings**: parse fields into the spec's Typography shape — `family` → `fontFamily`, `size: 17` → `fontSize: 17px`, `weight` → `fontWeight`, `lineHeight: 1.2999...` → unitless multiplier, round to 2 decimals (`1.3`), `letterSpacing: 0` → omit or `0em`. The `style` field (e.g. "Regular", "Condensed Medium") may carry width/stretch info that weight alone doesn't — note condensed/expanded styles in the typography prose.
- **`Effect(...)` strings**: not representable as spec tokens. Transcribe precisely into the Elevation & Depth prose, e.g. "cards use a glass-blur effect (radius 20) with a subtle inner highlight (`#FFFFFF1A`, offset 4,4, blur 4)".

### get_metadata (XML)

- `<symbol>` nodes are components. Their position on the page often groups variants together.
- Variant naming: either Figma variant syntax (`State=Pressed, Size=Large`) or separate sibling symbols (`Meal Card`, `Dinner Budget Card`). Both mean "variants" for DESIGN.md purposes.
- **Foundation frames encode values in layer names and text nodes.** Observed patterns worth mining:
  - Color swatches: frames containing text pairs like `Gray 900` / `#0A0A0A` — gives human names *and* hex for the full ramp, including shades no component uses yet.
  - Type specimens: text layers named like `Headline DEFAULT - Mona Sans CONDENSED MEDIUM, 28 pt` — name, face, style, size in one string.
  - Loose text nodes on foundation pages may contain the designer's own to-do notes — these often reveal intent (e.g. semantic color meanings) worth asking about in the interview.
- Cross-check: foundation frames give the *full designed palette*; variables give the *tokens actually wired up*. Both belong in DESIGN.md — full ramps as color tokens, with prose noting the workhorse ones.

### get_design_context (React+Tailwind)

Mine the generated code for values, ignore the code itself:

- `var(--space\/4,16px)` → the node uses variable `Space/4` = 16px. CSS var fallbacks confirm resolved values.
- `p-[...]`, `gap-[...]`, `rounded-[...]` classes → exact padding/gap/radius for the component's token entry.
- The trailing "These styles are contained in the design: …" block is a free `get_variable_defs` for that node — harvest it instead of making a second call.
- Hardcoded values (e.g. `rounded-[20px]` with no var) are un-tokenized design decisions — record the literal and flag it as a candidate interview question ("cards use 20px radius but there's no radius token — intentional?").

## Aggregation strategy

1. Page inventory → classify foundation vs component pages.
2. `get_metadata` per component page → component + variant inventory (names, counts, sizes).
3. `get_variable_defs` on 1 symbol per component (more if variants differ visually) → merged token pool.
4. `get_metadata` on foundation frames → full color ramp + type scale with human names.
5. Reconcile: variables ↔ foundation labels (e.g. variable `Foreground/Primary` = `#0a0a0a` = swatch "Gray 900"). Record the correspondence in prose.
6. `get_design_context` only where needed: one representative component per *family* (one card, one button, one input) to capture padding/radius patterns, plus any component whose construction is unclear.
7. For >10 component pages: if subagents are available, batch pages across them, each returning per component: `name, variants[], tokens_used{}, padding, radius, one_line_description`. Without subagents, inline extraction works fine (verified on a 25-page system in ~30 tool calls) — just stay disciplined about cheap tools and batch independent calls in parallel where the client allows it.

## Mapping to DESIGN.md frontmatter

| Figma | DESIGN.md | Example |
|---|---|---|
| Color variable `Foreground/Primary` | `colors.foreground-primary` | `"#0a0a0a"` |
| Opacity ramp `Halftone/50` | `colors.halftone-50` | `"#0a0a0a80"` |
| Swatch-only color `Lime 600` | `colors.lime-600` | `"#B8F402"` |
| Float/text style `Body/Default` | `typography.body-default` | Typography object |
| Number variable `Space/4` | `spacing.space-4` | `16px` |
| Corner radius (from design context) | `rounded.*` | `rounded.card: 16px` |
| Component symbol `Meal Card` | `components.meal-card` | property map with `{references}` |
| Effect style | Elevation & Depth prose | — |

Naming: lowercase, `/` and spaces → `-`. Never rename the system's vocabulary to generic terms — if the system says "halftone", DESIGN.md says halftone (the spec accepts any descriptive key). Do ensure a `primary` color exists, aliasing if needed (`primary: "#0a0a0a"` alongside `foreground-primary`) since the spec requires it.
