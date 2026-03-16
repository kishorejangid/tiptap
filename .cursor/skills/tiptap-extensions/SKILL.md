---
name: tiptap-extensions
description: Create custom TipTap nodes, marks, extensions, React node views, and decorations. Use when building any custom editor element — block nodes, inline nodes, inline formatting marks, ProseMirror plugins, React-rendered node views, or visual decoration overlays.
---

# TipTap — Custom Nodes, Marks, Extensions, Node Views & Decorations

## When to use this skill
- "Create a custom [callout / card / embed / divider / …] block"
- "Add a custom [highlight / comment / spoiler / badge / …] mark"
- "Build an extension that [does X]"
- "Render a node as a React component"
- "Add inline decorations / highlights / overlays"

---

## CRITICAL RULES

1. **ALWAYS add a `declare module '@tiptap/core'` block** for every `addCommands`. Without it `editor.commands.yourCmd` is typed `never`.
2. **NEVER use JSX in `renderHTML`** — return a ProseMirror `DOMOutputSpec` array.
3. **When using Collaboration** set `StarterKit.configure({ history: false })`.
4. **Guard `dispatch`** — all `tr.*` mutations inside `addCommands` must be wrapped in `if (dispatch) { … }`.
5. **`renderHTML` `0` is the "hole"** — it marks where child content goes. Atom/leaf nodes don't need it.

---

## Custom Block Node — Complete Template

```ts
// extensions/Callout.ts
import { Node, mergeAttributes } from '@tiptap/core'

export interface CalloutOptions {
  HTMLAttributes: Record<string, any>
}

// REQUIRED: augment Commands interface
declare module '@tiptap/core' {
  interface Commands<ReturnType> {
    callout: {
      setCallout: (attrs?: { type: 'info' | 'warning' | 'error' }) => ReturnType
      unsetCallout: () => ReturnType
    }
  }
}

export const Callout = Node.create<CalloutOptions>({
  name: 'callout',
  group: 'block',
  content: 'block+',
  defining: true,

  addOptions() {
    return { HTMLAttributes: {} }
  },

  addAttributes() {
    return {
      type: {
        default: 'info',
        parseHTML: el => el.getAttribute('data-callout-type'),
        renderHTML: attrs => ({ 'data-callout-type': attrs.type }),
      },
    }
  },

  parseHTML() {
    return [{ tag: 'div[data-callout]' }]
  },

  renderHTML({ node, HTMLAttributes }) {
    return [
      'div',
      mergeAttributes(this.options.HTMLAttributes, HTMLAttributes, {
        'data-callout': '',
        class: `callout callout--${node.attrs.type}`,
      }),
      0,  // ← hole for editable children
    ]
  },

  addCommands() {
    return {
      setCallout: attrs => ({ commands }) => commands.wrapIn(this.name, attrs),
      unsetCallout: () => ({ commands }) => commands.lift(this.name),
    }
  },

  addKeyboardShortcuts() {
    return { 'Mod-Shift-c': () => this.editor.commands.setCallout() }
  },
})
```

### Node schema property table

| Property | Values | Description |
|----------|--------|-------------|
| `group` | `'block'` `'inline'` custom | Block/inline placement |
| `content` | `'block+'` `'inline*'` `'paragraph block*'` | Child content grammar |
| `marks` | `'_'` (all) `''` (none) `'bold italic'` | Allowed marks |
| `inline` | boolean | True for inline nodes |
| `atom` | boolean | Leaf — no editable children |
| `selectable` | boolean | Can be node-selected |
| `draggable` | boolean | Drag without selecting first |
| `defining` | boolean | Structural boundary (backspace/Enter stops here) |
| `isolating` | boolean | Hard wall both sides (table cell) |
| `code` | boolean | Enter = newline, not new block |
| `whitespace` | `'normal'` `'pre'` | Whitespace handling |

### Inline / Atom node (chip, emoji, math)

```ts
export const MyChip = Node.create({
  name: 'myChip',
  group: 'inline',
  inline: true,
  atom: true,       // single unit, no editable children
  selectable: true,

  addAttributes() {
    return {
      label: { default: '' },
      value: { default: '' },
    }
  },

  parseHTML() { return [{ tag: 'span[data-chip]' }] },

  renderHTML({ node, HTMLAttributes }) {
    return ['span', mergeAttributes(HTMLAttributes, { 'data-chip': node.attrs.value }), node.attrs.label]
  },
})
```

---

## Custom Mark — Complete Template

```ts
// extensions/MyMark.ts
import { Mark, markInputRule, markPasteRule, mergeAttributes } from '@tiptap/core'

declare module '@tiptap/core' {
  interface Commands<ReturnType> {
    myMark: {
      setMyMark: (attrs?: { color: string }) => ReturnType
      toggleMyMark: (attrs?: { color: string }) => ReturnType
      unsetMyMark: () => ReturnType
    }
  }
}

export const MyMark = Mark.create({
  name: 'myMark',

  inclusive: true,    // cursor enters mark at its edge
  excludes: '',       // '' = compatible with all; use name to exclude specific mark
  spanning: true,     // spans across nodes
  keepOnSplit: true,  // preserved on Enter
  exitable: false,    // Arrow key can exit the mark

  addAttributes() {
    return {
      color: {
        default: '#ffeb3b',
        parseHTML: el => el.getAttribute('data-color'),
        renderHTML: attrs => ({ 'data-color': attrs.color, style: `background-color: ${attrs.color}` }),
      },
    }
  },

  parseHTML() { return [{ tag: 'mark[data-my-mark]' }] },

  renderHTML({ HTMLAttributes }) {
    return ['mark', mergeAttributes(HTMLAttributes, { 'data-my-mark': '' }), 0]
  },

  addCommands() {
    return {
      setMyMark: attrs => ({ commands }) => commands.setMark(this.name, attrs),
      toggleMyMark: attrs => ({ commands }) => commands.toggleMark(this.name, attrs),
      unsetMyMark: () => ({ commands }) => commands.unsetMark(this.name),
    }
  },

  addKeyboardShortcuts() {
    return { 'Mod-Shift-m': () => this.editor.commands.toggleMyMark() }
  },

  // Triggered by ==text== pattern while typing
  addInputRules() {
    return [markInputRule({ find: /(?:^|\s)(==(?!\s+==)((?:[^=]+))==(?!\s+==))$/, type: this.type })]
  },
  addPasteRules() {
    return [markPasteRule({ find: /(?:^|\s)(==(?!\s+==)((?:[^=]+))==(?!\s+==))/g, type: this.type })]
  },
})
```

---

## Custom Extension (Behavior Only, No Schema)

```ts
// extensions/MyBehavior.ts
import { Extension } from '@tiptap/core'
import { Plugin, PluginKey } from '@tiptap/pm/state'

export const MyBehavior = Extension.create<{ onEvent?: (data: unknown) => void }, { cache: Map<string, unknown> }>({
  name: 'myBehavior',

  addOptions() {
    return { onEvent: undefined }
  },

  addStorage() {
    return { cache: new Map() }  // access via editor.storage.myBehavior.cache
  },

  // Inject attributes into EXISTING nodes without editing those extensions
  addGlobalAttributes() {
    return [{
      types: ['paragraph', 'heading'],
      attributes: {
        blockId: {
          default: null,
          parseHTML: el => el.getAttribute('data-block-id'),
          renderHTML: attrs => attrs.blockId ? { 'data-block-id': attrs.blockId } : {},
        },
      },
    }]
  },

  addKeyboardShortcuts() {
    return { 'Mod-Shift-x': () => { /* return true = handled */ return true } }
  },

  addProseMirrorPlugins() {
    const key = new PluginKey('myBehavior')
    return [new Plugin({
      key,
      props: {
        handleDOMEvents: {
          click: (view, event) => false,  // false = let other handlers process
        },
      },
    })]
  },

  addExtensions() { return [] },  // bundle other extensions (kit pattern)

  onCreate({ editor }) {},
  onUpdate({ editor }) {},
  onDestroy() { /* clean up timers, subscriptions, WebSockets */ },

  dispatchTransaction({ transaction, next }) {
    // Intercept every transaction — always call next()
    next(transaction)
  },
})
```

### Extending an existing extension

```ts
import { Bold } from '@tiptap/extension-bold'

const CustomBold = Bold.extend({
  renderHTML({ HTMLAttributes }) {
    return ['strong', { ...HTMLAttributes, class: 'font-bold' }, 0]
  },
  addKeyboardShortcuts() {
    return {
      ...this.parent?.(),       // inherit parent shortcuts
      'Mod-Alt-b': () => this.editor.commands.toggleBold(),
    }
  },
})
```

### Configure an extension (change options, don't override behavior)

```ts
Highlight.configure({ multicolor: true })
Heading.configure({ levels: [1, 2, 3] })
Link.configure({ openOnClick: false, HTMLAttributes: { rel: 'noopener' } })
```

---

## React Node Views — Render a Node as a React Component

### The React component

```tsx
// components/CalloutView.tsx
import { NodeViewWrapper, NodeViewContent } from '@tiptap/react'
import type { NodeViewProps } from '@tiptap/core'

export function CalloutView({ node, updateAttributes, deleteNode, selected }: NodeViewProps) {
  const { type } = node.attrs
  return (
    // NodeViewWrapper MUST be root — sets up ProseMirror event delegation
    <NodeViewWrapper className={`callout callout--${type} ${selected ? 'ring-2' : ''}`}>

      {/* contentEditable={false} prevents PM swallowing interactive element events */}
      <div className="callout__header" contentEditable={false}>
        <select value={type} onChange={e => updateAttributes({ type: e.target.value })}>
          <option value="info">ℹ Info</option>
          <option value="warning">⚠ Warning</option>
          <option value="error">✖ Error</option>
        </select>
        <button onClick={deleteNode}>Delete</button>
      </div>

      {/* NodeViewContent = editable children — use `as` to change tag */}
      <NodeViewContent as="div" className="callout__body" />
    </NodeViewWrapper>
  )
}
```

### Wire up in the Node extension

```ts
import { ReactNodeViewRenderer } from '@tiptap/react'
import { CalloutView } from '../components/CalloutView'

// Inside Node.create({ ... })
addNodeView() {
  return ReactNodeViewRenderer(CalloutView, {
    as: 'div',
    className: 'callout-outer',
    // Fine-grained re-render control:
    update: ({ oldNode, newNode, updateProps }) => {
      if (oldNode.attrs.type === newNode.attrs.type) {
        updateProps()  // update props, skip full remount
        return true
      }
      return false  // let TipTap do default update
    },
  })
},
```

### NodeViewProps reference

```ts
interface NodeViewProps {
  editor: Editor
  node: ProseMirrorNode           // the current node
  decorations: Decoration[]
  innerDecorations: DecorationSource
  selected: boolean               // true when node-selected
  extension: Node
  getPos: () => number | undefined  // always guard: if (getPos() == null) return
  updateAttributes: (attrs: Record<string, any>) => void
  deleteNode: () => void
}
```

### Leaf / Atom Node View (no NodeViewContent)

```tsx
export function MathView({ node, selected }: NodeViewProps) {
  return (
    <NodeViewWrapper as="span" className={`math ${selected ? 'ring-2' : ''}`}>
      <span contentEditable={false}>{renderMath(node.attrs.formula)}</span>
    </NodeViewWrapper>
  )
  // No NodeViewContent — atom nodes have no editable children
}
```

---

## Decorations — Visual Overlays (Don't Modify Document)

Use decorations for: search highlights, AI streaming preview, validation errors, collaboration cursors.

```ts
import { Plugin, PluginKey } from '@tiptap/pm/state'
import { Decoration, DecorationSet } from '@tiptap/pm/view'

const decoKey = new PluginKey<DecorationSet>('myDecos')

// In addProseMirrorPlugins():
new Plugin<DecorationSet>({
  key: decoKey,
  state: {
    init: (_, { doc }) => buildDecos(doc),
    apply: (tr, old) => {
      if (!tr.docChanged && !tr.getMeta(decoKey)) return old
      return buildDecos(tr.doc)
    },
  },
  props: {
    decorations: state => decoKey.getState(state),
  },
})

function buildDecos(doc: any): DecorationSet {
  const decos: Decoration[] = []
  doc.descendants((node: any, pos: number) => {
    if (!node.isText) return
    // Inline deco — color/annotate a text range
    decos.push(Decoration.inline(pos, pos + node.nodeSize, { class: 'my-highlight' }))
  })
  return DecorationSet.create(doc, decos)
}

// Widget deco — insert a DOM element at a position
const el = document.createElement('span')
el.className = 'badge'
el.textContent = '●'
Decoration.widget(pos, () => el, { side: 1, key: 'my-badge' })

// Node deco — add class/attrs to a block's wrapper
Decoration.node(from, to, { class: 'is-active-block' })

// Trigger rebuild from outside:
editor.view.dispatch(editor.state.tr.setMeta(decoKey, { refresh: true }))
```

---

## Input Rule Helpers

```ts
import { markInputRule, markPasteRule, nodeInputRule, textblockTypeInputRule, wrappingInputRule } from '@tiptap/core'

markInputRule({ find: /\*\*(.*)\*\*$/, type: this.type })       // apply mark on type
markPasteRule({ find: /\*\*(.*)\*\*/g, type: this.type })       // apply mark on paste
nodeInputRule({ find: /^---$/, type: this.type })               // replace text with node
textblockTypeInputRule({ find: /^#{1,6}\s$/, type: this.type, getAttributes: { level: 1 } })  // change block type
wrappingInputRule({ find: /^>\s$/, type: this.type })           // wrap in node
```

---

## Extension Management — File Structure & Registration

### Recommended folder layout

```
src/
└── extensions/
    ├── index.ts            ← barrel — export everything here
    ├── kit.ts              ← your project's StarterKit replacement
    │
    ├── callout/
    │   ├── Callout.ts      ← Node/Mark/Extension definition
    │   ├── CalloutView.tsx ← React NodeView component (if needed)
    │   └── callout.css     ← scoped styles (if not using Tailwind)
    │
    ├── slash-command/
    │   ├── SlashCommand.ts
    │   ├── SlashList.tsx
    │   └── commands.ts
    │
    └── my-mark/
        └── MyMark.ts
```

**Rules:**
- One extension per folder. Keep the Node/Mark/Extension class and its NodeView component co-located.
- Never import a NodeView component outside its extension folder — only the extension itself knows its renderer.
- Export everything from `extensions/index.ts` so consumers have a single import point.

### Barrel export (`extensions/index.ts`)

```ts
// extensions/index.ts
export { Callout } from './callout/Callout'
export type { CalloutOptions } from './callout/Callout'

export { MyMark } from './my-mark/MyMark'
export type { MyMarkOptions } from './my-mark/MyMark'

export { SlashCommand } from './slash-command/SlashCommand'

// Re-export the kit for one-line editor setup
export { EditorKit } from './kit'
```

### The Kit pattern — bundle extensions into one

Create a single `EditorKit` extension that bundles all your project extensions. This is the same pattern as `StarterKit`.

```ts
// extensions/kit.ts
import { Extension } from '@tiptap/core'
import StarterKit from '@tiptap/starter-kit'
import { Highlight } from '@tiptap/extension-highlight'
import { Underline } from '@tiptap/extension-underline'
import { TextAlign } from '@tiptap/extension-text-align'
import { Link } from '@tiptap/extension-link'
import { Callout } from './callout/Callout'
import { MyMark } from './my-mark/MyMark'
import { SlashCommand } from './slash-command/SlashCommand'

export interface EditorKitOptions {
  // Expose per-extension options — false = disable that extension
  callout?: Partial<CalloutOptions> | false
  myMark?: Partial<MyMarkOptions> | false
  slashCommand?: boolean
  // Pass-through to StarterKit
  heading?: { levels: (1 | 2 | 3 | 4 | 5 | 6)[] } | false
}

export const EditorKit = Extension.create<EditorKitOptions>({
  name: 'editorKit',

  addOptions() {
    return {
      callout: {},
      myMark: {},
      slashCommand: true,
      heading: { levels: [1, 2, 3] },
    }
  },

  addExtensions() {
    const ext = []

    // StarterKit (always included)
    ext.push(StarterKit.configure({
      heading: this.options.heading === false ? false : this.options.heading,
      history: true,
    }))

    // Common formatting
    ext.push(Highlight.configure({ multicolor: true }))
    ext.push(Underline)
    ext.push(TextAlign.configure({ types: ['heading', 'paragraph'] }))
    ext.push(Link.configure({ openOnClick: false }))

    // Project-specific — only add if not disabled
    if (this.options.callout !== false) {
      ext.push(Callout.configure(this.options.callout ?? {}))
    }
    if (this.options.myMark !== false) {
      ext.push(MyMark.configure(this.options.myMark ?? {}))
    }
    if (this.options.slashCommand) {
      ext.push(SlashCommand)
    }

    return ext
  },
})
```

**Usage — one line to set up the entire editor:**

```ts
import { EditorKit } from '@/extensions'

const editor = useEditor({
  extensions: [
    EditorKit.configure({
      callout: { HTMLAttributes: { class: 'rounded-xl' } },
      myMark: false,      // disable for this editor instance
      heading: { levels: [1, 2] },
    }),
  ],
})
```

---

## Registration — Extension Order & Priority

### Order in the `extensions` array matters for:
- **Input rules** — first match wins; put specific rules before general ones
- **Keyboard shortcuts** — same key in two extensions: higher `priority` wins
- **Schema** — order affects how ProseMirror resolves ambiguous content

### Priority system

```ts
// Default priority is 100. Higher = evaluated first.
export const MyMark = Mark.create({
  name: 'myMark',
  priority: 200,  // runs before default-priority extensions
})

// Override priority when extending:
const HighPriorityBold = Bold.extend({ priority: 150 })
```

### Recommended registration order

```ts
extensions: [
  // 1. Document structure (highest priority — StarterKit handles this)
  StarterKit,

  // 2. Custom block nodes
  Callout,
  MyEmbed,

  // 3. Custom marks
  MyMark,

  // 4. Behavior extensions (no schema)
  SlashCommand,
  MyBehavior,

  // 5. UI helpers (lowest priority — attach last)
  BubbleMenu,
  FloatingMenu,
]
```

### Disabling a StarterKit extension to replace it

```ts
StarterKit.configure({
  // Set to false to remove; provide your custom version separately
  bold: false,
  codeBlock: false,
}),
// Then add your custom versions:
CustomBold,
CodeBlockLowlight.configure({ lowlight }),
```

---

## Passing Options from Outside the Extension

Extensions receive options at `configure()` time. For **runtime changes** (e.g. toggling a feature based on user auth), use `addStorage` + direct mutation:

```ts
// Configure at init time:
editor = useEditor({
  extensions: [
    MyExtension.configure({ featureEnabled: false }),
  ],
})

// Change at runtime — direct storage mutation:
editor.storage.myExtension.featureEnabled = true

// Or re-create the editor (heavier but cleaner for large option changes):
editor.destroy()
editor = createEditor({ ...newOptions })
```

### Expose a typed storage interface

```ts
export interface MyExtensionStorage {
  featureEnabled: boolean
  lastAction: string | null
}

export const MyExtension = Extension.create<MyExtensionOptions, MyExtensionStorage>({
  addStorage(): MyExtensionStorage {
    return { featureEnabled: false, lastAction: null }
  },
})

// Consumer gets fully typed access:
const enabled: boolean = editor.storage.myExtension.featureEnabled
```

---

## Testing Custom Extensions

### Unit test setup (Vitest / Jest)

```bash
npm install -D @tiptap/core @tiptap/pm jsdom vitest
```

```ts
// extensions/callout/__tests__/Callout.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { Editor } from '@tiptap/core'
import StarterKit from '@tiptap/starter-kit'
import { Callout } from '../Callout'

function createEditor(content = '<p>Hello</p>') {
  return new Editor({
    extensions: [StarterKit, Callout],
    content,
    element: document.createElement('div'),
  })
}

describe('Callout extension', () => {
  let editor: Editor

  beforeEach(() => {
    editor = createEditor()
  })

  it('setCallout wraps the current block', () => {
    editor.commands.setCallout({ type: 'info' })
    expect(editor.isActive('callout')).toBe(true)
    expect(editor.getAttributes('callout').type).toBe('info')
  })

  it('unsetCallout lifts the callout', () => {
    editor.commands.setCallout()
    editor.commands.unsetCallout()
    expect(editor.isActive('callout')).toBe(false)
  })

  it('serializes to correct HTML', () => {
    editor.commands.setCallout({ type: 'warning' })
    const html = editor.getHTML()
    expect(html).toContain('data-callout')
    expect(html).toContain('data-callout-type="warning"')
  })

  it('parses from HTML', () => {
    editor.commands.setContent('<div data-callout data-callout-type="error"><p>text</p></div>')
    expect(editor.isActive('callout')).toBe(true)
    expect(editor.getAttributes('callout').type).toBe('error')
  })

  afterEach(() => editor.destroy())
})
```

### vitest.config.ts for TipTap

```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    environment: 'jsdom',       // TipTap needs a DOM
    globals: true,
    setupFiles: ['./test-setup.ts'],
  },
})
```

```ts
// test-setup.ts
import { vi } from 'vitest'

// TipTap uses ResizeObserver — polyfill for jsdom
global.ResizeObserver = vi.fn().mockImplementation(() => ({
  observe: vi.fn(),
  unobserve: vi.fn(),
  disconnect: vi.fn(),
}))
```

---

## Common Mistakes & Fixes

| Mistake | Fix |
|---------|-----|
| `editor.commands.myCmd` typed as `never` | Add `declare module '@tiptap/core' { interface Commands<R> { ... } }` |
| Children not editable in NodeView | Check `renderHTML` has `0` hole; or use `NodeViewContent` in React view |
| Interactive elements don't receive events | Add `contentEditable={false}` to non-content elements inside NodeView |
| Mark disappears on Enter | Set `keepOnSplit: true` |
| Wrong extension runs first | Set higher `priority` (default 100; higher = earlier) |
| `dispatch` errors | Guard: `if (dispatch) { tr.insertText(...) }` |
| Global attributes not shown | Check `types` array matches extension `name` exactly |
| Node can't be inserted | Ensure parent `content` expression allows this node's `group` |
| `getPos()` undefined crash | Guard: `const pos = getPos(); if (pos == null) return` |
| Decoration flickers | Set stable `key` on `Decoration.widget(pos, el, { key: 'stable-id' })` |
