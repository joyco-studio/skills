---
name: markdown-content
description: >
  Guide for building markdown content pages in Next.js using fumadocs-mdx, rehype-pretty-code,
  and shiki. Covers the full content stack: MDX collections, prose typography, syntax-highlighted
  code blocks, package manager command tabs, and the component mappings that keep rendered
  markdown and custom React code components visually consistent. Based on the patterns used
  in the JOYCO Hub (joyco-studio/hub).
license: MIT
metadata:
  author: joyco-studio
  version: "0.0.1"
---

# Markdown Content Pages — Next.js

This skill defines how to build content-driven pages (docs, blog posts, changelogs, component pages) in a Next.js App Router project using MDX. It captures the exact stack, configuration, and component patterns used in the JOYCO Hub.

The goal is **visual and behavioral consistency** between:

- Markdown codeblocks written in `.mdx` files (processed at build time)
- Custom React code components rendered on the page (highlighted at runtime)

---

## Stack Overview

| Layer | Library | Purpose |
|---|---|---|
| Content framework | `fumadocs-mdx` | MDX collection definitions, frontmatter schemas, build-time processing |
| Content runtime | `fumadocs-core` | Source loader, page tree, search, breadcrumbs |
| UI primitives | `fumadocs-ui` | Layout shells, TOC, sidebar, default MDX component base |
| Syntax highlighting | `shiki` + `rehype-pretty-code` | Dual-theme code highlighting at build time (MDX) and runtime (components) |
| Schema validation | `zod` | Typed frontmatter with defaults and optional fields |
| Styling | Tailwind CSS v4 | Utility-first styles, custom prose CSS, code block styles |
| Framework | Next.js (App Router) | File-based routing with catch-all `[[...slug]]` for content pages |

Install the core content dependencies:

```bash
npm install fumadocs-mdx fumadocs-core fumadocs-ui shiki rehype-pretty-code zod
```

---

## Project Structure

```
├── app/
│   ├── layout.tsx                       # Root layout with RootProvider from fumadocs-ui
│   ├── styles/
│   │   ├── globals.css                  # Tailwind imports + custom stylesheet imports
│   │   ├── prose.css                    # Typography overrides for .prose class
│   │   ├── shiki.css                    # Code block styling (line numbers, highlights, themes)
│   │   └── theming.css                  # Color scheme variables (light/dark/custom themes)
│   └── (docs)/                          # Route group for content pages
│       ├── layout.tsx                   # Docs layout with sidebar, navigation
│       └── [[...slug]]/
│           └── page.tsx                 # Catch-all page rendering MDX content
├── content/
│   ├── meta.json                        # Top-level page ordering
│   ├── index.mdx                        # Landing page
│   └── <category>/                      # e.g. components/, guides/, blog/
│       ├── meta.json                    # Category page ordering
│       └── <page>.mdx                   # Individual content pages
├── components/
│   ├── code-block.tsx                   # Single code block with copy/download
│   ├── code-block-tabs.tsx              # Multi-file tabbed code block
│   ├── code-block-cmd.tsx               # Package manager command tabs
│   ├── copy-button.tsx                  # Clipboard copy with visual feedback
│   ├── package-manager-command.tsx       # npm/yarn/pnpm/bun switcher
│   └── code-source.tsx                  # Server component that reads + highlights source files
├── lib/
│   ├── source.ts                        # fumadocs-core source loader configuration
│   ├── shiki.ts                         # Shiki config, transformers, language map, highlightCode()
│   └── cn.ts                            # Class name utility (clsx + tailwind-merge)
├── mdx-components.tsx                   # Global MDX component mappings
└── source.config.ts                     # fumadocs-mdx collection + rehype pipeline config
```

---

## Content Collection Setup

### `source.config.ts`

This is the entry point for fumadocs-mdx. It defines:

1. **The content collection** with a Zod-extended frontmatter schema
2. **The MDX processing pipeline** with rehype-pretty-code replacing the default highlighter

```ts
import lastModified from 'fumadocs-mdx/plugins/last-modified'
import rehypePrettyCode from 'rehype-pretty-code'
import type { Element } from 'hast'
import { z } from 'zod'
import {
  defineConfig,
  defineDocs,
  frontmatterSchema,
  metaSchema,
} from 'fumadocs-mdx/config'
import { transformers } from './lib/shiki'
import { cn } from './lib/utils'

export const docs = defineDocs({
  dir: 'content',
  docs: {
    schema: frontmatterSchema.extend({
      author: z.string().optional(),
      maintainers: z.array(z.string()).default([]),
      // Add your own custom frontmatter fields here
    }),
  },
  meta: {
    schema: metaSchema,
  },
})

export default defineConfig({
  mdxOptions: {
    rehypePlugins: (plugins) => {
      // Remove the default fumadocs syntax highlighter
      plugins.shift()
      // Add rehype-pretty-code with shiki dual themes
      plugins.push([
        rehypePrettyCode,
        {
          theme: {
            dark: 'github-dark',
            light: 'github-light-default',
          },
          transformers,
          onVisitTitle(node: Element) {
            node.properties['class'] = cn(
              'not-prose',
              node.properties['class']?.toString()
            )
          },
        },
      ])
      return plugins
    },
  },
  plugins: [lastModified()],
})
```

Key decisions:

- **`plugins.shift()`** removes the built-in fumadocs syntax highlighter so rehype-pretty-code takes full control.
- **Dual themes** (`github-dark` + `github-light-default`) generate both light and dark markup. CSS toggles visibility.
- **`transformers`** inject raw code text and command detection metadata into the AST.
- **`onVisitTitle`** adds `not-prose` so code block titles are not affected by prose typography.

### `next.config.ts`

Wrap your Next.js config with fumadocs-mdx:

```ts
import { createMDX } from 'fumadocs-mdx/next'

const withMDX = createMDX()

const config = {
  // your Next.js config
}

export default withMDX(config)
```

### `lib/source.ts`

The runtime source loader that makes content pages queryable:

```ts
import { docs } from 'fumadocs-mdx:collections/server'
import { type InferPageType, loader } from 'fumadocs-core/source'

export const source = loader({
  baseUrl: '/',
  source: docs.toFumadocsSource(),
})

export type Page = InferPageType<typeof source>
```

---

## Content File Format

### MDX Frontmatter

Every `.mdx` file starts with YAML frontmatter:

```mdx
---
title: Action Hint
description: A component that displays contextual action hints.
author: joyco
maintainers:
  - joyco
---

import { ComponentPreview } from '@/components/preview/component-preview'

<ComponentPreview name="action-hint" />

## Installation

```bash
npx shadcn@latest add "https://registry.joyco.studio/r/action-hint"
```

## Usage

```tsx
import { ActionHintEmitter, useActionHint } from '@/components/ui/action-hint'
```
```

Rules for content files:

- Frontmatter fields are validated by the Zod schema in `source.config.ts`.
- Imports go right after the closing `---` of frontmatter.
- React components can be used inline alongside standard markdown.
- Code blocks use standard markdown fenced syntax with language tags.
- Code blocks with a `title="filename.tsx"` meta string get a title bar.

### `meta.json`

Controls page ordering within a directory:

```json
{
  "pages": ["index", "action-hint", "typewriter", "marquee"]
}
```

---

## Catch-All Content Page

The `[[...slug]]/page.tsx` file resolves slugs to content pages and renders them:

```tsx
import { source } from '@/lib/source'
import { getMDXComponents } from '@/mdx-components'
import { notFound } from 'next/navigation'

export default async function Page(props: {
  params: Promise<{ slug?: string[] }>
}) {
  const params = await props.params
  const page = source.getPage(params.slug)
  if (!page) notFound()

  const MDX = page.data.body

  return (
    <article className="prose">
      <h1>{page.data.title}</h1>
      <p className="text-fd-muted-foreground">{page.data.description}</p>
      <MDX components={getMDXComponents()} />
    </article>
  )
}

export function generateStaticParams() {
  return source.generateParams()
}
```

---

## Shiki Configuration

### `lib/shiki.ts`

This module centralizes all syntax highlighting configuration. It is used by **both** the MDX pipeline (build-time) and custom code components (runtime).

#### Language Map

Maps file extensions to shiki language identifiers:

```ts
import { codeToHtml, type ShikiTransformer, BuiltinLanguage } from 'shiki'

export const LANGUAGE_MAP: Record<string, BuiltinLanguage> = {
  json: 'json',
  js: 'javascript',
  ts: 'typescript',
  tsx: 'tsx',
  jsx: 'jsx',
  md: 'markdown',
  mdx: 'mdx',
  css: 'css',
  html: 'html',
  yml: 'yaml',
  yaml: 'yaml',
  sh: 'bash',
  bash: 'bash',
  py: 'python',
}

export function getLanguageFromExtension(ext: string): string {
  return LANGUAGE_MAP[ext] || ext || 'text'
}
```

#### Command Detection

Detects package manager install commands and generates per-manager variants. This powers the automatic command tabs in MDX code blocks:

```ts
const commandDetectors: ({
  match: (raw: string) => boolean
  transform: (raw: string) => Record<string, string>
})[] = [
  {
    match: (raw) => raw.startsWith('npm install'),
    transform: (raw) => ({
      __npm__: raw,
      __yarn__: raw.replace('npm install', 'yarn add'),
      __pnpm__: raw.replace('npm install', 'pnpm add'),
      __bun__: raw.replace('npm install', 'bun add'),
    }),
  },
  {
    match: (raw) => raw.startsWith('npx'),
    transform: (raw) => ({
      __npm__: raw,
      __yarn__: raw.replace('npx', 'yarn'),
      __pnpm__: raw.replace('npx', 'pnpm dlx'),
      __bun__: raw.replace('npx', 'bunx --bun'),
    }),
  },
  {
    match: (raw) => raw.startsWith('npm run'),
    transform: (raw) => ({
      __npm__: raw,
      __yarn__: raw.replace('npm run', 'yarn'),
      __pnpm__: raw.replace('npm run', 'pnpm'),
      __bun__: raw.replace('npm run', 'bun'),
    }),
  },
]

export function isCommand(raw: string): boolean {
  return commandDetectors.some((cmd) => cmd.match(raw))
}
```

#### Shiki Transformers (MDX Level)

These transformers run during MDX compilation. They attach metadata to the AST that the `mdx-components.tsx` mappings read at render time:

```ts
export const transformers = [
  {
    pre(node) {
      const raw = this.source
      node.properties['__raw__'] = raw
      node.properties['__iscommand__'] = String(isCommand(raw))
    },
    code(node) {
      if (node.tagName === 'code') {
        const raw = this.source
        node.properties['__raw__'] = raw
        node.properties['__iscommand__'] = String(isCommand(raw))
        applyCommandTransformations(raw, node.properties)
      }
    },
  },
] as ShikiTransformer[]
```

Metadata injected:

- `__raw__` — the original unprocessed code string (used by `CopyButton`)
- `__iscommand__` — `"true"` if the code block is a detected command
- `__npm__`, `__yarn__`, `__pnpm__`, `__bun__` — per-manager command variants (when applicable)

#### `highlightCode()` — Runtime Highlighting

Used by custom React code components (not MDX) to produce the same visual output:

```ts
export const codeClasses = {
  pre: "not-fumadocs-codeblock relative w-full overflow-auto p-4 has-[[data-line-numbers]]:px-0",
}

export async function highlightCode(code: string, language: string = 'tsx') {
  const html = await codeToHtml(code, {
    lang: language,
    themes: {
      dark: 'github-dark',
      light: 'github-light-default',
    },
    transformers: [
      {
        code(node) {
          node.properties['data-line-numbers'] = ''
        },
        pre(node) {
          node.properties['class'] = cn(
            codeClasses.pre,
            node.properties['class']?.toString()
          )
        },
        line(node) {
          node.properties['data-line'] = ''
        },
      },
    ],
  })
  return html
}
```

This is the **consistency mechanism** — both the MDX pipeline and `highlightCode()` use the same shiki themes (`github-dark` / `github-light-default`), ensuring identical syntax coloring everywhere.

---

## MDX Component Mappings

### `mdx-components.tsx`

This file maps HTML elements produced by the MDX compiler to custom React components. It is the bridge between raw markdown and the component system.

```tsx
import type { MDXComponents } from 'mdx/types'
import defaultMdxComponents from 'fumadocs-ui/mdx'
import { CodeTabs } from '@/components/code-tabs'
import { FileCodeblock } from '@/components/code-source'
import { ImageCols } from '@/components/image-cols'
import { CopyButton } from '@/components/copy-button'
import { PackageManagerCommand } from '@/components/package-manager-command'
import { cn } from '@/lib/cn'

export function getMDXComponents(): MDXComponents {
  return {
    ...defaultMdxComponents,

    // Custom components available in MDX files
    CodeTabs,
    FileCodeblock,
    ImageCols,

    // Inline code
    code: ({ children, className, ...props }) => {
      // Detect package manager commands from shiki transformer metadata
      const npm = props['__npm__' as keyof typeof props] as string | undefined
      const yarn = props['__yarn__' as keyof typeof props] as string | undefined
      const pnpm = props['__pnpm__' as keyof typeof props] as string | undefined
      const bun = props['__bun__' as keyof typeof props] as string | undefined

      if (npm || yarn || pnpm || bun) {
        return <PackageManagerCommand npm={npm} yarn={yarn} pnpm={pnpm} bun={bun} />
      }

      return (
        <code className={cn('bg-muted rounded px-1 py-0.5', className)} {...props}>
          {children}
        </code>
      )
    },

    // Block code (pre element wrapping highlighted code)
    pre: ({ children, className, ...props }) => {
      const raw = props['__raw__' as keyof typeof props] as string | undefined
      const isCommand = props['__iscommand__' as keyof typeof props] === 'true'

      // Commands are handled by the code element mapping above
      if (isCommand) {
        return <>{children}</>
      }

      return (
        <pre className={cn('relative', className)} {...props}>
          {children}
          {raw && <CopyButton text={raw} />}
        </pre>
      )
    },
  }
}
```

How the flow works:

1. Author writes a fenced code block in `.mdx`
2. rehype-pretty-code + shiki transform it into `<pre><code>` HTML with syntax tokens
3. Shiki transformers attach `__raw__`, `__iscommand__`, `__npm__`, etc. as properties
4. MDX compiles the HTML into React elements, calling `getMDXComponents()` mappings
5. The `code` mapping checks for command metadata and renders `PackageManagerCommand` if detected
6. The `pre` mapping adds a `CopyButton` using the `__raw__` text
7. Non-command code blocks render the shiki-highlighted HTML directly with a copy button overlay

---

## Code Block Components

### `CodeBlock` — Single Code Display

Client component that renders pre-highlighted HTML with copy and download buttons:

```tsx
'use client'

import { CopyButton } from '@/components/copy-button'
import { cn } from '@/lib/cn'

type CodeBlockProps = {
  highlightedCode: string
  language: string
  title?: string
  rawCode?: string
  maxHeight?: number
  wrap?: boolean
}

export function CodeBlock({
  highlightedCode,
  language,
  title,
  rawCode,
  maxHeight,
  wrap,
}: CodeBlockProps) {
  return (
    <figure data-rehype-pretty-code-figure="">
      {title && (
        <figcaption data-rehype-pretty-code-title="">
          {title}
        </figcaption>
      )}
      <div
        className="relative"
        style={maxHeight ? { '--max-height': `${maxHeight}px` } as React.CSSProperties : undefined}
        data-wrap={wrap ? 'true' : undefined}
        dangerouslySetInnerHTML={{ __html: highlightedCode }}
      />
      {rawCode && <CopyButton text={rawCode} />}
    </figure>
  )
}
```

### `CodeBlockTabs` — Multi-file Tabbed Code

Client component with tabs for displaying multiple code files:

```tsx
'use client'

import * as React from 'react'
import { Tabs, TabsList, TabsTrigger, TabsContent } from '@/components/ui/tabs'
import { CopyButton } from '@/components/copy-button'

type CodeTab = {
  filename: string
  highlightedCode: string
  rawCode: string
}

export function CodeBlockTabs({
  tabs,
  defaultTab,
  maxHeight = 400,
}: {
  tabs: CodeTab[]
  defaultTab?: string
  maxHeight?: number
}) {
  const [activeTab, setActiveTab] = React.useState(defaultTab || tabs[0]?.filename)
  if (!tabs.length) return null

  const current = tabs.find((t) => t.filename === activeTab) || tabs[0]

  return (
    <figure data-rehype-pretty-code-figure="">
      <Tabs value={activeTab} onValueChange={setActiveTab}>
        <figcaption>
          <TabsList>
            {tabs.map((tab) => (
              <TabsTrigger key={tab.filename} value={tab.filename}>
                {tab.filename}
              </TabsTrigger>
            ))}
          </TabsList>
          <CopyButton text={current.rawCode} />
        </figcaption>
        {tabs.map((tab) => (
          <TabsContent key={tab.filename} value={tab.filename}>
            <div dangerouslySetInnerHTML={{ __html: tab.highlightedCode }} />
          </TabsContent>
        ))}
      </Tabs>
    </figure>
  )
}
```

### `CodeBlockCommand` — Package Manager Tabs

Renders install commands with tabs for each package manager:

```tsx
'use client'

import * as React from 'react'
import { CopyButton } from '@/components/copy-button'

export type CommandTab = {
  label: string
  content: string
}

export function CodeBlockCommand({
  tabs,
  activeTab,
  onTabChange,
}: {
  tabs: CommandTab[]
  activeTab?: string
  onTabChange?: (tab: string) => void
}) {
  const [selected, setSelected] = React.useState(activeTab || tabs[0]?.label)

  const handleChange = (tab: string) => {
    setSelected(tab)
    onTabChange?.(tab)
  }

  const current = tabs.find((t) => t.label === selected) || tabs[0]

  return (
    <div data-slot="command-block">
      <div className="flex items-center justify-between border-b px-4 py-2">
        <div className="flex gap-2">
          {tabs.map((tab) => (
            <button
              key={tab.label}
              onClick={() => handleChange(tab.label)}
              className={selected === tab.label ? 'font-medium' : 'text-muted-foreground'}
            >
              {tab.label}
            </button>
          ))}
        </div>
        <CopyButton text={current.content} />
      </div>
      <pre className="overflow-x-auto p-4 font-mono text-sm">
        <code>{current.content}</code>
      </pre>
    </div>
  )
}
```

### `CodeSource` (FileCodeblock) — Server-Side Source Display

Reads a source file from disk, highlights it with shiki, and renders a `CodeBlock`. This is a **server component**:

```tsx
import { readFile } from 'fs/promises'
import { join } from 'path'
import { CodeBlock } from '@/components/code-block'
import { highlightCode, getLanguageFromExtension } from '@/lib/shiki'

export async function FileCodeblock({
  filePath,
  title,
  language,
  ...props
}: {
  filePath: string
  title?: string
  language?: string
}) {
  const fullPath = join(process.cwd(), 'public', filePath)
  const content = await readFile(fullPath, 'utf-8')
  const ext = filePath.split('.').pop() || ''
  const lang = language || getLanguageFromExtension(ext)
  const html = await highlightCode(content, lang)

  return (
    <CodeBlock
      highlightedCode={html}
      language={lang}
      title={title || filePath.split('/').pop() || filePath}
      rawCode={content}
      {...props}
    />
  )
}
```

This component is used in MDX files to display actual source code from the project:

```mdx
<FileCodeblock filePath="registry/joyco/blocks/action-hint.tsx" />
```

---

## Copy Button

Shared clipboard component used by every code display:

```tsx
'use client'

import * as React from 'react'
import { Check, Copy } from 'lucide-react'

export function useCopyToClipboard(timeout = 2000) {
  const [copied, setCopied] = React.useState(false)

  const copy = React.useCallback(
    async (text: string) => {
      await navigator.clipboard.writeText(text)
      setCopied(true)
      setTimeout(() => setCopied(false), timeout)
    },
    [timeout]
  )

  return { copied, copy }
}

export function CopyButton({ text }: { text: string }) {
  const { copied, copy } = useCopyToClipboard()

  return (
    <button
      onClick={() => copy(text)}
      className="absolute right-3 top-3 opacity-0 transition-opacity group-hover:opacity-100"
      aria-label="Copy code"
    >
      {copied ? <Check className="size-4" /> : <Copy className="size-4" />}
    </button>
  )
}
```

---

## Prose Typography

### `prose.css`

Minimal overrides on top of fumadocs-ui defaults:

```css
.prose {
  --tw-prose-body: color-mix(in oklab, var(--foreground) 70%, transparent);
}

.prose
  :where(figure[data-rehype-pretty-code-figure]):not(
    :where(.not-prose, .not-prose *)
  ) {
  @apply my-6 first:mt-0 last:mb-0;
}
```

- **Body color** uses `color-mix` to set prose text at 70% of the foreground color, giving a softer reading experience.
- **Code figures** get vertical margin (`my-6`) so they sit comfortably within prose, with first/last child margin resets.

### Applying Prose

The `.prose` class is applied on the article wrapper in the content page:

```tsx
<article className="prose">
  <MDX components={getMDXComponents()} />
</article>
```

fumadocs-ui provides base prose styles. The custom `prose.css` applies targeted overrides.

---

## Code Block Styles

### `shiki.css`

Comprehensive styling for all code blocks — both MDX-rendered and component-rendered.

Key aspects:

**Dual-theme switching:**

```css
.dark [data-rehype-pretty-code-figure] span {
  color: var(--shiki-dark) !important;
}
```

Light theme colors are the default. In dark mode, the CSS swaps to `--shiki-dark` variables that shiki generates alongside the light theme tokens.

**Code block container:**

```css
[data-rehype-pretty-code-figure] {
  background-color: var(--color-code);
  color: var(--color-code-foreground);
  position: relative;
  overflow: hidden;
  border-radius: var(--radius-lg);
  font-size: var(--text-sm);
}
```

**Line numbers (CSS counters):**

```css
[data-line-numbers] {
  counter-reset: line;
}

[data-line-numbers] [data-line]::before {
  counter-increment: line;
  content: counter(line);
  display: inline-block;
  width: 2ch;
  margin-right: 1.5rem;
  text-align: right;
  color: color-mix(in oklab, var(--color-code-foreground) 30%, transparent);
  position: sticky;
  left: 0;
}
```

**Line highlighting:**

```css
[data-highlighted-line] {
  background-color: var(--color-code-highlight);
  border-left: 2px solid var(--foreground);
}
```

**Title bar:**

```css
[data-rehype-pretty-code-title] {
  font-family: var(--font-mono);
  color: color-mix(in oklab, var(--color-code-foreground) 60%, transparent);
  border-bottom: 1px solid color-mix(in oklab, var(--color-code-foreground) 10%, transparent);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
```

**Text wrapping:**

```css
[data-wrap='true'] code {
  white-space: pre-wrap;
  overflow-wrap: anywhere;
}
```

---

## Theming

### `theming.css`

Defines color variables for light, dark, and custom themes using OKLCH color space. Code block colors are part of this system:

```css
:root {
  --color-code: oklch(0.96 0 0);
  --color-code-foreground: oklch(0.3 0 0);
  --color-code-highlight: oklch(0.94 0 0);
}

.dark {
  --color-code: oklch(0.2 0 0);
  --color-code-foreground: oklch(0.9 0 0);
  --color-code-highlight: oklch(0.25 0 0);
}
```

These variables are consumed by both `shiki.css` and the component classes, ensuring code blocks look identical whether rendered from MDX or from React components.

---

## Consistency Model

The entire system is designed so that a code block written in markdown:

````mdx
```tsx title="example.tsx"
const x = 1
```
````

looks **identical** to a code block rendered by a React component:

```tsx
const html = await highlightCode('const x = 1', 'tsx')
return <CodeBlock highlightedCode={html} language="tsx" title="example.tsx" rawCode="const x = 1" />
```

This consistency is achieved through:

1. **Same shiki themes** — Both paths use `github-dark` / `github-light-default`.
2. **Same CSS** — Both produce elements with `[data-rehype-pretty-code-figure]` that `shiki.css` styles.
3. **Same `CopyButton`** — Every code display uses the same copy component.
4. **Same color variables** — `--color-code`, `--color-code-foreground`, `--color-code-highlight` from `theming.css`.
5. **Same data attributes** — `data-line`, `data-line-numbers`, `data-highlighted-line`, `data-wrap` are used by both pipelines.
6. **`codeClasses` constant** — Shared class string for `pre` elements prevents style drift between MDX and component output.

---

## Globals CSS Structure

The main stylesheet imports everything in order:

```css
@import 'tailwindcss';
@import 'fumadocs-ui/css/neutral.css';
@import 'fumadocs-ui/css/animations.css';
@import './shiki.css';
@import './prose.css';
@import './theming.css';
@import './utilities.css';
```

Order matters — theming variables must be defined before they are consumed by shiki and prose styles.

---

## Implementation Checklist

When setting up markdown content pages in a new project:

1. Install dependencies: `fumadocs-mdx`, `fumadocs-core`, `fumadocs-ui`, `shiki`, `rehype-pretty-code`, `zod`
2. Create `source.config.ts` with your frontmatter schema and rehype-pretty-code pipeline
3. Wrap `next.config.ts` with `createMDX()` from fumadocs-mdx
4. Create `lib/source.ts` with the fumadocs-core loader
5. Create `lib/shiki.ts` with language map, command detectors, transformers, and `highlightCode()`
6. Create `mdx-components.tsx` mapping `pre` and `code` to your custom components
7. Create the code block components (`CodeBlock`, `CodeBlockTabs`, `CodeBlockCommand`, `CopyButton`)
8. Set up `prose.css` and `shiki.css` for typography and code block styling
9. Define theme variables in `theming.css` including `--color-code` family
10. Create the content directory structure with `meta.json` files
11. Create the catch-all `[[...slug]]/page.tsx` route
12. Import all stylesheets in `globals.css` in the correct order
