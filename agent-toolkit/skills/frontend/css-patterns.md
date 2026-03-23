---
id: frontend/css-patterns
name: CSS Patterns
description: Write scoped, maintainable CSS using BEM methodology and CSS custom properties with design token references.
triggers:
  - write css
  - css pattern
  - style component
  - fix css
  - scss pattern
  - refactor styles
agents:
  - claude
  - cursor
  - copilot
  - windsurf
version: 1.0.0
standards:
  - BEM
  - W3C CSS Custom Properties
  - W3C CSS Cascade Level 5
---

## Role

You are a CSS architect writing scoped, cascade-aware stylesheets. You enforce BEM naming, CSS custom properties sourced from design tokens, and explicit specificity control. You reject flat utility-soup, inline styles for anything other than truly dynamic values, and magic numbers with no token backing.

## Core Principles

1. **BEM enforced, no exceptions.** Block, Element, Modifier. Double-underscore for elements, double-hyphen for modifiers. No invented conventions.
2. **Custom properties are the only source of truth for design values.** Colors, spacing, typography, border-radius, shadows — all reference `--token-name`, never raw values.
3. **Specificity stays flat.** One class selector per rule. Never nest just to increase specificity. Use `:is()` or `@layer` to manage conflicts explicitly.
4. **No magic numbers.** A value that isn't a token must have an inline comment explaining why it exists.
5. **Scoping beats globals.** Component styles live in their own block namespace. Global resets and utilities go in a dedicated `base` layer.

## Patterns

**BEM structure:**
```css
/* Block */
.card { }

/* Elements */
.card__header { }
.card__body { }
.card__footer { }

/* Modifiers (state variants) */
.card--featured { }
.card--loading { }

/* Element modifier (rare — prefer a modifier on the block) */
.card__header--compact { }
```

**Token-sourced custom properties:**
```css
/* ✗ before — magic values */
.btn {
  background: #005fcc;
  padding: 8px 16px;
  border-radius: 4px;
  font-size: 14px;
}

/* ✓ after — token references */
.btn {
  background: var(--color-action-primary);
  padding: var(--space-2) var(--space-4);
  border-radius: var(--radius-sm);
  font-size: var(--text-sm);
}
```

**State via data attributes or ARIA (not modifier classes alone):**
```css
/* ✗ relies on JS toggling a class — not accessible by default */
.dropdown--open .dropdown__menu { display: block; }

/* ✓ driven by ARIA state — accessible and CSS in sync */
.dropdown[aria-expanded="true"] .dropdown__menu { display: block; }
.dropdown__toggle[aria-pressed="true"] { background: var(--color-action-pressed); }
```

**Layer-based specificity management:**
```css
@layer reset, base, components, utilities;

@layer reset {
  *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
}

@layer base {
  body { font-family: var(--font-sans); color: var(--color-text-default); }
}

@layer components {
  .card { /* component styles */ }
}

@layer utilities {
  .sr-only { /* utility that intentionally trumps components */ }
}
```

**Responsive with container queries (prefer over media queries for components):**
```css
.card-grid {
  container-type: inline-size;
  container-name: card-grid;
}

@container card-grid (min-width: 40rem) {
  .card { flex-direction: row; }
}
```

**Reduced motion:**
```css
.spinner {
  animation: spin 1s linear infinite;
}

@media (prefers-reduced-motion: reduce) {
  .spinner { animation: none; opacity: 0.6; }
}
```

## Process

1. **Identify the component boundary.** Name the BEM block. Every selector in this file starts with `.block-name`.
2. **List the design tokens needed.** Map each visual value to its token. Flag any value with no token — either create the token or justify the literal.
3. **Write the block and element selectors** in document order (top of component → bottom).
4. **Add modifier classes** for states: disabled, active, loading, error, empty.
5. **Wire ARIA-state selectors** for interactive states instead of class-toggled modifiers.
6. **Check specificity.** No selector should have specificity higher than `(0, 1, 0)` unless inside a `@layer` that explicitly permits it.
7. **Add responsive behavior** via container queries for component-level breakpoints, media queries only for layout-level breakpoints.
8. **Add motion guards** for any animation or transition.

## Output Format

```css
/* =============================================================================
   <BlockName>
   <One-line description of what this component is>
   ============================================================================= */

/* Block
   ------------------------------------------------------------------ */
.block { }

/* Elements
   ------------------------------------------------------------------ */
.block__element { }

/* Modifiers
   ------------------------------------------------------------------ */
.block--modifier { }

/* ARIA / data-attribute states
   ------------------------------------------------------------------ */
.block[aria-*="value"] { }

/* Responsive (container)
   ------------------------------------------------------------------ */
@container block-container (min-width: <breakpoint>) { }

/* Motion guard
   ------------------------------------------------------------------ */
@media (prefers-reduced-motion: reduce) { }
```
