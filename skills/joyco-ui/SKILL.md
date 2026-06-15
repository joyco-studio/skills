---
name: joyco-ui
description: >
  Build interfaces with the JOYCO UI kit — a bento/console-style component
  library on top of shadcn/ui, Radix primitives, and Tailwind CSS. Use when
  building UI in a JOYCO project, when the user imports from `@/components/ui/*`
  or installs `@joyco/ui` / `@joyco/<component>` via shadcn, or mentions
  Cluster, Filler, the bento layout, sliced-corner badges, or the JOYCO design
  system. Covers the Cluster/Filler layout model, the component API reference,
  theme tokens, and styling conventions (`data-slot`, single `className`).
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# JOYCO UI Kit

A bento/console-style component library built on shadcn/ui with Radix primitives and Tailwind CSS. Interfaces are modular blocks that snap together with tight gaps, like a hardware panel or console dashboard.

Use this skill when building or editing UI in a JOYCO project — the giveaways are imports from `@/components/ui/*`, the `Cluster`/`Filler` primitives, sliced-corner badges, or `@joyco/*` shadcn registry installs.

---

## Installation

Full kit, or a single component, via the shadcn registry:

```bash
pnpm dlx shadcn@latest add @joyco/ui
pnpm dlx shadcn@latest add @joyco/button
```

---

## Design philosophy

JOYCO interfaces are **modular and bento-like**. Every piece of UI is a discrete block that snaps together with tight gaps. Two rules follow from this:

1. **No loose spacing.** Never use `justify-between`, `justify-end`, or large gaps/margins to push elements apart. Use `<Filler />` — a visible, decorative spacer that fills remaining space with the cluster's background. Dead space is always intentional and styled.

2. **No conventional cards.** Don't wrap content in a `<div>` with `bg-*` + `p-*` to fake a card. Use `<Cluster>` — a transparent flex container whose *children* inherit a shared background via a CSS custom property. The Cluster has no visual weight; each child owns its appearance.

The base gap is `--gap: 0.25rem` (4px). This tight spacing gives JOYCO interfaces their dense, modular feel. Reference it in Tailwind as `gap-gap`.

---

## Cluster & Filler (layout primitives)

These are the most important components in the system — they replace conventional flex layouts.

### Cluster

A transparent flex container. Children inherit a shared background color.

```tsx
import { Cluster, Filler } from '@/components/ui/cluster'
```

| Prop | Values | Default | Description |
|------|--------|---------|-------------|
| `direction` | `'row'` \| `'col'` | `'row'` | Flex direction |
| `align` | `'start'` \| `'center'` \| `'end'` \| `'stretch'` \| `'baseline'` | `'center'` | Align items |
| `wrap` | `boolean` | `false` | Flex wrap |
| `bg` | `'muted'` \| `'accent'` | `'muted'` | Background applied to children via `--cluster-bg` |
| `display` | `'flex'` \| `'inline-flex'` | `'flex'` | Display type |
| `asChild` | `boolean` | `false` | Merge props onto child element |

Cluster sets `--cluster-bg` on itself, and a global CSS rule propagates it to direct children:

```css
:where([data-slot='cluster'] > *) {
  background-color: var(--cluster-bg, transparent);
}
```

The gap between children (`gap-gap`, 4px) reveals the page background underneath, creating the bento grid effect.

### Filler

A decorative spacer with `flex-1`. It takes up remaining space and receives the cluster's background — a visible element, not empty whitespace. It carries `role="presentation"` and `aria-hidden="true"`.

```tsx
<Cluster>
  <span className="p-3 text-sm font-medium">Title</span>
  <Filler />
  <Button>Action</Button>
</Cluster>
```

**Rule: anywhere you would reach for `justify-between` or `justify-end`, use `<Filler />` instead.**

### Layout examples

Row with title and action:

```tsx
<Cluster>
  <div className="flex items-center gap-2 px-3 py-2">
    <span className="text-sm font-medium">Project Setup</span>
  </div>
  <Filler />
  <Badge variant="accent">Draft</Badge>
</Cluster>
```

Column form layout:

```tsx
<Cluster direction="col" align="stretch" bg="muted">
  <div className="p-3 pt-2">
    <span className="text-sm font-medium">Project Setup</span>
    <p className="text-muted-foreground text-sm">Configure your project settings.</p>
  </div>
  <label className="focus-within:bg-accent/70 flex items-center gap-3 p-3">
    <PresentationIcon className="text-muted-foreground size-4 shrink-0" />
    <input placeholder="Project name" className="min-w-0 text-sm outline-none" />
  </label>
  <Cluster bg="muted">
    <Filler />
    <Button variant="secondary">Cancel</Button>
    <Button>Create</Button>
  </Cluster>
</Cluster>
```

---

## Component reference

For the full per-component API — Button, Badge, Input, InputGroup, ButtonGroup, Textarea, Select, Tabs, Switch, Slider, Separator, Kbd, Avatar, Tooltip, Popover, DropdownMenu, Collapsible — see [`components.md`](./components.md). Open it when you need the exact variants, sizes, sub-components, or import paths for a specific component.

---

## Theme

JOYCO uses the OKLch color space. Key semantic tokens:

| Token | Purpose |
|-------|---------|
| `background` / `foreground` | Page base |
| `primary` / `primary-foreground` | Brand color (purple-blue) for CTAs |
| `secondary` / `secondary-foreground` | Secondary actions |
| `muted` / `muted-foreground` | Subdued surfaces and text |
| `accent` / `accent-foreground` | Highlighted surfaces |
| `destructive` | Danger/delete actions |
| `border` | Borders |
| `input` | Input borders |
| `ring` | Focus rings |
| `card` / `popover` | Elevated surfaces |

**Typography:** Public Sans (sans), Roboto Mono (mono). Headings use decreasing letter-spacing. Use `font-mono uppercase tracking-wide` for labels and badges.

**Focus states:** all interactive elements use `focus-visible:ring-ring/50 focus-visible:ring-[3px]`. Never remove outlines without a focus replacement.

---

## Styling conventions

- **Single `className` on root.** Components accept one `className` prop. Style internals from outside using `data-slot` selectors — don't add multiple `*ClassName` props.
- **`data-slot` attributes.** Every component part has a `data-slot` (e.g. `data-slot="button"`, `data-slot="badge"`, `data-slot="cluster"`). Target with `**:data-[slot=button]:bg-red-500`.
- **`data-variant` / `data-size`.** Buttons expose these for conditional parent styling.
- **Responsive hiding.** Use `max-{breakpoint}:hidden` rather than hiding by default and revealing at a larger breakpoint.
- **Hover states.** Every interactive element needs a `hover:` state. Use `hocus:` (custom variant for `:hover` + `:focus-visible`) where available.

---

## Pitfalls

- Reaching for `justify-between`/`justify-end`/large margins instead of `<Filler />`. This is the most common mistake — it breaks the bento look.
- Building a card as a `<div>` with `bg-*` + `p-*` instead of composing a `<Cluster>`.
- Adding per-internal `className` props instead of styling via `data-slot` from the parent.
- Icon-only buttons without `aria-label`.
- Removing focus outlines (`outline-none`) without a `focus-visible` replacement.

---

## Checklist

- [ ] Layout uses `Cluster` + `Filler`, not `justify-*` or ad-hoc cards.
- [ ] Gaps reference `gap-gap` (4px base) for the bento spacing.
- [ ] Internals styled via `data-slot` from the parent, single `className` on the root.
- [ ] Correct component variant/size pulled from [`components.md`](./components.md).
- [ ] Icon-only buttons have `aria-label`; interactive elements have visible focus.
- [ ] Semantic theme tokens (`muted`, `accent`, `primary`…) instead of hardcoded colors.
