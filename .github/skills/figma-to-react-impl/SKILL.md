---
name: figma-to-react-impl
description: 'Implement a Figma design as production-ready React code using the project design system. Use when: a Figma URL is provided, "implement design", "convert Figma to code", "generate from Figma".'
---

# Figma-to-React Implementation

Convert a Figma design into production-ready React components using `@am92/react-design-system`.

Read `GUIDELINES.md` for repository coding standards, then
`.github/figma-project-profile.md` for local paths and conventions.

## Instruction Precedence

Use this implementation guidance through `@FigmaToReact`, after its lifecycle
gates pass. It cannot override user intent, manager approval, repository
conventions and validation, or the Figma token map.

## When to Use

- Developer pastes a Figma frame/component URL
- "Implement this design", "convert to code", "generate component from Figma"
- Any request involving a `figma.com/design/` URL

## Procedure

### Step 1: Parse the Figma URL

Extract `fileKey` and `nodeId` from the URL:

```
figma.com/design/:fileKey/:fileName?node-id=:nodeId
```

- Convert `-` to `:` in `nodeId` (e.g., `2397-120172` → `2397:120172`)
- For branch URLs: `figma.com/design/:fileKey/branch/:branchKey/:fileName` → use `branchKey` as fileKey

### Step 2: Fetch Design Context

Call the Figma MCP tool `get_design_context` with:

- `fileKey`: extracted from URL
- `nodeId`: extracted from URL (with `:` separator)
- `clientFrameworks`: `react`
- `clientLanguages`: `typescript`

This returns:

- Reference React+Subzero(Custom Design system uses @am92/react-design-system) code (REFERENCE ONLY — do not use directly)
- Screenshot of the design node
- Design tokens and styles used
- Code Connect mappings (if the component is already mapped)

### Step 3: Analyze the Design

From the response, identify:

1. **Component hierarchy** — parent/child relationships, nesting depth
2. **Layout** — flex direction, alignment, wrapping, gaps
3. **Typography** — font family, weight, size, line-height → map to DS variant
4. **Colors** — map hex values to `var(--ds-colour-*)` tokens (see token reference below)
5. **Spacing** — map pixel values to `var(--ds-spacing-*)` tokens (see token reference below)
6. **Border radius** — map to `var(--ds-radius-*)` tokens (see token reference below)
7. **Interactive states** — selected, disabled, hover, pressed
8. **Data patterns** — repeated items suggesting lists/maps
9. **Responsive hints** — width constraints, breakpoints

### Step 4: Map to Design System

Map every visual element to `@am92/react-design-system` components:

| Figma Element             | DS Component                      |
| ------------------------- | --------------------------------- |
| Rectangle/Frame container | `DsBox`                           |
| Auto-layout vertical      | `DsStack` (direction column)      |
| Auto-layout horizontal    | `DsStack` (direction row)         |
| Grid layout               | `DsGrid`                          |
| Text layer                | `DsTypography`                    |
| Button                    | `DsButton` / `DsIconButton`       |
| Input field               | `DsTextField` / `DsTextArea`      |
| Dropdown                  | `DsSelect`                        |
| Checkbox                  | `DsCheckbox`                      |
| Radio button              | `DsRadio`                         |
| Toggle                    | `DsSwitch`                        |
| Card                      | `DsBox` with elevation/border     |
| Modal/Dialog              | `DsDialog`                        |
| Bottom sheet              | `DsBottomSheet`                   |
| Tab bar                   | `DsTabs` / `DsTab` / `DsTabPanel` |
| List                      | `DsList` / `DsListItem`           |
| Navigation                | `DsAppBar` / `DsDrawer`           |
| Tag/Chip                  | `DsTag`                           |
| Icon                      | `DsRemixIcon`                     |
| Divider/Line              | `DsDivider`                       |
| Image                     | `DsImage` or `<img>`              |
| Progress bar              | `DsLinearProgress`                |
| Spinner                   | `DsCircularProgress`              |
| Accordion                 | `DsAccordion`                     |
| Tooltip                   | `DsTooltip`                       |
| Badge                     | `DsBadge`                         |

If no DS equivalent exists:

- Prefer extending DS primitives (`DsBox` with `sx`) over creating from scratch
- Only create custom components as a last resort

### Step 5: Design Token Reference

Use these tokens when converting Figma pixel/hex values to code. **NEVER hardcode values.**

#### Spacing

| Pixel | Variable                        |
| ----- | ------------------------------- |
| 0px   | `var(--ds-spacing-zero)`        |
| 2px   | `var(--ds-spacing-deepFreeze)`  |
| 4px   | `var(--ds-spacing-quickFreeze)` |
| 8px   | `var(--ds-spacing-glacial)`     |
| 12px  | `var(--ds-spacing-frostbite)`   |
| 16px  | `var(--ds-spacing-bitterCold)`  |
| 20px  | `var(--ds-spacing-cool)`        |
| 24px  | `var(--ds-spacing-mild)`        |
| 28px  | `var(--ds-spacing-pleasant)`    |
| 32px  | `var(--ds-spacing-warm)`        |
| 36px  | `var(--ds-spacing-tepid)`       |
| 40px  | `var(--ds-spacing-tropical)`    |
| 44px  | `var(--ds-spacing-hot)`         |
| 48px  | `var(--ds-spacing-blazing)`     |
| 64px  | `var(--ds-spacing-molten)`      |
| 80px  | `var(--ds-spacing-superheated)` |

#### Colors (always use CSS variables, never hex)

**Actions (clickable):**

- `var(--ds-colour-actionPrimary)` → #97144D
- `var(--ds-colour-actionSecondary)` → #ED1164
- `var(--ds-colour-actionTertiary)` → #0C746C

**Surfaces (backgrounds):**

- `var(--ds-colour-surfaceBackground)` → #FFFFFF (light) / #282828 (dark)
- `var(--ds-colour-surfacePrimary)` → #FFFFFF (light) / #000000 (dark)
- `var(--ds-colour-surfaceSecondary)` → #F9F9F9 (light) / #404040 (dark)

**Typography:**

- `var(--ds-colour-typoPrimary)` → #282828 (light) / #F9F9F9 (dark)
- `var(--ds-colour-typoSecondary)` → #575757 (light) / #E2E2E2 (dark)
- `var(--ds-colour-typoTertiary)` → #6E6E6E (light) / #B4B4B4 (dark)
- `var(--ds-colour-typoDisabled)` → #B4B4B4 (light) / #9D9D9D (dark)
- `var(--ds-colour-typoOnSurface)` → #FFFFFF

**Strokes (borders):**

- `var(--ds-colour-strokeDefault)` → #E2E2E2 (light) / #6E6E6E (dark)
- `var(--ds-colour-strokeSelected)` → #F14687 (light) / #F57BA9 (dark)
- `var(--ds-colour-strokeHover)` → #81C1BD (light) / #B8DDDB (dark)
- `var(--ds-colour-strokeDisabled)` → #B4B4B4 (light) / #9D9D9D (dark)

**Icons:**

- `var(--ds-colour-iconActionPrimary)` → #97144D
- `var(--ds-colour-iconDefault)` → #FFFFFF
- `var(--ds-colour-iconNegative)` → #EB0000 / #FF6C6C
- `var(--ds-colour-iconPositive)` → #278829 / #62D264
- `var(--ds-colour-iconWarning)` → #D84008 / #F87647

**Support (status):**

- `var(--ds-colour-supportNegative)` → #EB0000 / #FF6C6C
- `var(--ds-colour-supportPositive)` → #278829 / #62D264
- `var(--ds-colour-supportWarning)` → #D84008 / #F87647

#### Border Radius

| Value | Variable                       |
| ----- | ------------------------------ |
| 0px   | `var(--ds-radius-zero)`        |
| 2px   | `var(--ds-radius-deepFreeze)`  |
| 4px   | `var(--ds-radius-quickFreeze)` |
| 8px   | `var(--ds-radius-glacial)`     |
| 12px  | `var(--ds-radius-frostbite)`   |
| 16px  | `var(--ds-radius-bitterCold)`  |
| 20px  | `var(--ds-radius-cool)`        |
| 24px  | `var(--ds-radius-mild)`        |

#### Styling with `sx` prop

```tsx
// Use DS variables — NEVER hardcode pixel/hex values
<DsBox sx={{ padding: 'var(--ds-spacing-bitterCold)', gap: 'var(--ds-spacing-glacial)' }}>

// Responsive values for mobile/desktop
<DsStack gap={{ xs: 'var(--ds-spacing-glacial)', md: 'var(--ds-spacing-cool)' }}>
```

### Step 6: Search Existing Codebase

Before generating code:

1. Search for similar components already in `src/Components/` or `src/Pages/`
2. Check for existing hooks, utilities, or constants that can be reused
3. Check `src/Redux/` for existing domain slices if data-fetching is involved

### Step 7: Generate Code

Follow these project conventions:

**File Naming:**

- Pages: `<Name>.Page.tsx`
- Components: `<Name>.tsx` (PascalCase)
- Hooks: `use<Name>.ts`
- Constants: `<NAME>.ts` (UPPER_SNAKE_CASE)
- Types: `<Name>.types.ts`
- Services: `<name>.Service.ts`
- Utilities: `<name>.util.ts`

**Component Template:**

```tsx
import type { FC } from 'react'
import { DsBox, DsTypography } from '@am92/react-design-system'

import type { I<ComponentName>Props } from './<ComponentName>.types'

const <ComponentName>: FC<I<ComponentName>Props> = ({ prop1, prop2 }) => {
  return (
    <DsBox sx={{ padding: 'var(--ds-spacing-bitterCold)' }}>
      <DsTypography variant="bodyRegularMedium">
        {prop1}
      </DsTypography>
    </DsBox>
  )
}

export default <ComponentName>
```

**Key Rules:**

- **Max 200 lines per file** — if a component exceeds 200 lines, split it into smaller sub-components in a `Components/` subfolder. Each sub-component should be a focused, single-responsibility unit (e.g., header, list item, footer, form section). The parent component should compose these sub-components.
- **All handlers in custom hooks** — Components must NOT define event handlers, callbacks, or business logic inline. All handler definitions (`handleClick`, `handleSubmit`, `handleBack`, etc.) MUST live in a custom hook (`use<PageName>.ts` or `use<ComponentName>.ts` in a `Hooks/` folder). The component simply imports the hook and binds the returned functions to event props. Components are purely presentational — they render UI and delegate all logic to hooks.
- Use `sx` prop for styling with DS CSS variables
- Use `variant` prop on `DsTypography` for text styling
- Use `DsStack` for flex layouts (gap, direction, alignment)
- Use `DsBox` as a general container
- Type all props with `interface I<Name>Props`
- Use `import type` for type-only imports
- Constants in separate files, UPPER_SNAKE_CASE
- Use `useCallback` inside hooks for handlers passed to child components
- Use `useMemo` for expensive computations
- Lazy load route-level pages with `React.lazy`
- Use `React.memo` for pure presentational components with frequent re-renders
- Include `aria-*` attributes for accessibility
- Use semantic HTML elements

**Responsive Design:**

- Use `withBreakpoints` HOC for responsive components
- Use `getDeviceActiveScreen(breakpoints)` for conditional rendering
- Apply responsive `sx` values: `{ xs: '...', md: '...' }`

### Step 8: Handle Routes (if applicable)

If the design is a new page:

1. Read `.github/figma-project-profile.md` to locate the route constants and router configuration.
2. Inspect the existing route pattern before adding a route constant, lazy import, or route definition.

### Step 9: Handle Data Patterns (if applicable)

If the design shows data-driven UI (lists from API, user data, etc.):

1. Create typed interfaces for API response
2. Read `.github/figma-project-profile.md` and inspect the target Redux domain before choosing service and selector locations
3. Use the typed Redux hooks and selector pattern defined by the profile or established by the target domain

## Output Format

Present results as:

1. **Design Analysis** (2-3 sentences) — what the design shows
2. **Component Mapping** — table of Figma elements → DS components
3. **Generated Files** — complete, runnable code for each file
4. **Integration Notes** — any wiring needed (routes, state, etc.)

### Step 10: Post-Generation Validation

After ALL files are generated, run these checks in order:

#### 10a. Corruption Check

Read back each generated file and verify there are no garbled/repeated characters. If any file content appears corrupted (repeated fragments, missing closing tags, unreadable sequences), delete and regenerate that file.

#### 10b. Prettier

Run Prettier on all generated files:

```bash
npx prettier --write "<path-to-generated-folder>/**/*.{ts,tsx}"
```

If any file is reformatted, that's fine — Prettier is the source of truth for formatting.

#### 10c. ESLint

Run ESLint on all generated files:

```bash
npx eslint "<path-to-generated-folder>/**/*.{ts,tsx}"
```

If errors are reported, fix them in-place. Common issues:

- Import sorting (`Run autofix to sort these imports!`) — run `npx eslint --fix`
- Unused variables — remove them
- Missing types — add explicit types

#### 10d. TypeScript Check

Verify no compile errors exist on the generated files using the IDE error checker. Fix any type mismatches, missing imports, or incorrect prop types.

#### 10e. CodeAuditor Review

Invoke the **CodeAuditor** agent to review all generated files against project guidelines:

```
Review these newly generated files against GUIDELINES.md, .github/figma-project-profile.md, and .github/review-prompt.md:
<list all generated file paths>
Provide findings grouped by severity. Focus on: hardcoded values, missing design tokens, incorrect DS component usage, missing accessibility attributes, and performance issues.
```

Fix all 🔴 Error findings immediately. Fix 🟡 Warnings if the fix is straightforward. Note 🔵 Info items but do not fix unless trivial.

#### 10f. Final Summary

After all checks pass, present a summary:

| Check       | Status              |
| ----------- | ------------------- |
| Corruption  | ✅ Clean            |
| Prettier    | ✅ Formatted        |
| ESLint      | ✅ No errors        |
| TypeScript  | ✅ No errors        |
| CodeAuditor | ✅ All errors fixed |

## Critical Constraints

- **Context window limit (80%)** — Monitor token usage throughout the process. If accumulated input+output tokens exceed 80% of the agent's context window, **STOP immediately** and inform the user: _"⚠️ Context window limit reached. Please select smaller Figma frames (individual sections or components) instead of an entire page, and run the agent separately for each."_ Do not attempt to continue generating code past this threshold — output quality degrades and files may be corrupted.
- **NO Tailwind** — this project does not use Tailwind CSS
- **NO hardcoded values** — always use `var(--ds-spacing-*)`, `var(--ds-colour-*)`, `var(--ds-radius-*)`
- **NO raw HTML** when a DS component exists — always prefer `Ds*` components
- **NO `any` type** — use proper TypeScript typing
- The Figma MCP returns React+ Subzero(Custom Design System which uses MUI under the hood) reference code — this must be FULLY ADAPTED to the project stack
- If Code Connect snippets are present, use the mapped component directly from the codebase
