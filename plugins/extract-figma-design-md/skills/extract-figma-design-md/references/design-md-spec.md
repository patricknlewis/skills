# DESIGN.md Format Specification

Source: https://github.com/google-labs-code/design.md (docs/spec.md, version: alpha). Bundled here so the skill works offline; if the user asks for the latest spec, fetch the source.

DESIGN.md is a self-contained, plain-text representation of a design system. It defines the visual identity of a brand and product, ensuring stylistic choices can be followed across design sessions and between different AI agents and tools. As a human-readable, open-format document, it serves as a living source of truth that both humans and AI can understand and refine.

A DESIGN.md file contains two parts: an optional YAML frontmatter, and a markdown body. The YAML frontmatter contains machine-readable design tokens. The markdown body sections provide human-readable design rationale and guidance. Prose may use descriptive color names (e.g., "Midnight Forest Green") that correspond to systematic token names (e.g., `primary`). **The tokens are the normative values; the prose provides context for how to apply them.**

## Design Tokens (YAML frontmatter)

Inspired by the Design Token JSON spec: typed token groups (colors, typography, spacing) and `{path.to.token}` reference syntax. Easily converted from/to `tokens.json`, Figma variables, and Tailwind theme configs.

The frontmatter block must begin and end with a line containing exactly `---`.

Example:

```yaml
---
version: alpha
name: Daylight Prestige
colors:
  primary: "#1A1C1E"
  secondary: "#6C7278"
  tertiary: "#B8422E"
typography:
  h1:
    fontFamily: Public Sans
    fontSize: 48px
    fontWeight: 600
    lineHeight: 1.1
    letterSpacing: -0.02em
---
```

### Schema

```yaml
version: <string>          # optional, current version: "alpha"
name: <string>
description: <string>      # optional
colors:
  <token-name>: <Color>
typography:
  <token-name>: <Typography>
rounded:
  <scale-level>: <Dimension>
spacing:
  <scale-level>: <Dimension | number>
components:
  <component-name>:
    <token-name>: <string|token reference>
```

`<scale-level>` is a named level in a sizing/spacing scale (`xs`, `sm`, `md`, `lg`, `xl`, `full`, …). Any descriptive string key is valid.

**Color**: must start with `#` followed by a hex color code in the SRGB color space.

**Typography** properties:

- `fontFamily` (string)
- `fontSize` (Dimension)
- `fontWeight` (number) — e.g. `400`, `700`; bare number or quoted string both valid
- `lineHeight` (Dimension | number) — unitless number = multiplier of fontSize (recommended)
- `letterSpacing` (Dimension)
- `fontFeature` (string) — CSS `font-feature-settings`
- `fontVariation` (string) — CSS `font-variation-settings`

**Dimension**: string with unit suffix. Valid units: `px`, `em`, `rem`.

**Token references**: wrapped in curly braces, containing an object path to another value in the YAML tree (e.g. `{colors.primary-60}`). Outside `components`, references must point to a primitive value, not a group. Within `components`, references to composite values (e.g. `{typography.label-md}`) are permitted.

## Sections

Every DESIGN.md follows the same structure. Sections can be omitted if not relevant, but those present must appear in this order, all as `##` headings. An optional `#` h1 title may appear but is not parsed as a section.

1. **Overview** (also: "Brand & Style")
2. **Colors**
3. **Typography**
4. **Layout** (also: "Layout & Spacing")
5. **Elevation & Depth** (also: "Elevation")
6. **Shapes**
7. **Components**
8. **Do's and Don'ts**

### Overview

Holistic description of the product's look and feel: brand personality, target audience, the emotional response the UI should evoke (playful or professional, dense or spacious). Foundational context guiding the agent's high-level stylistic decisions when a specific rule or token isn't defined.

### Colors

Defines the color palettes. At least the `primary` color palette must be defined. With multiple palettes, assign a semantic role to each; common convention: `primary`, `secondary`, `tertiary`, `neutral`.

Example prose:

```markdown
## Colors

The palette is rooted in high-contrast neutrals and a single, evocative accent color.

- **Primary (#1A1C1E):** A deep ink used for headlines and core text to provide
  maximum readability and a sense of permanence.
- **Secondary (#6C7278):** A sophisticated slate used primarily for utilitarian
  elements like borders, captions, and metadata.
- **Tertiary (#B8422E):** A vibrant earthy red as the sole driver for
  interaction, used exclusively for primary actions and critical highlights.
- **Neutral (#F7F5F2):** A warm limestone that serves as the foundation for all
  pages, providing a softer, more organic feel than pure white.
```

The `colors` token group is a map<string, Color>, derived from the palettes in the prose. Any consistent naming convention is fine.

### Typography

Defines typography levels. Most design systems have 9–15 levels. Common naming: semantic categories (`headline`, `display`, `body`, `label`, `caption`) divided into sizes (`small`, `medium`, `large`).

The `typography` token group is a map<string, Typography>:

```yaml
typography:
  h1:
    fontFamily: Public Sans
    fontSize: 48px
    fontWeight: 600
    lineHeight: 1.1
    letterSpacing: -0.02em
  body-md:
    fontFamily: Public Sans
    fontSize: 16px
    fontWeight: 400
    lineHeight: 1.6
  label-caps:
    fontFamily: Space Grotesk
    fontSize: 12px
    fontWeight: 500
    lineHeight: 1
    letterSpacing: 0.1em
```

### Layout

Describes the layout and spacing strategy. Many design systems use a grid-based layout; others (e.g. Liquid Glass) use margins, safe areas, and dynamic padding.

Example prose: fluid grid on mobile, fixed-max-width grid on desktop (max 1200px); strict 8px spacing scale with a 4px half-step; containment via cards with generous internal padding.

The `spacing` token group is a map<string, Dimension | number> (unitless numbers allowed for column counts or ratios):

```yaml
spacing:
  base: 16px
  xs: 4px
  sm: 8px
  md: 16px
  lg: 32px
  xl: 64px
  gutter: 24px
  margin: 32px
```

### Elevation & Depth

How visual hierarchy is conveyed. If elevation is used, define the required styling (spread, blur, color). For flat designs, explain the alternative (borders, color contrast, tonal layers).

### Shapes

How visual elements are shaped (corner radii, sharpness vs. softness, what the shape language communicates).

The `rounded` token group is a map<string, Dimension>:

```yaml
rounded:
  sm: 4px
  md: 8px
  lg: 12px
  full: 9999px
```

### Components

Style guidance for component atoms. Common types: Buttons (variants, sizing, padding, states), Chips, Lists, Tooltips, Checkboxes, Radio buttons, Input fields (labels, helper text, error states). Design systems are encouraged to define additional components relevant to their domain.

The `components` token group is a map<string, map<string, string>> mapping a component identifier to sub-token names and values. Values may be literals or references to previously defined tokens.

**Variants**: a component may have variants for UI states (active, hover, pressed) defined under related keys: `button-primary`, `button-primary-hover`, `button-primary-active`. The agent considers all variants when styling.

```yaml
components:
  button-primary:
    backgroundColor: "{colors.primary-60}"
    textColor: "{colors.primary-20}"
    rounded: "{rounded.md}"
    padding: 12px
  button-primary-hover:
    backgroundColor: "{colors.primary-70}"
```

Component property tokens: `backgroundColor` (Color), `textColor` (Color), `typography` (Typography), `rounded` (Dimension), `padding` (Dimension), `size` (Dimension), `height` (Dimension), `width` (Dimension).

### Do's and Don'ts

Practical guidelines and common pitfalls — guardrails for design generation:

```markdown
## Do's and Don'ts

- Do use the primary color only for the single most important action per screen
- Don't mix rounded and sharp corners in the same view
- Do maintain WCAG AA contrast ratios (4.5:1 for normal text)
- Don't use more than two font weights on a single screen
```

## Recommended token names (non-normative)

- **Colors:** `primary`, `secondary`, `tertiary`, `neutral`, `surface`, `on-surface`, `error`
- **Typography:** `headline-display`, `headline-lg`, `headline-md`, `body-lg`, `body-md`, `body-sm`, `label-lg`, `label-md`, `label-sm`
- **Rounded:** `none`, `sm`, `md`, `lg`, `xl`, `full`

## Consumer behavior for unknown content

| Scenario | Behavior | Example |
|---|---|---|
| Unknown section heading | Preserve; do not error | `## Iconography` |
| Unknown color token name | Accept if value is valid | `surface-container-high: '#ede7dd'` |
| Unknown typography token name | Accept as valid typography | `telemetry-data` |
| Unknown spacing value | Accept; store as string if not a valid dimension | `grid-columns: '5'` |
| Unknown component property | Accept with warning | `borderColor` |
| Duplicate section heading | **Error; reject the file** | Two `## Colors` headings |

Practical implications for generation: extra sections like `## Iconography` are safe to add after the core order; custom component properties are tolerated but prefer the standard eight; never emit duplicate section headings.
