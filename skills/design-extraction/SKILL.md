---
name: design-extraction
description: "Extracts design tokens, components, and screens from a codebase and builds a structured design system in Pencil (.pen) files. Use when user says 'design extraction', 'extract design', 'build design system', 'code to design', 'デザイン抽出', 'デザインシステム構築', 'コンポーネント抽出', or wants to reverse-engineer UI from code into a design tool."
argument-hint: "[pen-file-path]"
allowed-tools: Read, Glob, Grep, Bash, Write, Edit, Agent, WebFetch, mcp__pencil__get_editor_state, mcp__pencil__open_document, mcp__pencil__get_guidelines, mcp__pencil__batch_design, mcp__pencil__batch_get, mcp__pencil__get_screenshot, mcp__pencil__get_variables, mcp__pencil__set_variables, mcp__pencil__find_empty_space_on_canvas, mcp__pencil__snapshot_layout, mcp__pencil__get_style_guide_tags, mcp__pencil__get_style_guide, mcp__pencil__search_all_unique_properties, mcp__pencil__replace_all_matching_properties
model: opus
---

# Design Extraction — Code to Design System

## Overview

This skill provides a structured workflow for extracting design elements from an existing codebase and building a complete design system in Pencil (.pen) files. It reverse-engineers UI implementations — especially those created through vibe coding or by non-designers — into organized, inspectable design artifacts.

**Direction:** Code → Design (the reverse of Figma's design → code workflow)

## Recommended Project Size

This skill works best with **small to medium-sized projects** (10–30 components, a few dozen screens) — especially those created through vibe coding or by non-designers. For large or complex codebases, ask the user to narrow the scope to a specific directory or subset of components before starting.

## Prerequisites

- Pencil MCP server must be connected and accessible
  - Before proceeding, verify the Pencil MCP server is connected by checking if Pencil tools (e.g., `get_editor_state`) are available.
  - If the tools are not available, guide the user to enable the Pencil MCP server. They may need to restart their MCP client afterward.
- Target project must contain UI code (HTML/CSS/components in any framework)
- A `.pen` file must exist in the repository and be open in the Pencil editor
  - The user should create a new `.pen` file in Pencil beforehand (e.g., `design.pen` at the project root)
  - If no `.pen` file exists, prompt the user: "Please create a new .pen file in Pencil and open it, then let me know when ready."
  - Use `get_editor_state` to confirm the .pen file is open before starting

## Core Principles

**Follow these principles throughout every step. Violations lead to hallucination.**

### 1. Code is the Source of Truth

Code reading is the single most important factor for accuracy. Screenshots are supplementary.

- Read the **actual component code** (Vue/React/Svelte/HTML files) to extract exact values
- Extract style values directly from inline styles, CSS classes, Tailwind utilities, or CSS variables
- Never guess — if a value isn't in the code, don't invent it
- If something is plain text in code, it must be plain text in the design (no embellishment)

### 2. Images Must Be Placed as Images

When the code uses image files, they **must** be placed as image fills in the .pen file. Never substitute with text labels.

**Detection patterns** — search code for:
- `import xxxImage from '../../assets/xxx.png'` (ESM import)
- `<img :src="xxxImage" />` / `<img src="./xxx.png" />` (img tags)
- `require('./assets/xxx.png')` (CommonJS require)
- `background-image: url(...)` (CSS background)
- `src={xxxImage}` (JSX/TSX)

**Placement method** in .pen files:
```javascript
img=I(parent,{type:"frame",width:44,height:44,fill:{type:"image",url:"./relative/path/to/image.png",mode:"fit"}})
```

**Important:** The `url` must be a **relative path from the .pen file**. This applies to logos, mascot characters, brand labels, icons — everything that is an image in code must be an image in the design.

### 3. Use Design Tokens, Not Hardcoded Values

Every color, font size, spacing, and border radius in the .pen file should reference a `$variable-name`, not a raw hex/px value.

### 4. Design System Completeness

This skill does not just extract what exists in code — it builds a **complete design system**. That means actively identifying and proposing missing tokens and component states that a proper design system needs, even if the code hasn't implemented them yet.

### 5. Single Source of Truth — Catalogs Are the Definitions

**CRITICAL:** The .pen file has exactly two catalogs — named **"Token Catalog"** and **"Component Catalog"**. Do NOT use other names (e.g., "Variables Catalog", "Design Tokens", "Reusable Components"). These two catalogs are the ONLY locations where tokens and components are defined.

- Do NOT create reusable components outside the Component Catalog (no off-canvas components, no separate "Reusable Components" section)
- Do NOT create tokens outside the Token Catalog
- When new tokens or component states are added in later steps (e.g., Step 4), they MUST be added directly into the existing catalogs, not placed elsewhere
- Screen reproductions reference catalog components via `ref` instances

## Required Workflow

**There are exactly 6 steps. Follow them in strict order:**

| Step | Name | GATE |
|------|------|------|
| 1 | Repository Analysis | Yes |
| 2 | Design Token Extraction & Completion | Yes |
| 3 | Component Catalog | No — proceed directly to Step 4 |
| 4 | Design System Audit | Yes (covers Step 3 + 4 together) |
| 5 | Token–Component Binding | Yes |
| 6 | Screen Reproduction (with optional screenshot comparison) | Yes |

**There is NO separate screenshot capture step.** Screenshots are only captured as part of Step 6-E (optional, after screen reproduction).

Each step has a GATE — you must show the deliverable to the user and get confirmation before proceeding to the next step. Do NOT skip any step.

---

### Step 1: Repository Analysis

**Purpose:** Understand the project's tech stack, file structure, UI implementation patterns, and locate all style value sources for token extraction.

**Actions:**

1. Identify frameworks and libraries (package.json, pom.xml, Gemfile, etc.)
2. Map the UI component directory structure
3. Identify styling approach (CSS Modules, Tailwind, inline styles, CSS-in-JS, etc.)
4. Check for shared layouts and templates
5. **Image asset inventory:**
   - Scan for all images: `Glob("**/*.{png,jpg,svg,webp}")`
   - Identify where each image is used: `Grep("import.*from.*assets")`
   - Create an **image asset table** (filename / purpose / usage location / relative path from .pen)
   - This table is referenced in later steps when placing images
6. Identify icon libraries (lucide, heroicons, material icons, etc.)
7. **Style value source mapping** — locate and catalog all sources of design values:
   - CSS/SCSS variable definition files (e.g., `variables.css`, `:root` blocks)
   - Tailwind config (`tailwind.config.js` / `tailwind.config.ts`)
   - Global/shared CSS files
   - Theme configuration files
   - Scan for inline style density: `Grep('style="')` across all component files
   - If inline styles are prevalent, list the files with the most inline styles — these must be read in Step 2
   - Note: this step identifies WHERE values live, not what they are. Actual extraction happens in Step 2.

**Output:** Project overview report (tech stack, file structure, styling approach, image asset table, style value source map)

**🚫 GATE: Report to the user and wait for confirmation before proceeding.**

---

### Step 2: Design Token Extraction & Completion

**Purpose:** Extract tokens from code AND build a complete design token system by adding missing tokens that a proper design system requires.

**Actions:**

#### 2-A: Extract from Code

Using the **style value source map** from Step 1, systematically read each source:

1. **CSS/SCSS variable files** — read and extract all custom property definitions
2. **Tailwind config** — read `theme.extend` and extract all custom values
3. **Global CSS files** — extract repeated values from shared stylesheets
4. **Inline styles** — for files identified in Step 1 as having many inline styles, read each file and extract all `style="..."` values. Collect:
   - Color values (hex, rgb, hsl)
   - Font sizes, weights, line heights
   - Padding, margin, gap values
   - Border radius, box-shadow values
5. **Deduplicate and organize** — group identical values, identify semantic purpose

#### 2-B: Add Missing Design System Tokens

After extracting from code, **actively identify and add tokens that a complete design system needs but the code doesn't define.** This is mandatory, not optional.

**Required token categories to check and fill:**

| Category | What to check | Examples of commonly missing tokens |
|----------|--------------|--------------------------------------|
| **Interactive colors** | hover, active, focus variants for every primary/secondary color | `primary-hover`, `primary-active`, `primary-disabled` |
| **Semantic colors** | success, warning, error, info — each with base, light, and dark | `error`, `error-light`, `error-dark` |
| **Text colors** | primary, secondary, tertiary, disabled, inverse, on-primary | `text-secondary`, `text-disabled`, `text-on-primary` |
| **Background colors** | surface, overlay, disabled, hover states | `bg-hover`, `bg-disabled`, `bg-overlay` |
| **Border colors** | default, focus, error, disabled | `border-focus`, `border-error` |
| **Shadows** | sm, md, lg elevation levels | `shadow-sm`, `shadow-md`, `shadow-lg` |
| **Typography** | complete size scale (xs through 2xl), weight scale | `font-size-xs`, `font-size-2xl` |
| **Spacing** | complete scale (2xs through 3xl) | `spacing-2xs`, `spacing-3xl` |
| **Border radius** | complete scale (none, sm, md, lg, xl, full/pill) | `radius-none`, `radius-xl`, `radius-full` |

#### 2-C: Create Token Catalog

Create a "Token Catalog" section in the .pen file. This is the **single source of truth** for all design tokens. All tokens (extracted + proposed) are visualized here.

Organize by category:

1. **Theme Colors** — primary, secondary, accent, etc.
2. **Gray Scale** — full gray palette
3. **Semantic/Status Colors** — success, warning, error, info (with light/dark variants)
4. **Interactive Colors** — hover, active, focus, disabled states
5. **Typography** — font size and weight specimens
6. **Spacing** — visual scale
7. **Border Radius** — visual samples
8. **Shadows** — elevation samples

For each token, show:
- Variable name
- Actual value (color swatch for colors, specimen text for typography, etc.)
- **Implementation status — MUST be visually indicated for EVERY token:**
  - **Implemented** (extracted from code) → green dot (fill: `#22c55e`) + green colored label text (fill: `#22c55e`)
  - **Proposed** (not in code) → orange dot (fill: `#f97316`) + orange colored label text (fill: `#f97316`) + ⚠ marker

**This status visualization is mandatory for the Token Catalog**, just as it is for the Component Catalog. Every single token must have either a green or orange indicator. Do NOT omit status indicators from tokens. Do NOT leave indicators as default gray/black.

Save all tokens (extracted + proposed) using `set_variables`.

**Naming conventions:**
- Colors: `primary`, `primary-hover`, `text-main`, `gray-100` (purpose-based)
- Fonts: `font-size-sm`, `font-weight-bold-str`
- Spacing: `spacing-xs`, `spacing-md`
- Radii: `radius-sm`, `radius-lg`, `radius-pill`

**🚫 GATE: Show the Token Catalog screenshot to the user. Report how many tokens were extracted vs. proposed. Wait for confirmation before proceeding.**

---

### Step 3: Component Catalog

**Purpose:** Extract components from code and build them directly **inside the Component Catalog**. The catalog IS the component library — components are defined here and nowhere else.

**⚠️ CRITICAL RULES:**
- **Do NOT create components outside the Component Catalog frame.** No off-canvas components. No separate "Reusable Components" section.
- Every component is created as `reusable: true` **directly inside the Component Catalog**.
- The catalog is not a "reference to" components — it IS the components.

**Actions:**

#### 3-A: Identify Components

Extract **two categories** of components:

**Atomic components** (small, reusable UI primitives):
1. Collect explicit component files (Vue/React/Svelte component files) for buttons, inputs, badges, etc.
2. Record each component's structure, variations, and usage locations

**Composite/project-specific components** (larger patterns unique to this project):
1. Identify patterns that are reused across multiple views — e.g., custom section headings with decorations, info cards with specific layouts, list item patterns with icons and badges
2. Look for repeated HTML/template structures across different views: `Grep` for similar class names, structure patterns, or wrapper components
3. Check for layout patterns: headers with specific arrangements, footer patterns, sidebar structures
4. These are NOT generic design system primitives — they are **this project's unique reusable patterns**

**Examples of composite components:**
- A section heading with a left border and icon (used across 5 views)
- A product card with image, title, price badge, and metadata layout
- A stats row with icon + label + value pattern
- A FAQ accordion item with question/answer/toggle structure
- A profile summary card with avatar + name + stats

#### 3-B: Build Component Catalog

Create a "Component Catalog" section in the .pen file. For each component:

1. Create a **component group frame** inside the Component Catalog with the component name as title
2. Inside that group, build the **default state** as a `reusable: true` component
3. Inside that same group, build **all implemented variants** side by side (e.g., Primary Button, Secondary Button, Danger Button)
4. Add usage location annotations (which views/screens use this component)
5. Mark each variant with a green dot (fill: `#22c55e`) + green colored label text (fill: `#22c55e`) to indicate "implemented in code"

**Atomic components to look for:**
- Buttons (Primary, Secondary, Soft, Danger, Disabled)
- Input fields (Default, Focus, Error, Disabled)
- Badges/Chips (status, category)
- Alerts/Toasts
- Icons

**Composite components to look for:**
- Section headings (custom styled headings used across views)
- Cards (product card, info card, profile card — project-specific layouts)
- List items (with specific icon + text + metadata arrangements)
- Navigation (header, tabs, bottom nav)
- Modals/Sheets
- Accordion/expandable items
- Any repeated multi-element pattern used in 2+ views

**Layout:** Each component group shows variants in a horizontal row with labels, all inside the Component Catalog frame.

**Do NOT stop here — proceed directly to Step 4 (Design System Audit) without waiting for user confirmation.** The GATE for component work comes at the end of Step 4, after both extraction and audit are complete.

---

### Step 4: Design System Audit — Missing States & Proposals

**Purpose:** Audit every component and token for completeness as a design system. This step is the primary value of the skill — **making gaps visible.**

**This step is MANDATORY and must produce concrete, visible output in the .pen file.** Do not skip or minimize this step. If you skip this step, the skill has failed its primary purpose.

**Actions:**

#### 4-A: Audit Each Component for Missing States

For **every** component created in Step 3, systematically check for these states and create them if missing:

| Component Type | Required States to Check |
|---------------|-------------------------|
| **Buttons** | Default, Hover, Active/Pressed, Focus (with focus ring), Disabled, Loading |
| **Input fields** | Default, Placeholder, Filled, Focus, Error (with message), Disabled, Read-only |
| **Cards** | Default, Hover, Selected/Active, Loading (skeleton), Empty |
| **Badges/Chips** | Each semantic color variant (success, warning, error, info), Removable |
| **Navigation** | Active tab/link, Inactive, Hover, With notification badge |
| **Modals/Sheets** | Open state, With overlay backdrop |
| **List items** | Default, Hover, Selected, Loading (skeleton), Empty state |
| **Alerts/Toasts** | Success, Warning, Error, Info, With dismiss button |

For each missing state:
1. **Create the state variant directly inside the Component Catalog**, placed next to existing variants in the same component group. Create it as `reusable: true` — same as implemented states.
2. Mark with orange dot (fill: `#f97316`) + orange colored label text (fill: `#f97316`) + ⚠ marker (proposed, not in code)
3. Use proposed tokens from Step 2-B (e.g., `$primary-hover` for button hover state)
4. Base the proposed design on common UI patterns and the project's existing visual language

**⚠️ FORBIDDEN ACTIONS — Do NOT do any of the following:**
- Do NOT create a new frame like "Component States (Proposed)" or "Proposed States" or any similarly named frame
- Do NOT create proposed states outside the Component Catalog
- Do NOT create any new top-level sections — the .pen file has ONLY: Token Catalog, Component Catalog, and Screens
- You MUST add proposed states into the **existing component group frames** created in Step 3 (e.g., add "Hover" variant next to "Default" inside the "Button" group)

#### 4-B: Audit Tokens Against Components

Verify that all tokens needed by the proposed component states exist:
- If a proposed hover state needs `$primary-hover` but it doesn't exist → add it to variables with `set_variables`
- If a proposed disabled state needs `$text-disabled` but it doesn't exist → add it to variables with `set_variables`
- **Add the new tokens to the Token Catalog** — insert them into the existing Token Catalog frame created in Step 2-C, with 🟠 indicators. Do NOT create a separate section for these tokens.

#### 4-C: Summary Report

Present a summary to the user:
```
Design System Audit Complete:

Components audited: N
- Button: 3 implemented states, 3 proposed (hover, focus, loading)
- Input: 2 implemented states, 4 proposed (focus, error, disabled, read-only)
- Card: 1 implemented state, 2 proposed (hover, skeleton)
- ...

Tokens added to Token Catalog: M
- 5 interactive color tokens (hover/active/disabled variants)
- 3 semantic color tokens (error-light, warning-light, info-light)
- 2 shadow tokens (shadow-sm, shadow-md)

All proposals are marked with 🟠 in the catalogs.
```

**🚫 GATE: Show screenshots of the updated Component Catalog (with proposed states) and updated Token Catalog (with added tokens). Present the summary report. Wait for confirmation before proceeding.**

---

### Step 5: Token–Component Binding

**Purpose:** Ensure every component property references a design token variable, not a hardcoded value. This step runs AFTER all tokens (including proposed ones from Step 4) are in place.

**Actions:**

1. Run `batch_get` on the Component Catalog with `readDepth: 4` to inspect all component nodes
2. **Scan for hardcoded values** in every component (both implemented and proposed):
   - Raw hex colors → replace with `$variable-name`
   - Raw font sizes → replace with `$font-size-*`
   - Raw spacing values → replace with `$spacing-*`
   - Raw border radii → replace with `$radius-*`
3. Replace all found hardcoded values with their corresponding `$variable-name` references
4. Verify proposed component states use proposed tokens (e.g., hover states use `$*-hover` tokens)
5. Take `get_screenshot` of the Component Catalog to confirm nothing broke

**Deliverable:** Report to the user how many hardcoded values were found and replaced.

**🚫 GATE: Report binding results to the user. Wait for confirmation before proceeding.**

---

### Step 6: Screen Reproduction

**Purpose:** Reproduce actual screens using the tokens and components from the catalogs.

#### Step 6-A: Screen Selection (User Approval Required)

1. List **all pages/views** from the router configuration with their URL paths and view names
2. Briefly describe each screen's purpose
3. Present to the user for selection:
   ```
   Found N screens. Which ones should I reproduce?

   Recommended (main screens):
   - [ ] /products — Product listing (ProductListView)
   - [ ] /products/:id — Product detail (ProductDetailView)
   - [ ] /mypage — My page top (MyPageTopView)

   Other screens:
   - [ ] /cart — Shopping cart (CartView)
   - [ ] /mypage/profile — Profile (ProfileView)
   - [ ] ...
   ```
4. Only reproduce screens the user selects
5. If many screens, suggest: "Let's start with 3, then do the rest later"

#### Step 6-B: Code Reading and Style Value Extraction

For each selected screen:

1. **Read the full View component code** (this is the most important step)
2. **Extract exact style values** from code:
   - Inline styles: `style="font-size: 20px; color: #1e293b; font-weight: 800"` → record exact values
   - CSS classes: `.section-title` → trace to CSS definition to find actual values
   - Tailwind: `text-lg font-bold text-slate-800` → convert to px/hex values
   - CSS variables: `var(--primary)` → check variable definition
3. **Map extracted values to design tokens** from Step 2:
   - If a matching token exists → use `$variable-name`
   - If no match → use as-is and record as "tokenization candidate"
4. **Check import statements for image references** → cross-reference with Step 1 image asset table

#### Step 6-C: Reproduce in .pen File

1. **Reproduce faithfully based on code** — code is the source of truth
2. **Use components from the Component Catalog** (Step 3) as `ref` instances wherever possible
3. Use design token variables (`$variable-name`) for all values
4. Place images with `fill: {type: "image", url: "relative-path"}`
5. Use multiple `batch_design` calls if needed — do NOT stop partway through a screen

**Screen type rules:**

- **List/index screens** (e.g., product listing, search results): Repeating items should be limited to **3 items**. Do not reproduce all 20+ items from the data — 3 is enough to show the pattern.
- **Detail screens** (e.g., product detail, profile): Reproduce **ALL elements and sections** from the code. Do not stop partway through. If the screen has many sections, use multiple `batch_design` calls (each up to 25 operations) until every section is complete.

**⚠️ Detail screens MUST be complete.** If a detail screen has 10 sections in the code, all 10 must appear in the .pen file. A single `batch_design` call may not be enough — call it as many times as needed. After each `batch_design`, check progress with `get_screenshot` and continue building remaining sections until the entire screen is reproduced.

#### Step 6-D: Token and Component Binding Verification

After **all selected screens** are reproduced, verify tokens and components:

1. For each screen, run `batch_get` with `readDepth: 4` to inspect all nodes
2. **Scan for hardcoded values** — search the node tree for:
   - Raw hex colors (e.g., `#1e293b`) that should be `$text-main`
   - Raw font sizes (e.g., `16`) that should be `$font-size-md`
   - Raw spacing values (e.g., `12`) that should be `$spacing-md`
   - Raw border radii (e.g., `14`) that should be `$radius-md`
3. **Replace all hardcoded values** with their corresponding `$variable-name` references
4. **Check reusable component usage** — if a pattern exists as a `reusable: true` component in the Component Catalog, it should be used as a `ref` instance, not duplicated inline

**Strict rules:**
- Do NOT add decorations or layouts that don't exist in code
- Do NOT make plain text into rich UI (hallucination prevention)
- Use default values and fallbacks from code for data
- Maintain the exact section order from code
- Do NOT "improve" the design speculatively — reflect current code as-is
- Images in code MUST be images in .pen — reference the Step 1 asset table

#### Step 6-E: Screenshot Comparison (Optional)

After all screens are reproduced, **ask the user** whether to run screenshot comparison:

```
All screens have been reproduced. Would you like me to capture live screenshots and compare them with the .pen reproductions to check for missing elements?
- Yes: I'll start the dev server, capture screenshots, and compare
- Skip: I'll verify manually
```

**If the user says yes:**

1. Start the dev server (backend + frontend)
2. Health check: `curl -s -o /dev/null -w "%{http_code}" http://localhost:PORT/`
3. Determine viewport size from code analysis:
   - Mobile-first projects → `448,900`
   - Desktop-first projects → `1440,900`
   - Responsive projects → capture both
4. Capture screenshots with Playwright for each reproduced screen:
   ```bash
   npx playwright screenshot --viewport-size="WIDTH,900" --full-page \
     "http://localhost:PORT/path" screenshots/screen-name.png
   ```
5. Stop servers
6. For each screen, **compare** the live screenshot with the .pen `get_screenshot`:
   - View both screenshots
   - Identify missing or incorrect elements
   - List discrepancies for the user
7. Fix any discrepancies found
8. Place captured screenshots as reference images in the .pen file

**🚫 GATE: Present the comparison results (or skip confirmation) to the user. Ask the user to verify the reproduced screens for any remaining issues.**

---

## .pen File Structure

The final .pen file should have this structure:

```
document
├── Reference Screenshots    # Live screenshots for comparison (Step 6-E, optional)
├── Token Catalog            # ALL tokens live here (Step 2 + additions from Step 4)
│   ├── Theme Colors         # 🟢 extracted + 🟠 proposed
│   ├── Gray Scale
│   ├── Semantic Colors
│   ├── Interactive Colors
│   ├── Typography
│   ├── Spacing
│   ├── Border Radius
│   └── Shadows
├── Component Catalog        # ALL components live here (Step 3 + additions from Step 4)
│   ├── Button               # reusable: true — Default🟢, Hover🟠, Focus🟠, Disabled🟠
│   ├── Input                # reusable: true — Default🟢, Focus🟠, Error🟠
│   ├── Card                 # reusable: true — Default🟢, Hover🟠
│   └── ...
├── Screens                  # Screen reproductions using ref instances (Step 6)
│   ├── Screen 1
│   ├── Screen 2
│   └── ...
```

**⚠️ There are ONLY these sections. The following are explicitly FORBIDDEN:**
- No "Reusable Components" section
- No "Component States (Proposed)" section
- No "Proposed States" or "Missing States" section
- No components outside the Component Catalog
- No tokens outside the Token Catalog
- Any new item (proposed or otherwise) MUST be added into the existing catalog frames

---

## Validation Checklist

Before marking a screen reproduction as complete, verify:

- [ ] Section order matches code exactly
- [ ] All text content matches code defaults/fallbacks
- [ ] Colors use `$variable-name` references (no hardcoded hex)
- [ ] Font sizes use `$font-size-*` variables
- [ ] Spacing uses `$spacing-*` variables
- [ ] Border radii use `$radius-*` variables
- [ ] All images from code are placed as image fills (not text)
- [ ] No elements exist that aren't in the code (no hallucination)
- [ ] Plain text in code remains plain text in design
- [ ] `get_screenshot` output visually matches the code structure
- [ ] Components from Component Catalog are used as `ref` instances

---

## Implementation Rules

### .pen File Operations

- Keep each `batch_design` call to **maximum 25 operations**
- **If a screen or component requires more than 25 operations, use multiple `batch_design` calls.** Do NOT stop at 25 operations and leave the screen incomplete. Call `batch_design` repeatedly until the entire screen is finished.
- Use **placeholder flag** on frames during work, remove when done
- Always verify structure with **snapshot_layout** after major changes
- Use `get_screenshot` periodically for visual validation — especially between `batch_design` calls to track progress

### Design Token Usage

- Every `fill`, `fontSize`, `fontWeight`, `padding`, `gap`, `cornerRadius` should use `$variable-name`
- Never hardcode values that have a matching token
- If a value has no matching token, add it as a new token immediately and add it into the Token Catalog

### Component Organization

- All reusable components live in the **Component Catalog** — nowhere else
- When adding proposed states in Step 4, add them **inside the existing component groups** in the catalog
- Use `ref` instances in screen reproductions, not duplicated structures
- Extend existing components rather than creating overlapping ones

---

## Common Issues and Solutions

### Issue: Hallucinated UI elements

**Cause:** Designing based on assumptions instead of code reading.
**Solution:** Always read the full View component code before reproducing. If code shows `<p>plain text</p>`, the design must be plain text — not a styled card or step indicator.

### Issue: Images replaced with text labels

**Cause:** Not checking `import` statements for image assets.
**Solution:** In Step 1, create the image asset table. In Step 6-B, check all `import` statements. Every imported image must be placed with `fill: {type: "image", url: "..."}`.

### Issue: Inaccurate colors or font sizes

**Cause:** Guessing token values instead of reading inline styles.
**Solution:** In Step 1, map all style value sources including inline-heavy files. In Step 2-A, read those files and extract exact values.

### Issue: Wrong section order

**Cause:** Arranging sections based on screenshot appearance rather than code structure.
**Solution:** Follow the exact order of elements in the component template/JSX. Code order = design order.

### Issue: Components split across multiple locations

**Cause:** Creating reusable components off-canvas AND in the catalog separately.
**Solution:** The Component Catalog is the ONLY location for reusable components. Do not create a separate off-canvas section. Build components directly inside the catalog frame.

### Issue: No missing states proposed / Step 4 skipped

**Cause:** Step 4 was skipped or treated as optional.
**Solution:** Step 4 is mandatory and has a GATE. For every component, systematically check the required states table and create proposals for each missing state with 🟠 indicators. The skill has failed if Step 4 produces no output.

### Issue: Tokens are minimal / code-only

**Cause:** Only extracting tokens that exist in code, not building a complete design system.
**Solution:** Step 2-B is mandatory. Check every token category in the table and add missing tokens. A design system needs hover states, disabled states, semantic colors, shadows, etc. even if the code doesn't define them yet.

### Issue: Proposed items not added to catalogs

**Cause:** New tokens or component states created in Step 4 are placed outside the existing catalogs.
**Solution:** Always add new items INSIDE the existing catalog frames. Insert new tokens into the Token Catalog. Insert new component states into the corresponding component group in the Component Catalog. Never create a separate section.

### Issue: Step 5 (binding) skipped


**Cause:** Assumed binding was already done during component creation.
**Solution:** Step 5 is a dedicated verification pass. Even if you used `$variable-name` during creation, run `batch_get` to verify — hardcoded values often slip through. Report the number of replacements to the user.

---

## Best Practices

### Always Start with Code

Never reproduce a screen based solely on screenshots. Always read the full component code first. Screenshots are for overview validation only.

### Incremental Validation

Validate frequently with `get_screenshot` during reproduction, not just at the end. This catches structural issues early.

### Extract and Complete Before Reproducing

Complete Steps 1–5 (analysis, token extraction + completion, component catalog, design audit, binding) before starting Step 6 (screen reproduction). The tokens and components created earlier make reproduction faster and more consistent.

### Make Gaps Visible

The primary value of this skill is making design gaps visible. When creating the catalogs, clearly distinguish between implemented (🟢) and proposed (🟠⚠) elements. A catalog with only green dots means the audit in Step 4 was not thorough enough.

### Single Source of Truth

The Token Catalog and Component Catalog are the single sources of truth. Never create tokens or components in multiple places. Everything lives in the catalogs. Screen reproductions reference the catalog via `ref` instances.

### Respect the Gates

Each step has a GATE (🚫). You MUST show the deliverable to the user and wait for their confirmation before proceeding. This prevents skipping steps and ensures quality at each stage.

---

## Examples

### Example 1: Extracting from a Vue.js E-Commerce App

User says: "Extract design system from this project"

**Step 1 — Repository Analysis:**
Read `package.json` → Vue 3 + Vite + vue-router + lucide-vue-next. Scan `frontend/src/` → map views, components, styles, assets. Find `logo.png`, `hero-banner.png` → create image asset table. Map style value sources: `styles.css` (CSS variables), inline-heavy view files. → **GATE**

**Step 2 — Token Extraction & Completion:**
Extract CSS variables from `styles.css`. Read inline-heavy files for additional values. Add missing tokens: hover colors, disabled states, shadows, full spacing scale. Create Token Catalog in .pen with 🟢/🟠 indicators. → **GATE**

**Step 3 + 4 — Component Catalog & Audit:**
Read component files → build Component Catalog with Button, Card, Badge, Header (atomic) + ProductCard, SectionHeading (composite). Audit each component → add hover, focus, disabled, loading states with 🟠 into existing groups. Add missing tokens into Token Catalog. → **GATE**

**Step 5 — Binding:**
Run `batch_get` on all components → find and replace hardcoded values with `$variable-name`. → **GATE**

**Step 6 — Screen Reproduction:**
Ask user which screens → user selects 3. Read each View's full code → reproduce using catalog components as refs. Optionally capture live screenshots and compare.

**Result:** Complete design system with Token Catalog, Component Catalog (with proposed states visible), and 3 faithfully reproduced screens.

### Example 2: Extracting from a Tailwind + React Project

User says: "/design-extraction:design-extraction design.pen"

**Step 1:** Read `package.json` → React + Tailwind CSS. Read `tailwind.config.js` → map custom theme. → **GATE**

**Step 2:** Extract Tailwind theme tokens. Convert utilities to explicit values (text-lg → 18px). Add missing tokens: focus ring colors, disabled opacity, elevation shadows. Create Token Catalog. → **GATE**

**Step 3 + 4:** Scan `src/components/` → build Component Catalog (atomic + composite). Audit → add missing states into catalog groups. → **GATE**

**Step 5:** Verify binding across all components. → **GATE**

**Step 6:** Reproduce selected screens with full token and component references.

**Result:** Design system that bridges Tailwind config and visual design, with gaps clearly identified.
