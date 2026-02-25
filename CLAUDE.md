# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
bun dev          # Start dev server (Next.js with Turbopack)
bun run build    # Production build
bun run start    # Start production server
bun run lint     # Run ESLint
```

Package manager is **Bun** — use `bun add` / `bun remove` for dependencies.

Add new shadcn components via: `bunx shadcn@latest add <component-name>`

## Architecture

Next.js 16 App Router project using React 19 with Server Components enabled. This is a UI component tutorial/showcase built on shadcn/ui.

### Key Stack

- **Styling**: Tailwind CSS 4 via PostCSS, with OKLch CSS variables for theming (light/dark mode via `.dark` class)
- **UI Components**: shadcn/ui (radix-mira style) backed by Radix UI and @base-ui/react
- **Component Variants**: Class Variance Authority (CVA) for type-safe variant props
- **Icons**: Phosphor Icons (`@phosphor-icons/react`)
- **Utility**: `cn()` helper in `lib/utils.ts` — merges classnames via clsx + tailwind-merge

### Path Aliases

`@/*` maps to project root (e.g., `@/components/ui/button`, `@/lib/utils`).

### Component Patterns

- **Compound components**: Card, AlertDialog, DropdownMenu etc. export multiple sub-components (e.g., `Card`, `CardHeader`, `CardContent`)
- **`data-slot` attributes**: All UI components use `data-slot="name"` for styling hooks
- **`asChild` / Slot pattern**: Components support render delegation via Radix's Slot
- **FieldGroup system**: Form fields use `FieldGroup > Field > FieldLabel/FieldContent/FieldError` composition with orientation variants (vertical/horizontal/responsive)

### Directory Layout

- `app/` — Next.js App Router pages and layouts
- `components/ui/` — shadcn UI primitives (button, card, input, select, etc.)
- `components/` — App-level components that compose UI primitives
- `lib/utils.ts` — Shared utilities (`cn()`)

### shadcn Configuration

`components.json` defines the shadcn setup: radix-mira style, Phosphor icons, CSS variables enabled, RSC mode on. New components are generated into `components/ui/`.
