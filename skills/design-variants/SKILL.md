---
name: design-variants
description: "Analyzes conditional branches in UI code and generates all screen variants (user roles, states, toggles, errors) in Pencil (.pen) files. Use when user says 'generate variants', 'screen states', 'design variants', 'conditional states', 'バリアント生成', '画面状態', '条件分岐のデザイン', or wants to visualize all UI states from code."
argument-hint: "<view-file-path> [pen-file-path]"
allowed-tools: Read, Glob, Grep, Bash, Agent, mcp__pencil__get_editor_state, mcp__pencil__open_document, mcp__pencil__get_guidelines, mcp__pencil__batch_design, mcp__pencil__batch_get, mcp__pencil__get_screenshot, mcp__pencil__get_variables, mcp__pencil__set_variables, mcp__pencil__find_empty_space_on_canvas, mcp__pencil__snapshot_layout, mcp__pencil__search_all_unique_properties, mcp__pencil__replace_all_matching_properties
model: opus
---

# Design Variants — Generate All Screen States from Code

## Overview

This skill analyzes conditional rendering logic in UI components and generates all meaningful screen variants in a Pencil (.pen) file. Instead of maintaining a complex master design in Figma, let the code be the source of truth for all UI states.

**Prerequisite:** The `design-extraction` skill should have been run first, so that design tokens, reusable components, and at least one default screen reproduction already exist in the .pen file.

## The Problem

A single screen can have many visual states driven by code conditions:

```
Recipe Detail Screen
├── Guest user (not logged in)
├── Free user (logged in)
├── Premium user
├── Bookmark: saved / not saved
├── Nutrition info: available / loading / unavailable
├── Instructions: collapsed / expanded
├── Review section: collapsed / expanded
├── Share sheet: closed / open
├── Error state
└── Loading state
```

Managing all these as separate Figma frames manually is painful and error-prone. The code already handles all conditions correctly — this skill extracts them automatically.

## Core Principles

### 1. Code is the Source of Truth

All variants are derived from actual code conditions, not speculation. If a condition doesn't exist in code, don't create a variant for it.

### 2. Meaningful Combinations, Not Combinatorial Explosion

10 boolean conditions = 1024 combinations. Don't generate all of them. Focus on **meaningful, user-facing states** that designers and stakeholders need to review.

### 3. Reuse the Existing Design System

Variants should use the same design tokens (`$variable-name`) and reusable components from the **Component Catalog** already established by `design-extraction`. Each variant is a reconfiguration of existing elements, not a rebuild from scratch.

### 4. Use Design Tokens, Not Hardcoded Values

Every color, font size, spacing, and border radius in variant screens must reference `$variable-name`. When modifying elements for a variant, use token references, not raw hex/px values.

### 5. Variants Live in the "Screen Variants" Section

All variant screens are placed inside a "Screen Variants" section in the .pen file. Do NOT create variants in other locations.

## Required Workflow

**Follow these steps in strict order. Each step has a GATE — you must show the deliverable to the user and get confirmation before proceeding to the next step. Do NOT skip any step.**

---

### Step 1: Identify Target Component

**Actions:**

1. If the user provides a file path, read that component
2. If not, list available View components and let the user choose:
   ```
   Available screens:
   - /recipes — RecipeListView.vue
   - /recipes/:id — RecipeDetailView.vue
   - /mypage — MyPageTopView.vue
   ...
   Which screen should I analyze for variants?
   ```
3. Read the **full component code** — template, script, and style sections
4. Also read all child components that are imported and used in the template

**🚫 GATE: Confirm target component with the user before proceeding.**

---

### Step 2: Analyze Conditional Branches

**Purpose:** Extract all conditions that change the UI rendering.

**Detection patterns to search for:**

| Framework | Patterns |
|-----------|----------|
| Vue | `v-if`, `v-else-if`, `v-else`, `v-show`, ternary in `:class`/`:style` |
| React | `{condition && <...>}`, `{condition ? <A> : <B>}`, early returns |
| Svelte | `{#if}`, `{:else if}`, `{:else}` |
| General | Computed properties, derived state, props with defaults |

**For each condition found, record:**

1. **Condition expression** — e.g., `v-if="isBookmarked"`
2. **What changes** — which UI elements appear/disappear/change
3. **Source of the condition** — prop, computed, API response, user action
4. **Category** — classify into one of:
   - **User role** (guest, free, paid, admin)
   - **Data state** (loading, loaded, empty, error)
   - **Feature toggle** (badge visible/hidden, section expanded/collapsed)
   - **Interaction state** (modal open/closed, accordion expanded)

**Output:** A structured list of all conditions with their categories.

**🚫 GATE: Present the condition list to the user. Wait for confirmation before proceeding.**

---

### Step 3: Define Variant Matrix

**Purpose:** Group conditions into meaningful variant sets and get user approval.

**Actions:**

1. Organize conditions by category
2. Identify **independent** vs **dependent** conditions:
   - Independent: can be toggled regardless of others (e.g., FAQ expanded)
   - Dependent: only exist under certain parent conditions (e.g., paid-user features)
3. Propose a **variant matrix** — a prioritized list of variants to generate:

   ```
   Found 12 conditions in RecipeDetailView. Proposed variants:

   High priority (recommended):
   - [ ] Default state (logged-in user, all data loaded)
   - [ ] Guest user (not logged in — CTA changes)
   - [ ] Loading state
   - [ ] Error state
   - [ ] Empty/not found state

   Medium priority:
   - [ ] Bookmarked recipe (saved indicator visible)
   - [ ] Non-bookmarked recipe (save button shown)
   - [ ] Instructions expanded
   - [ ] Review section expanded

   Low priority (interaction states):
   - [ ] Share sheet open
   - [ ] All sections collapsed
   - [ ] ...

   Which variants should I generate?
   ```

4. **Wait for user selection** before generating anything

**🚫 GATE: The user MUST select which variants to generate. Do NOT proceed without explicit selection.**

---

### Step 4: Prepare Variant Data

**Purpose:** For each selected variant, determine exactly what changes from the default state.

**Actions:**

1. For each variant, create a **diff spec** describing what changes:
   ```
   Variant: "Guest User"
   Changes from default:
   - CTA button text: "応募画面へ進む" → "ログインして応募"
   - Keep button: enabled → shows login prompt
   - AI summary: visible → hidden (requires auth)
   - Profile-dependent sections: hidden
   ```

2. Identify the **mock data** that produces each state:
   - What API response shape triggers this variant?
   - What prop values are needed?
   - What computed property results change?

3. Verify each diff against the code — don't invent changes that the code doesn't make

**🚫 GATE: Present the diff specs to the user. Wait for confirmation before generating.**

---

### Step 5: Generate Variant Screens

**Purpose:** Create each variant as a screen in the .pen file.

**Actions:**

1. Find empty space on canvas for the variant screens: `find_empty_space_on_canvas`
2. Create a **variant group frame** inside the "Screen Variants" section with a clear title:
   ```
   "RecipeDetailView — Variants"
   ```
3. For each variant:
   a. **Copy the default screen** as a starting point (use `C()` operation)
   b. **Apply the diff** from Step 4 — modify, show/hide, or replace elements
   c. **Label the variant** clearly (e.g., "Guest User", "Loading", "Error")
   d. **Add a condition annotation** showing the code condition:
      - e.g., `v-if="!isLoggedIn"` displayed as a small note near the variant
   e. Use `$variable-name` references for all values

4. Arrange variants in a logical grid:
   - Row = variant category (user role, data state, feature toggle)
   - Column = individual variants within that category

**Screen reproduction rules (same as design-extraction):**

- **List screens**: Repeating items should be limited to **3 items**. Do not reproduce all 20+ items.
- **Detail screens**: Reproduce **ALL elements and sections**. Do not stop partway through. Use multiple `batch_design` calls (each up to 25 operations) until every section is complete.
- Each `batch_design` call has a 25 operation limit, but you can call it **multiple times** to finish a screen. Check progress with `get_screenshot` between calls.

**⚠️ Detail screen variants MUST be complete.** If the default screen has 10 sections, each variant must also have all 10 sections (with the appropriate modifications). Do NOT generate partial variants.

**⚠️ FORBIDDEN ACTIONS:**
- Do NOT create variant screens outside the "Screen Variants" section
- Do NOT create a new top-level section for variants — use the existing "Screen Variants" section
- Do NOT modify the original default screen — always copy it first

---

### Step 6: Token and Component Binding Verification

**Purpose:** Verify that all variant screens use design tokens and Component Catalog components correctly.

**Actions:**

1. For each variant screen, run `batch_get` with `readDepth: 4` to inspect all nodes
2. **Scan for hardcoded values** — search the node tree for:
   - Raw hex colors (e.g., `#1e293b`) that should be `$text-main`
   - Raw font sizes (e.g., `16`) that should be `$font-size-md`
   - Raw spacing values (e.g., `12`) that should be `$spacing-md`
   - Raw border radii (e.g., `14`) that should be `$radius-md`
3. **Replace all hardcoded values** with their corresponding `$variable-name` references
4. **Check reusable component usage** — if a pattern exists as a `reusable: true` component in the Component Catalog, it should be used as a `ref` instance, not duplicated inline

**Deliverable:** Report to the user how many hardcoded values were found and replaced.

**🚫 GATE: Report binding results to the user. Wait for confirmation before proceeding.**

---

### Step 7: Validate and Label

**Purpose:** Ensure all variants are accurate and clearly labeled.

**Actions:**

1. Take `get_screenshot` of the variant group
2. Verify each variant visually — does it match what the code would render?
3. Ensure clear labeling:
   - Variant name (e.g., "Error State")
   - Code condition (e.g., `v-else-if="error"`)
   - What's different from default (brief description)
4. Verify all values use design token variables (no hardcoded values)
5. Run the **Validation Checklist** (see below)

**🚫 GATE: Show the variant group screenshot to the user. Ask the user to verify the variants for accuracy and completeness.**

---

## Variant Layout in .pen File

```
document
├── ... (existing design-extraction content: Token Catalog, Component Catalog, Screens)
└── Screen Variants
    ├── RecipeDetailView — Variants
    │   ├── Default (reference)
    │   ├── Guest User
    │   ├── Loading
    │   ├── Error
    │   ├── Bookmarked
    │   └── Instructions Expanded
    ├── RecipeListView — Variants
    │   ├── Default
    │   ├── Empty Results
    │   └── ...
    └── ...
```

---

## Validation Checklist

Before marking variants as complete:

- [ ] Each variant is derived from an actual code condition (no hallucination)
- [ ] Variant labels clearly describe the state
- [ ] Code condition is annotated on each variant
- [ ] Diff from default is accurate (only changed elements are modified)
- [ ] All values use `$variable-name` references (no hardcoded hex/px)
- [ ] Visual appearance matches what the code would render for that condition
- [ ] Variants are arranged in a logical, scannable grid
- [ ] Detail screen variants are complete (all sections present)
- [ ] Components from Component Catalog are used as `ref` instances

---

## Common Issues and Solutions

### Issue: Too many variants proposed

**Cause:** Every `v-if` generates a variant, leading to combinatorial explosion.
**Solution:** Focus on **user-facing states**, not internal code conditions. Group related conditions. Let the user choose priority.

### Issue: Variant doesn't match actual rendering

**Cause:** Misunderstanding the condition's effect — e.g., `v-if` hides a section but the surrounding layout also changes.
**Solution:** Trace the full impact of each condition. A hidden section may affect spacing, padding, or sibling layout.

### Issue: Dependent conditions missed

**Cause:** Some conditions only apply when a parent condition is true.
**Solution:** In Step 2, map condition dependencies. Note which conditions are nested inside others.

### Issue: Hardcoded values in variant copies

**Cause:** Copying a screen duplicates node IDs and values; modifications introduce raw values.
**Solution:** Step 6 (Token and Component Binding Verification) catches this. Run `batch_get` on every variant and replace hardcoded values with `$variable-name` references.

### Issue: Variant screens are incomplete (missing sections)

**Cause:** Single `batch_design` call ran out of operations, and no follow-up was made.
**Solution:** Use multiple `batch_design` calls. After each call, check progress with `get_screenshot` and continue until the variant is complete. Detail screen variants must have ALL sections.

### Issue: Steps skipped

**Cause:** Model jumps ahead to generation without completing analysis or getting user approval.
**Solution:** Each step has a GATE. You MUST show the deliverable and get user confirmation before proceeding. If you skip a GATE, go back and complete it.

---

## Best Practices

### Start with High-Impact Variants

Generate the states that designers and stakeholders most need to review first: error states, empty states, and role-based differences.

### Keep Variant Diffs Minimal

Each variant should change only what the code condition changes. Don't redesign the entire screen for each variant.

### Annotate with Code References

Adding the actual code condition (`v-if="loading"`) to each variant helps developers and designers trace back to the source.

### Use Copy, Not Rebuild

Always copy the default screen and modify it, rather than building each variant from scratch. This ensures consistency and is faster.

### Respect the Gates

Each step has a GATE (🚫). You MUST show the deliverable to the user and wait for their confirmation before proceeding. This prevents skipping steps and ensures quality at each stage.

### Verify Token Binding After Generation

After generating all variants, always run the token and component binding verification (Step 6). Copying screens and modifying them often introduces hardcoded values that need to be replaced with token references.
