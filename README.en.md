[日本語](./README.md) | **English**

# Design Extraction Skills for Claude Code

A Claude Code plugin that reverse-engineers UI from code into structured design systems in [Pencil](https://pencil.dev/) (.pen) files.

Application development no longer always starts in a design tool — more projects begin with code, and non-designers like engineers and PMs are increasingly involved in UI decisions. Yet someone still needs to ensure design consistency and quality. This plugin was built for designers who understand this shift and want to support design within these new workflows.

## Skills

### `/design-extraction:design-extraction`
**Code → Design System & Screen Designs**

Analyzes a codebase and reproduces a design system with screen designs in a .pen file:

1. **Design tokens** — Extracts colors, fonts, spacing, radii from code → saves as variables
2. **Components** — Identifies recurring UI patterns → creates reusable components
3. **Missing elements** — Discovers missing states (hover, disabled, error, etc.) → proposes additions
4. **Catalogs** — Creates visual overviews of tokens and components (implemented vs. proposed)
5. **Screen reproduction** — Faithfully reproduces main screens from code (optionally verified against live screenshots)

### `/design-extraction:design-variants`
**Code Conditions → Screen Variants**

Building on the design system and screens created by `design-extraction`, this skill analyzes conditional branches in code and generates designs for each visual pattern:

- **User roles** — Guest, free user, paid user, admin
- **Data states** — Loading, loaded, empty, error
- **Feature toggles** — Badges visible/hidden, sections enabled/disabled
- **Interaction states** — Modals open/closed, accordions expanded/collapsed

*For screens with complex conditional logic, generating design states from code on demand is more practical than maintaining them manually in a design tool.*

## Best suited for

This plugin is designed for **small to medium-sized projects** where code is the source of truth for design:

- Projects created through vibe coding or by non-designers
- 10–30 components, a few dozen screens
- Single-purpose apps, MVPs, internal tools, prototypes

For large or complex codebases, narrow the scope by specifying a target directory or a subset of components:

```bash
# Focus on a specific directory
/design-extraction:design-extraction --scope src/components/dashboard

# Or specify in conversation
"Extract design system from the /src/views/settings/ directory only"
```

## Prerequisites

- [Claude Code](https://claude.com/claude-code) installed
- [Pencil MCP server](https://pencil.dev/) connected
- A `.pen` file saved in the repository and open in Pencil editor
- Target project with UI code (any framework: Vue, React, Svelte, etc.)

## Installation

### Quick start (single session)

```bash
git clone https://github.com/tikeda/design-extraction-skill.git
claude --plugin-dir ./design-extraction-skill
```

### Permanent install

Inside a Claude Code session:

```
/plugin marketplace add tikeda/design-extraction-skill
/plugin install design-extraction@design-extraction-skill
```

## Usage

```bash
/design-extraction:design-extraction
```

Options:

```bash
# Specify a .pen file
/design-extraction:design-extraction design.pen
```

```bash
/design-extraction:design-variants
```

Claude Code will display a list of available views and variants after execution, so you can select interactively. You can also specify them directly:

```bash
# Specify a view file
/design-extraction:design-variants src/views/RecipeDetailView.vue

# Specify both view and .pen file
/design-extraction:design-variants src/views/RecipeDetailView.vue design.pen
```

## How It Works

### Design Extraction (6 steps)

| Step | What |
|------|------|
| 1 | Analyze repository (tech stack, assets, structure) |
| 2 | Extract design tokens + add missing tokens + create Token Catalog |
| 3 | Extract components (atomic + composite) → build Component Catalog |
| 4 | Design system audit — propose missing states for every component |
| 5 | Bind tokens ↔ components (with all tokens in place) |
| 6 | Reproduce screens (screenshot comparison is optional) |

### Design Variants (7 steps)

| Step | What |
|------|------|
| 1 | Identify target component |
| 2 | Analyze conditional branches (`v-if`, ternary, computed, etc.) |
| 3 | Build variant matrix → user selects which to generate |
| 4 | Prepare diff spec for each variant |
| 5 | Copy default screen → apply diffs |
| 6 | Token and component binding verification |
| 7 | Validate and label with code condition annotations |

## Design Philosophy

### Code First

Code is the source of truth. Screenshots are supplementary for overview validation only. Every design element must be traceable to actual code.

### Images Are Images

When code uses an image file (`import logo from './logo.png'`), it must be placed as an image fill in the .pen file — never substituted with text.

### Tokens, Not Hardcoded Values

Every color, font size, spacing, and border radius uses `$variable-name` references. Hardcoded hex/px values are verified and replaced after screen creation.

### Meaningful Variants, Not Explosion

Rather than generating every possible combination of conditions, only meaningful states that designers need to review are produced. Generating states from code on demand is more practical than preparing every pattern upfront.

## .pen File Structure

```
document
├── Reference Screenshots       # (Optional) Live screenshots for comparison
├── Token Catalog                # Design tokens with 🟢 implemented / 🟠 proposed
├── Component Catalog            # All components + states (single source of truth)
│   ├── Button                   # Default🟢, Hover🟠, Focus🟠, Disabled🟠
│   ├── Card                     # Default🟢, Hover🟠
│   └── ...
├── Screens                      # Screen reproductions
│   ├── Product List
│   ├── Product Detail
│   └── ...
└── Screen Variants              # Conditional state variants
    ├── RecipeDetailView — Variants
    │   ├── Guest User
    │   ├── Loading
    │   ├── Error
    │   └── ...
    └── ...
```
