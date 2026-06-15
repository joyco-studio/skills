# JOYCO UI — Component Reference

Full per-component API. All components use `data-slot` attributes for CSS targeting; style internals from the parent with `**:data-[slot=name]:styles` rather than adding multiple `className` props.

For the layout primitives (`Cluster`, `Filler`), the design philosophy, theme tokens, and styling conventions, see [`SKILL.md`](./SKILL.md).

---

## Button

```tsx
import { Button } from '@/components/ui/button'
```

**Variants:** `default` | `secondary` | `muted` | `accent` | `outline` | `ghost` | `destructive` | `link`

**Sizes:** `default` (h-9) | `sm` (h-8) | `lg` (h-10) | `icon` (size-9) | `icon-sm` (size-8) | `icon-lg` (size-10)

```tsx
<Button variant="default">Save</Button>
<Button variant="outline" size="sm">Cancel</Button>
<Button size="icon" aria-label="Settings"><SettingsIcon /></Button>
<Button variant="destructive"><TrashIcon /> Delete</Button>
```

Icon-only buttons always need `aria-label`. Icons auto-size to `size-4` unless given an explicit `size-*` class.

---

## Badge

```tsx
import { Badge } from '@/components/ui/badge'
```

**Variants:** `default` | `secondary` | `muted` | `accent` | `card` | `outline` | `destructive` | `key`

**Sizes:** `default` | `sm`

Badges have sliced corners by default (clip-path). Disable with `slicedCorners={false}`. Text is monospace, uppercase, with tight tracking.

```tsx
<Badge variant="accent">New</Badge>
<Badge variant="key">⌘K</Badge>
```

---

## Input

```tsx
import { Input } from '@/components/ui/input'
```

Standard text input with border, focus ring, and dark mode tint. Use correct `type` attributes (`email`, `tel`, `url`, `number`).

```tsx
<Input placeholder="Email address" type="email" />
<Input placeholder="Disabled" disabled />
<Input placeholder="Invalid" aria-invalid="true" />
```

---

## InputGroup

```tsx
import {
  InputGroup, InputGroupAddon, InputGroupButton,
  InputGroupInput, InputGroupText
} from '@/components/ui/input-group'
```

Composable input with addons. The group manages a unified focus ring.

```tsx
<InputGroup>
  <InputGroupAddon align="inline-start">
    <SearchIcon className="text-muted-foreground size-4" />
  </InputGroupAddon>
  <InputGroupInput placeholder="Search..." />
</InputGroup>

<InputGroup>
  <InputGroupInput placeholder="Type a message..." />
  <InputGroupAddon align="inline-end">
    <InputGroupButton size="icon-sm" aria-label="Send">
      <SendIcon />
    </InputGroupButton>
  </InputGroupAddon>
</InputGroup>
```

---

## ButtonGroup

```tsx
import { ButtonGroup } from '@/components/ui/button-group'
```

Groups adjacent buttons, removing internal borders. Supports `orientation="horizontal"` (default) or `"vertical"`.

```tsx
<ButtonGroup>
  <Button variant="outline" size="icon" aria-label="Bold"><BoldIcon /></Button>
  <Button variant="outline" size="icon" aria-label="Italic"><ItalicIcon /></Button>
</ButtonGroup>
```

---

## Textarea

```tsx
import { Textarea } from '@/components/ui/textarea'
```

Uses `field-sizing: content` for auto-height based on content.

```tsx
<Textarea placeholder="Write something..." rows={3} />
```

---

## Select

```tsx
import {
  Select, SelectContent, SelectItem,
  SelectTrigger, SelectValue
} from '@/components/ui/select'
```

Radix-based select with portal dropdown.

```tsx
<Select>
  <SelectTrigger>
    <SelectValue placeholder="Choose a framework" />
  </SelectTrigger>
  <SelectContent>
    <SelectItem value="next">Next.js</SelectItem>
    <SelectItem value="remix">Remix</SelectItem>
    <SelectItem value="astro">Astro</SelectItem>
  </SelectContent>
</Select>
```

---

## Tabs

```tsx
import { Tabs, TabsList, TabsTrigger, TabsContent } from '@/components/ui/tabs'
```

Triggers are monospace, uppercase, with active state via `data-[state=active]`.

```tsx
<Tabs defaultValue="overview">
  <TabsList>
    <TabsTrigger value="overview">Overview</TabsTrigger>
    <TabsTrigger value="settings">Settings</TabsTrigger>
  </TabsList>
  <TabsContent value="overview">Overview content.</TabsContent>
  <TabsContent value="settings">Settings content.</TabsContent>
</Tabs>
```

---

## Switch

```tsx
import { Switch } from '@/components/ui/switch'
```

**Sizes:** `default` | `sm`

```tsx
<Switch id="notifications" />
<Switch id="compact" size="sm" />
```

---

## Slider

```tsx
import { Slider } from '@/components/ui/slider'
```

Supports single value and range (pass an array to `defaultValue`). Thumb is a narrow 1.5-wide bar.

```tsx
<Slider defaultValue={[40]} max={100} aria-label="Volume" />
<Slider defaultValue={[25, 75]} max={100} aria-label="Price range" />
```

---

## Separator

```tsx
import { Separator } from '@/components/ui/separator'
```

**Props:** `orientation` (`horizontal` | `vertical`), `brackets` (boolean — decorative bracket endpoints), `align` (`top` | `center` | `bottom`), `thickness` (number, default 2).

```tsx
<Separator />
<Separator brackets />
<div className="flex h-8 items-center gap-4">
  <span>Item A</span>
  <Separator orientation="vertical" />
  <span>Item B</span>
</div>
```

---

## Kbd

```tsx
import { Kbd, KbdGroup } from '@/components/ui/kbd'
```

Keyboard shortcut indicators. Monospace, uppercase, with border and shadow.

```tsx
<KbdGroup><Kbd>⌘</Kbd><Kbd>K</Kbd></KbdGroup>
<Kbd>Enter</Kbd>
```

---

## Avatar

```tsx
import { Avatar, AvatarImage, AvatarFallback } from '@/components/ui/avatar'
```

```tsx
<Avatar>
  <AvatarImage src="/avatar.png" alt="User" />
  <AvatarFallback>JC</AvatarFallback>
</Avatar>
```

---

## Tooltip

```tsx
import { Tooltip, TooltipTrigger, TooltipContent } from '@/components/ui/tooltip'
```

Zero-delay by default. Inverted colors (`bg-foreground text-background`).

```tsx
<Tooltip>
  <TooltipTrigger asChild>
    <Button size="icon" aria-label="Settings"><SettingsIcon /></Button>
  </TooltipTrigger>
  <TooltipContent>Settings</TooltipContent>
</Tooltip>
```

---

## Popover

```tsx
import { Popover, PopoverTrigger, PopoverContent } from '@/components/ui/popover'
```

Backdrop blur with `bg-popover/60 backdrop-blur-lg`.

```tsx
<Popover>
  <PopoverTrigger asChild>
    <Button variant="outline">Open</Button>
  </PopoverTrigger>
  <PopoverContent>
    <div className="flex flex-col gap-2">
      <p className="text-sm font-medium">Settings</p>
      <Input placeholder="Name" />
    </div>
  </PopoverContent>
</Popover>
```

---

## DropdownMenu

```tsx
import {
  DropdownMenu, DropdownMenuContent, DropdownMenuItem,
  DropdownMenuLabel, DropdownMenuSeparator, DropdownMenuSub,
  DropdownMenuSubContent, DropdownMenuSubTrigger, DropdownMenuTrigger
} from '@/components/ui/dropdown-menu'
```

```tsx
<DropdownMenu>
  <DropdownMenuTrigger asChild>
    <Button variant="outline">Menu</Button>
  </DropdownMenuTrigger>
  <DropdownMenuContent>
    <DropdownMenuLabel>Account</DropdownMenuLabel>
    <DropdownMenuSeparator />
    <DropdownMenuItem><UserIcon /> Profile</DropdownMenuItem>
    <DropdownMenuItem><SettingsIcon /> Settings</DropdownMenuItem>
    <DropdownMenuSub>
      <DropdownMenuSubTrigger><ShareIcon /> Share</DropdownMenuSubTrigger>
      <DropdownMenuSubContent>
        <DropdownMenuItem>Copy Link</DropdownMenuItem>
        <DropdownMenuItem>Email</DropdownMenuItem>
      </DropdownMenuSubContent>
    </DropdownMenuSub>
    <DropdownMenuSeparator />
    <DropdownMenuItem variant="destructive"><TrashIcon /> Delete</DropdownMenuItem>
  </DropdownMenuContent>
</DropdownMenu>
```

---

## Collapsible

```tsx
import { Collapsible, CollapsibleTrigger, CollapsibleContent } from '@/components/ui/collapsible'
```

Animated expand/collapse.

```tsx
<Collapsible>
  <CollapsibleTrigger asChild>
    <Button variant="outline" size="sm">Toggle</Button>
  </CollapsibleTrigger>
  <CollapsibleContent className="mt-3">
    <div className="bg-muted rounded-md p-4 text-sm">
      Expandable content with enter/exit animations.
    </div>
  </CollapsibleContent>
</Collapsible>
```
