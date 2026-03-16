# TipTap Custom Editor — Integration Specification

> **Target stack:** React + TypeScript
> **Collaboration backend:** Hocuspocus (TipTap's official WebSocket server)
> **AI integration:** Provider-agnostic (abstract interface)
> **Track changes:** Full implementation with accept/reject, author attribution, and history

This document is a self-contained reference for building a feature-rich, block-based editor on top of TipTap. It covers every layer from project setup through AI content generation.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Project Setup](#2-project-setup)
3. [Editor Initialization](#3-editor-initialization)
4. [Extension System Deep-Dive](#4-extension-system-deep-dive)
5. [Custom Nodes](#5-custom-nodes)
6. [Custom Marks (Inline Formatting)](#6-custom-marks-inline-formatting)
7. [Node Decorations](#7-node-decorations)
8. [React Node Views](#8-react-node-views)
9. [Toolbar — Fixed & Bubble Menu](#9-toolbar--fixed--bubble-menu)
10. [Notion-Style Slash Commands](#10-notion-style-slash-commands)
11. [Collaborative Editing with Hocuspocus](#11-collaborative-editing-with-hocuspocus)
12. [Track Changes](#12-track-changes)
13. [AI Content Generation Extension](#13-ai-content-generation-extension)
14. [Custom Schema Reference](#14-custom-schema-reference)
15. [Command System](#15-command-system)
16. [Events & Lifecycle Hooks](#16-events--lifecycle-hooks)
17. [Full Editor Bootstrap Example](#17-full-editor-bootstrap-example)

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                    React Component                  │
│  ┌────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │  Toolbar   │  │  BubbleMenu │  │ FloatingMenu│  │
│  └────────────┘  └─────────────┘  └─────────────┘  │
│                  ┌─────────────┐                    │
│                  │EditorContent│                    │
│                  └──────┬──────┘                    │
└─────────────────────────┼───────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────┐
│              TipTap Editor Instance                 │
│  ┌─────────────────────────────────────────────┐    │
│  │           Extension Manager                 │    │
│  │  Nodes | Marks | Extensions | Plugins       │    │
│  └─────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────┐    │
│  │         ProseMirror Core                    │    │
│  │   Schema · State · View · Transaction       │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
                          │
         ┌────────────────┴────────────────┐
         ▼                                 ▼
┌─────────────────┐              ┌──────────────────┐
│  Y.js Document  │◄────────────►│  Hocuspocus WS   │
│  (CRDT in-mem)  │              │  Server          │
└─────────────────┘              └──────────────────┘
```

**Core primitives:**

| Primitive | Purpose |
|-----------|---------|
| `Node` | Block-level or inline structural elements (paragraph, heading, image…) |
| `Mark` | Inline formatting applied over text (bold, highlight, link…) |
| `Extension` | Behaviour only, no schema change (keyboard shortcuts, plugins…) |
| `NodeView` | Custom React component rendering a node |
| `Decoration` | Ephemeral UI overlay (highlights, caret indicators…) that does not affect document content |

---

## 2. Project Setup

### Installation

```bash
# Core
npm install @tiptap/core @tiptap/react @tiptap/pm

# Starter nodes & marks
npm install @tiptap/starter-kit
npm install @tiptap/extension-highlight @tiptap/extension-underline
npm install @tiptap/extension-text-align @tiptap/extension-color
npm install @tiptap/extension-font-family @tiptap/extension-text-style
npm install @tiptap/extension-link @tiptap/extension-image
npm install @tiptap/extension-table @tiptap/extension-code-block-lowlight
npm install @tiptap/extension-mention           # base for slash commands

# Menus
npm install @tiptap/extension-bubble-menu @tiptap/extension-floating-menu

# Collaboration
npm install @tiptap/extension-collaboration
npm install @tiptap/extension-collaboration-caret
npm install @hocuspocus/provider yjs

# Suggestion engine (slash commands)
npm install @tiptap/suggestion

# Optional: drag handles
npm install @tiptap/extension-drag-handle-react

# Markdown (optional)
npm install @tiptap/markdown
```

### TypeScript `tsconfig.json` (minimum)

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true
  }
}
```

---

## 3. Editor Initialization

### `useEditor` hook (React)

```tsx
import { useEditor, EditorContent } from '@tiptap/react'
import StarterKit from '@tiptap/starter-kit'

export function Editor() {
  const editor = useEditor({
    // Extensions register nodes, marks, plugins
    extensions: [StarterKit],

    // Initial document as JSON or HTML
    content: '<p>Hello world</p>',

    // Called once on mount
    onCreate({ editor }) {
      console.log('editor ready', editor)
    },

    // Called on every document change
    onUpdate({ editor }) {
      const json = editor.getJSON()
    },

    // Called when selection changes
    onSelectionUpdate({ editor }) {},

    // Called on every ProseMirror transaction
    onTransaction({ editor, transaction }) {},

    // Server-side rendering: set false until hydrated
    immediatelyRender: false,
  })

  return <EditorContent editor={editor} />
}
```

### `EditorOptions` reference

```ts
interface EditorOptions {
  element?: HTMLElement           // Mount target (managed by EditorContent in React)
  extensions: Extensions          // Array of Node/Mark/Extension instances
  content?: Content               // Initial content: string HTML, JSON, or null
  editable?: boolean              // Default true; toggles read-only mode
  autofocus?: 'start' | 'end' | 'all' | number | boolean
  injectCSS?: boolean             // Auto-inject base CSS (default true)
  injectNonce?: string            // CSP nonce for injected style tag
  enableCoreExtensions?: boolean  // Include built-in Dropcursor/Gapcursor (default true)
  enablePasteRules?: boolean      // default true
  enableInputRules?: boolean      // default true
  parseOptions?: ParseOptions     // ProseMirror parse options
  editorProps?: EditorProps       // ProseMirror EditorProps (handleDOMEvents etc.)
  onCreate?: (props: { editor: Editor }) => void
  onUpdate?: (props: { editor: Editor; transaction: Transaction }) => void
  onSelectionUpdate?: (props: { editor: Editor }) => void
  onTransaction?: (props: { editor: Editor; transaction: Transaction }) => void
  onFocus?: (props: { editor: Editor; event: FocusEvent }) => void
  onBlur?: (props: { editor: Editor; event: FocusEvent }) => void
  onDestroy?: (props: { editor: Editor }) => void
  onContentError?: (props: { editor: Editor; error: Error; disableCollaboration: () => void }) => void
}
```

---

## 4. Extension System Deep-Dive

All three primitives (`Extension`, `Node`, `Mark`) share the same base `Extendable` class and expose the same lifecycle hooks.

### Shared config hooks (available on all three primitives)

```ts
{
  name: string                    // REQUIRED – unique identifier

  priority?: number               // Higher = earlier execution (default 100)

  addOptions() {
    // Return default option values
    return { myOption: 'default' }
  }

  addStorage() {
    // Return mutable per-extension storage object
    return { cache: new Map() }
  }

  addGlobalAttributes() {
    // Inject attributes into OTHER nodes/marks
    return [{
      types: ['heading', 'paragraph'],
      attributes: {
        textAlign: {
          default: 'left',
          parseHTML: el => el.style.textAlign || 'left',
          renderHTML: attrs => ({ style: `text-align: ${attrs.textAlign}` }),
        },
      },
    }]
  }

  addCommands() {
    // Expose commands on editor.commands.*
    return {
      myCommand: (arg: string) => ({ commands }) => {
        return commands.setMark('bold')
      },
    }
  }

  addKeyboardShortcuts() {
    return {
      'Mod-b': () => this.editor.commands.toggleBold(),
      'Mod-Shift-h': () => this.editor.commands.toggleHighlight(),
    }
  }

  addInputRules() {
    // Transform text while typing
    return []
  }

  addPasteRules() {
    // Transform pasted text
    return []
  }

  addProseMirrorPlugins() {
    // Low-level ProseMirror Plugin API
    return []
  }

  addExtensions() {
    // Return dependency extensions (kit pattern)
    return [Bold, Italic]
  }

  // Lifecycle
  onBeforeCreate({ editor }) {}
  onCreate({ editor }) {}
  onUpdate({ editor, transaction }) {}
  onSelectionUpdate({ editor }) {}
  onTransaction({ editor, transaction }) {}
  onFocus({ editor, event }) {}
  onBlur({ editor, event }) {}
  onDestroy() {}
}
```

### Extending an existing extension

```ts
import { Bold } from '@tiptap/extension-bold'

const CustomBold = Bold.extend({
  // Override renderHTML
  renderHTML({ HTMLAttributes }) {
    return ['strong', { ...HTMLAttributes, class: 'my-bold' }, 0]
  },
  // Add new keyboard shortcut on top of parent's
  addKeyboardShortcuts() {
    return {
      ...this.parent?.(),           // keep parent shortcuts
      'Mod-Alt-b': () => this.editor.commands.toggleBold(),
    }
  },
})
```

### Augmenting the Commands type (TypeScript module augmentation)

Every extension that adds commands **must** augment the global `Commands` interface so TypeScript knows about them:

```ts
declare module '@tiptap/core' {
  interface Commands<ReturnType> {
    myExtension: {
      doSomething: (arg: string) => ReturnType
    }
  }
}
```

---

## 5. Custom Nodes

### Minimal block node

```ts
import { Node, mergeAttributes } from '@tiptap/core'

export interface CalloutOptions {
  HTMLAttributes: Record<string, any>
}

declare module '@tiptap/core' {
  interface Commands<ReturnType> {
    callout: {
      setCallout: (attributes?: { type: 'info' | 'warning' | 'error' }) => ReturnType
    }
  }
}

export const Callout = Node.create<CalloutOptions>({
  name: 'callout',

  group: 'block',           // belongs to the "block" group
  content: 'block+',        // contains one or more block nodes
  defining: true,           // acts as boundary for backspace/Enter

  addOptions() {
    return { HTMLAttributes: {} }
  },

  addAttributes() {
    return {
      type: {
        default: 'info',
        parseHTML: el => el.getAttribute('data-type'),
        renderHTML: attrs => ({ 'data-type': attrs.type }),
      },
    }
  },

  parseHTML() {
    return [{ tag: 'div[data-callout]' }]
  },

  renderHTML({ node, HTMLAttributes }) {
    return [
      'div',
      mergeAttributes(
        this.options.HTMLAttributes,
        HTMLAttributes,
        { 'data-callout': '', class: `callout callout--${node.attrs.type}` }
      ),
      0,   // "hole" = where children are inserted
    ]
  },

  addCommands() {
    return {
      setCallout: attrs => ({ commands }) =>
        commands.wrapIn(this.name, attrs),
    }
  },
})
```

### Inline / leaf (atom) node example

```ts
export const InlineMath = Node.create({
  name: 'inlineMath',

  group: 'inline',
  inline: true,
  atom: true,             // no editable children, treated as single unit
  selectable: true,
  draggable: false,

  addAttributes() {
    return {
      formula: { default: '' },
    }
  },

  parseHTML() {
    return [{ tag: 'span[data-inline-math]' }]
  },

  renderHTML({ node, HTMLAttributes }) {
    return ['span', mergeAttributes(HTMLAttributes, { 'data-inline-math': '' }), node.attrs.formula]
  },

  // Rendered as a React component via addNodeView (see §8)
})
```

### Node schema properties reference

| Property | Type | Description |
|----------|------|-------------|
| `group` | `string` | Space-separated groups, e.g. `'block'`, `'inline'`, `'list'` |
| `content` | `string` | ProseMirror content expression, e.g. `'block+'`, `'paragraph inline*'` |
| `marks` | `string` | Allowed marks, `'_'` = all, `''` = none |
| `inline` | `boolean` | True for inline nodes |
| `atom` | `boolean` | True for leaf nodes with no editable content |
| `selectable` | `boolean` | Whether node can be selected (default `true` for non-text) |
| `draggable` | `boolean` | Whether node can be dragged without selection |
| `code` | `boolean` | Affects keyboard behaviour inside (e.g. Enter inserts newline) |
| `defining` | `boolean` | Acts as structural boundary (blocks backspace/lift) |
| `isolating` | `boolean` | Treats node boundaries as hard walls (e.g. table cells) |
| `whitespace` | `'normal' \| 'pre'` | Whitespace handling |

---

## 6. Custom Marks (Inline Formatting)

```ts
import { Mark, markInputRule, markPasteRule, mergeAttributes } from '@tiptap/core'

export interface SpoilerOptions {
  HTMLAttributes: Record<string, any>
}

declare module '@tiptap/core' {
  interface Commands<ReturnType> {
    spoiler: {
      setSpoiler: () => ReturnType
      toggleSpoiler: () => ReturnType
      unsetSpoiler: () => ReturnType
    }
  }
}

export const Spoiler = Mark.create<SpoilerOptions>({
  name: 'spoiler',

  addOptions() {
    return { HTMLAttributes: {} }
  },

  // Mark schema
  inclusive: false,          // cursor does NOT "enter" the mark when placed at its edge
  excludes: '',              // compatible with all other marks
  spanning: false,           // does not span across nodes

  parseHTML() {
    return [{ tag: 'span[data-spoiler]' }]
  },

  renderHTML({ HTMLAttributes }) {
    return ['span', mergeAttributes(this.options.HTMLAttributes, HTMLAttributes, { 'data-spoiler': '' }), 0]
  },

  addCommands() {
    return {
      setSpoiler: () => ({ commands }) => commands.setMark(this.name),
      toggleSpoiler: () => ({ commands }) => commands.toggleMark(this.name),
      unsetSpoiler: () => ({ commands }) => commands.unsetMark(this.name),
    }
  },

  addKeyboardShortcuts() {
    return { 'Mod-Shift-s': () => this.editor.commands.toggleSpoiler() }
  },

  // ||spoiler text|| on input
  addInputRules() {
    return [markInputRule({ find: /(?:^|\s)(\|\|(?!\s)((?:[^|]+))\|\|)$/, type: this.type })]
  },

  addPasteRules() {
    return [markPasteRule({ find: /(?:^|\s)(\|\|(?!\s)((?:[^|]+))\|\|)/g, type: this.type })]
  },
})
```

### Mark schema properties reference

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `inclusive` | `boolean` | `true` | Cursor inside mark when placed at edge |
| `excludes` | `string` | mark name | Marks that cannot coexist with this one |
| `exitable` | `boolean` | `false` | Allow escaping the mark with Arrow key |
| `spanning` | `boolean` | `true` | Whether mark spans across multiple nodes |
| `group` | `string` | — | Mark group membership |
| `keepOnSplit` | `boolean` | `true` | Preserve mark on Enter/split |

---

## 7. Node Decorations

Decorations are visual overlays that do **not** modify the document. They are created via ProseMirror plugins and are ideal for: AI generation streaming preview, search highlights, collaboration cursors, validation errors.

### Three decoration types

| Type | Factory | Use case |
|------|---------|----------|
| Inline | `Decoration.inline(from, to, attrs)` | Color range of text |
| Node | `Decoration.node(from, to, attrs)` | Add class/attribute to node wrapper |
| Widget | `Decoration.widget(pos, dom, spec)` | Insert arbitrary DOM at position |

### Example: search highlight decoration

```ts
import { Extension } from '@tiptap/core'
import { Plugin, PluginKey } from '@tiptap/pm/state'
import { Decoration, DecorationSet } from '@tiptap/pm/view'

const searchHighlightKey = new PluginKey('searchHighlight')

export const SearchHighlight = Extension.create<{ term: string }>({
  name: 'searchHighlight',

  addOptions() {
    return { term: '' }
  },

  addProseMirrorPlugins() {
    return [
      new Plugin({
        key: searchHighlightKey,
        state: {
          init: (_, state) => buildDecorations(state.doc, this.options.term),
          apply: (tr, old) => {
            if (!tr.docChanged && !tr.getMeta(searchHighlightKey)) return old
            return buildDecorations(tr.doc, this.options.term)
          },
        },
        props: {
          decorations(state) {
            return searchHighlightKey.getState(state)
          },
        },
      }),
    ]
  },
})

function buildDecorations(doc: any, term: string): DecorationSet {
  if (!term) return DecorationSet.empty
  const decorations: Decoration[] = []
  const regex = new RegExp(term, 'gi')

  doc.descendants((node: any, pos: number) => {
    if (!node.isText) return
    let match: RegExpExecArray | null
    while ((match = regex.exec(node.text ?? '')) !== null) {
      decorations.push(
        Decoration.inline(pos + match.index, pos + match.index + match[0].length, {
          class: 'search-highlight',
        })
      )
    }
  })

  return DecorationSet.create(doc, decorations)
}
```

### Widget decoration (e.g. inline AI loading spinner)

```ts
function createSpinner(): HTMLElement {
  const el = document.createElement('span')
  el.className = 'ai-spinner'
  el.textContent = '…'
  return el
}

// Inside a plugin's apply():
const widget = Decoration.widget(cursorPos, createSpinner, {
  side: 1,      // render after cursor
  key: 'ai-spinner',
})
```

---

## 8. React Node Views

Node Views replace a node's default DOM rendering with a live React component.

### Step 1 – Create the React component

```tsx
import { NodeViewWrapper, NodeViewContent } from '@tiptap/react'
import type { NodeViewProps } from '@tiptap/core'

export function CalloutView({ node, updateAttributes, selected }: NodeViewProps) {
  const { type } = node.attrs

  return (
    <NodeViewWrapper className={`callout callout--${type} ${selected ? 'is-selected' : ''}`}>
      <select
        value={type}
        onChange={e => updateAttributes({ type: e.target.value })}
        contentEditable={false}   // prevent ProseMirror from handling events inside
      >
        <option value="info">ℹ Info</option>
        <option value="warning">⚠ Warning</option>
        <option value="error">✖ Error</option>
      </select>

      {/* NodeViewContent renders the editable children */}
      <NodeViewContent className="callout__content" />
    </NodeViewWrapper>
  )
}
```

### Step 2 – Wire up via `addNodeView`

```ts
import { ReactNodeViewRenderer } from '@tiptap/react'
import { CalloutView } from './CalloutView'

export const Callout = Node.create({
  name: 'callout',
  // ... rest of node config

  addNodeView() {
    return ReactNodeViewRenderer(CalloutView, {
      // Optional: custom wrapper tag (default 'div')
      as: 'div',
      // Optional: extra classes on the outer wrapper
      className: 'callout-wrapper',
      // Optional: fine-grained update control
      update: ({ oldNode, newNode }) => oldNode.attrs.type === newNode.attrs.type,
    })
  },
})
```

### `NodeViewProps` reference

```ts
interface NodeViewProps {
  editor: Editor
  node: ProseMirrorNode           // current node data
  decorations: Decoration[]       // decorations applied to this node
  innerDecorations: DecorationSource
  selected: boolean               // true when node is selected
  extension: Node                 // the extension instance
  getPos: () => number | undefined  // current document position
  updateAttributes: (attrs: Record<string, any>) => void  // update node attrs
  deleteNode: () => void          // remove this node from the document
}
```

### `NodeViewWrapper` & `NodeViewContent`

- **`NodeViewWrapper`** — must be the root element; sets up event delegation to ProseMirror
- **`NodeViewContent`** — renders the node's editable `contentDOM`; props like `as`, `className` control the wrapper element

---

## 9. Toolbar — Fixed & Bubble Menu

### Fixed toolbar (always visible)

```tsx
import { useCurrentEditor } from '@tiptap/react'

export function Toolbar() {
  const { editor } = useCurrentEditor()
  if (!editor) return null

  return (
    <div className="toolbar">
      {/* Marks */}
      <ToolbarButton
        onClick={() => editor.chain().focus().toggleBold().run()}
        active={editor.isActive('bold')}
        disabled={!editor.can().chain().focus().toggleBold().run()}
        label="Bold"
      />
      <ToolbarButton
        onClick={() => editor.chain().focus().toggleItalic().run()}
        active={editor.isActive('italic')}
        label="Italic"
      />
      <ToolbarButton
        onClick={() => editor.chain().focus().toggleHighlight({ color: '#ffeb3b' }).run()}
        active={editor.isActive('highlight')}
        label="Highlight"
      />

      {/* Headings */}
      {([1, 2, 3] as const).map(level => (
        <ToolbarButton
          key={level}
          onClick={() => editor.chain().focus().toggleHeading({ level }).run()}
          active={editor.isActive('heading', { level })}
          label={`H${level}`}
        />
      ))}

      {/* Alignment */}
      <ToolbarButton
        onClick={() => editor.chain().focus().setTextAlign('left').run()}
        active={editor.isActive({ textAlign: 'left' })}
        label="Align Left"
      />

      {/* Lists */}
      <ToolbarButton
        onClick={() => editor.chain().focus().toggleBulletList().run()}
        active={editor.isActive('bulletList')}
        label="Bullet List"
      />

      {/* Undo / Redo */}
      <ToolbarButton onClick={() => editor.chain().focus().undo().run()} label="Undo" />
      <ToolbarButton onClick={() => editor.chain().focus().redo().run()} label="Redo" />
    </div>
  )
}
```

### Bubble menu (appears on text selection)

```tsx
import { BubbleMenu } from '@tiptap/react'

export function SelectionBubbleMenu({ editor }: { editor: Editor }) {
  return (
    <BubbleMenu
      editor={editor}
      tippyOptions={{ duration: 100 }}
      shouldShow={({ state, from, to }) => {
        // Only show for non-empty text selections
        return from !== to && state.selection.$from.parent.type.name !== 'codeBlock'
      }}
    >
      <div className="bubble-menu">
        <button onClick={() => editor.chain().focus().toggleBold().run()}>B</button>
        <button onClick={() => editor.chain().focus().toggleItalic().run()}>I</button>
        <button onClick={() => editor.chain().focus().toggleUnderline().run()}>U</button>
        <button onClick={() => editor.chain().focus().toggleHighlight().run()}>H</button>
        <button onClick={() => editor.chain().focus().toggleLink({ href: '' }).run()}>🔗</button>
      </div>
    </BubbleMenu>
  )
}
```

### Floating menu (appears when cursor is on empty line)

```tsx
import { FloatingMenu } from '@tiptap/react'

export function BlockInsertMenu({ editor }: { editor: Editor }) {
  return (
    <FloatingMenu
      editor={editor}
      shouldShow={({ state }) => {
        const { $from } = state.selection
        return $from.parent.content.size === 0
      }}
    >
      <div className="floating-menu">
        <button onClick={() => editor.chain().focus().toggleHeading({ level: 1 }).run()}>
          H1
        </button>
        <button onClick={() => editor.chain().focus().toggleBulletList().run()}>
          List
        </button>
        <button onClick={() => editor.chain().focus().setCallout({ type: 'info' }).run()}>
          Callout
        </button>
      </div>
    </FloatingMenu>
  )
}
```

---

## 10. Notion-Style Slash Commands

Slash commands use `@tiptap/suggestion` as the underlying engine, exactly like `@tiptap/extension-mention`.

### Step 1 – Define the slash command list

```ts
export interface SlashCommandItem {
  title: string
  description: string
  icon: string
  command: (props: { editor: Editor; range: Range }) => void
}

export const SLASH_COMMANDS: SlashCommandItem[] = [
  {
    title: 'Heading 1',
    description: 'Large section heading',
    icon: 'H1',
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).setHeading({ level: 1 }).run(),
  },
  {
    title: 'Heading 2',
    description: 'Medium section heading',
    icon: 'H2',
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).setHeading({ level: 2 }).run(),
  },
  {
    title: 'Bullet List',
    description: 'Simple unsorted list',
    icon: '•',
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).toggleBulletList().run(),
  },
  {
    title: 'Ordered List',
    description: 'Numbered list',
    icon: '1.',
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).toggleOrderedList().run(),
  },
  {
    title: 'Callout',
    description: 'Highlighted info block',
    icon: 'ℹ',
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).setCallout({ type: 'info' }).run(),
  },
  {
    title: 'Code Block',
    description: 'Code with syntax highlighting',
    icon: '<>',
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).setCodeBlock().run(),
  },
  {
    title: 'Image',
    description: 'Upload or embed an image',
    icon: '🖼',
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).setImage({ src: '' }).run(),
  },
  {
    title: 'Table',
    description: 'Insert a table',
    icon: '⊞',
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range)
        .insertTable({ rows: 3, cols: 3, withHeaderRow: true }).run(),
  },
  {
    title: 'Ask AI',
    description: 'Generate content with AI',
    icon: '✨',
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).openAIPrompt().run(),
  },
]
```

### Step 2 – Build the React dropdown component

```tsx
import { forwardRef, useEffect, useImperativeHandle, useState } from 'react'
import type { SlashCommandItem } from './commands'

export interface SlashCommandListRef {
  onKeyDown: (event: KeyboardEvent) => boolean
}

interface Props {
  items: SlashCommandItem[]
  command: (item: SlashCommandItem) => void
}

export const SlashCommandList = forwardRef<SlashCommandListRef, Props>(
  ({ items, command }, ref) => {
    const [selectedIndex, setSelectedIndex] = useState(0)

    useEffect(() => setSelectedIndex(0), [items])

    useImperativeHandle(ref, () => ({
      onKeyDown({ key }) {
        if (key === 'ArrowUp') {
          setSelectedIndex(i => (i - 1 + items.length) % items.length)
          return true
        }
        if (key === 'ArrowDown') {
          setSelectedIndex(i => (i + 1) % items.length)
          return true
        }
        if (key === 'Enter') {
          command(items[selectedIndex])
          return true
        }
        return false
      },
    }))

    return (
      <div className="slash-command-list">
        {items.map((item, i) => (
          <button
            key={item.title}
            className={`slash-command-item ${i === selectedIndex ? 'is-selected' : ''}`}
            onClick={() => command(item)}
          >
            <span className="slash-command-item__icon">{item.icon}</span>
            <div>
              <div className="slash-command-item__title">{item.title}</div>
              <div className="slash-command-item__description">{item.description}</div>
            </div>
          </button>
        ))}
        {items.length === 0 && (
          <div className="slash-command-item slash-command-item--empty">No results</div>
        )}
      </div>
    )
  }
)
```

### Step 3 – Create the Slash Command extension

```ts
import { Extension, Range } from '@tiptap/core'
import { Suggestion } from '@tiptap/suggestion'
import { ReactRenderer } from '@tiptap/react'
import tippy, { type Instance as TippyInstance } from 'tippy.js'
import { SLASH_COMMANDS, type SlashCommandItem } from './commands'
import { SlashCommandList, type SlashCommandListRef } from './SlashCommandList'

export const SlashCommand = Extension.create({
  name: 'slashCommand',

  addProseMirrorPlugins() {
    return [
      Suggestion({
        editor: this.editor,

        char: '/',                   // trigger character
        startOfLine: false,          // can be triggered anywhere on a line
        allowSpaces: false,
        allowedPrefixes: [' '],      // only trigger after whitespace or line start

        // Filter items based on what user types after "/"
        items: ({ query }: { query: string }) => {
          return SLASH_COMMANDS.filter(item =>
            item.title.toLowerCase().includes(query.toLowerCase()) ||
            item.description.toLowerCase().includes(query.toLowerCase())
          ).slice(0, 10)
        },

        // Execute the selected command
        command: ({ editor, range, props }: {
          editor: Editor
          range: Range
          props: SlashCommandItem
        }) => {
          props.command({ editor, range })
        },

        // Render the popup
        render: () => {
          let component: ReactRenderer<SlashCommandListRef>
          let popup: TippyInstance[]

          return {
            onStart(props) {
              component = new ReactRenderer(SlashCommandList, {
                props,
                editor: props.editor,
              })

              popup = tippy('body', {
                getReferenceClientRect: props.clientRect as any,
                appendTo: () => document.body,
                content: component.element,
                showOnCreate: true,
                interactive: true,
                trigger: 'manual',
                placement: 'bottom-start',
              })
            },

            onUpdate(props) {
              component.updateProps(props)
              popup[0].setProps({ getReferenceClientRect: props.clientRect as any })
            },

            onKeyDown(props) {
              if (props.event.key === 'Escape') {
                popup[0].hide()
                return true
              }
              return component.ref?.onKeyDown(props.event) ?? false
            },

            onExit() {
              popup[0].destroy()
              component.destroy()
            },
          }
        },
      }),
    ]
  },
})
```

---

## 11. Collaborative Editing with Hocuspocus

### Server setup (`server.ts`)

```ts
import { Server } from '@hocuspocus/server'
import { Database } from '@hocuspocus/extension-database'

const server = Server.configure({
  port: 1234,

  extensions: [
    new Database({
      // Fetch stored document (called on first connection)
      fetch: async ({ documentName }) => {
        const row = await db.findOne({ name: documentName })
        return row?.content ?? null   // return Uint8Array (Y.js update) or null
      },
      // Persist document changes
      store: async ({ documentName, state }) => {
        await db.upsert({ name: documentName, content: state })
      },
    }),
  ],

  // Auth hook
  async onAuthenticate({ token }) {
    const user = await verifyToken(token)
    if (!user) throw new Error('Unauthorized')
    return { user }  // stored in connection context
  },

  // User presence
  async onConnect({ context }) {
    console.log('User connected', context.user)
  },
})

server.listen()
```

### Client setup

```ts
import * as Y from 'yjs'
import { HocuspocusProvider } from '@hocuspocus/provider'
import { Collaboration } from '@tiptap/extension-collaboration'
import { CollaborationCaret } from '@tiptap/extension-collaboration-caret'

// 1. Create a shared Y.js document
const ydoc = new Y.Doc()

// 2. Connect to Hocuspocus WebSocket server
const provider = new HocuspocusProvider({
  url: 'ws://localhost:1234',
  name: 'my-document-id',       // document identifier
  document: ydoc,
  token: 'user-auth-token',     // passed to onAuthenticate hook

  onSynced() {
    console.log('Synced with server')
  },
  onStatus({ status }) {
    console.log('Connection status:', status)
  },
})

// 3. Add to TipTap extensions
const extensions = [
  StarterKit.configure({ history: false }),  // IMPORTANT: disable built-in history

  Collaboration.configure({
    document: ydoc,
    field: 'default',           // Y.js fragment name (one doc can have multiple fields)
  }),

  CollaborationCaret.configure({
    provider,
    user: {
      name: 'Alice',
      color: '#3b82f6',
    },
    render(user) {
      const el = document.createElement('span')
      el.className = 'collab-caret'
      el.style.borderColor = user.color
      const label = document.createElement('div')
      label.className = 'collab-caret__label'
      label.style.backgroundColor = user.color
      label.textContent = user.name
      el.appendChild(label)
      return el
    },
  }),
]
```

### Presence & awareness

```ts
// Update the local user's information at any time
editor.commands.updateUser({ name: 'Alice', color: '#3b82f6', avatar: '/alice.png' })

// Read all connected users
const users = editor.storage.collaborationCaret.users
// => [{ clientId: 1, name: 'Alice', color: '#3b82f6' }, ...]

// Use the Y.js awareness API directly
provider.setAwarenessField('cursor', { pos: editor.state.selection.from })
const states = provider.awareness.getStates()
```

### Offline / conflict handling

Y.js uses a **CRDT** model — conflicts are resolved automatically. No manual merge logic is needed. The `onContentError` editor event fires if the loaded Y.js state is incompatible with the schema:

```ts
useEditor({
  onContentError({ disableCollaboration }) {
    // Safe fallback: load document without collaboration
    disableCollaboration()
  },
})
```

---

## 12. Track Changes

TipTap does not ship a built-in track-changes extension. The recommended approach is to build on top of Y.js transactions and ProseMirror decorations.

### Data model

Each tracked change is stored as a node attribute plus a global registry:

```ts
interface TrackedChange {
  id: string                        // UUID
  type: 'insertion' | 'deletion' | 'format_change'
  author: {
    id: string
    name: string
    color: string
  }
  createdAt: number                  // Unix ms
  data?: {
    from?: number                    // original positions (for deletions)
    to?: number
    marks?: string[]                 // changed marks (for format changes)
  }
}
```

### Step 1 – Track change extension

```ts
import { Extension } from '@tiptap/core'
import { Plugin, PluginKey } from '@tiptap/pm/state'
import { Decoration, DecorationSet } from '@tiptap/pm/view'
import { v4 as uuid } from 'uuid'
import type { TrackedChange } from './types'

const trackChangesKey = new PluginKey<TrackChangesState>('trackChanges')

interface TrackChangesState {
  enabled: boolean
  changes: Map<string, TrackedChange>
  decorations: DecorationSet
}

declare module '@tiptap/core' {
  interface Commands<ReturnType> {
    trackChanges: {
      enableTracking: () => ReturnType
      disableTracking: () => ReturnType
      acceptChange: (changeId: string) => ReturnType
      rejectChange: (changeId: string) => ReturnType
      acceptAllChanges: () => ReturnType
      rejectAllChanges: () => ReturnType
    }
  }
}

export const TrackChanges = Extension.create<{
  enabled: boolean
  author: { id: string; name: string; color: string }
}>({
  name: 'trackChanges',

  addOptions() {
    return {
      enabled: false,
      author: { id: '', name: 'Unknown', color: '#888' },
    }
  },

  addStorage() {
    return {
      changes: new Map<string, TrackedChange>(),
      enabled: this.options.enabled,
    }
  },

  addCommands() {
    return {
      enableTracking: () => ({ tr, dispatch }) => {
        if (dispatch) tr.setMeta(trackChangesKey, { type: 'SET_ENABLED', enabled: true })
        return true
      },
      disableTracking: () => ({ tr, dispatch }) => {
        if (dispatch) tr.setMeta(trackChangesKey, { type: 'SET_ENABLED', enabled: false })
        return true
      },
      acceptChange: (changeId) => ({ tr, dispatch, state }) => {
        // Remove the tracking mark, keep the content
        if (dispatch) {
          const change = trackChangesKey.getState(state)?.changes.get(changeId)
          if (!change) return false
          if (change.type === 'deletion') {
            // Remove the content that was marked for deletion
            this.editor.chain().unsetMark('trackedDeletion').run()
          } else {
            // Remove insertion marker, keep text
            this.editor.chain().unsetMark('trackedInsertion').run()
          }
          tr.setMeta(trackChangesKey, { type: 'REMOVE_CHANGE', changeId })
        }
        return true
      },
      rejectChange: (changeId) => ({ tr, dispatch, state }) => {
        if (dispatch) {
          const change = trackChangesKey.getState(state)?.changes.get(changeId)
          if (!change) return false
          if (change.type === 'insertion') {
            // Delete the inserted content
            tr.delete(change.data!.from!, change.data!.to!)
          } else if (change.type === 'deletion') {
            // Restore the deleted content (requires storing it)
            this.editor.chain().unsetMark('trackedDeletion').run()
          }
          tr.setMeta(trackChangesKey, { type: 'REMOVE_CHANGE', changeId })
        }
        return true
      },
      acceptAllChanges: () => ({ commands }) => {
        // Accept changes in reverse document order (preserves positions)
        const state = trackChangesKey.getState(this.editor.state)
        if (!state) return false
        const ids = [...state.changes.keys()]
        ids.forEach(id => commands.acceptChange(id))
        return true
      },
      rejectAllChanges: () => ({ commands }) => {
        const state = trackChangesKey.getState(this.editor.state)
        if (!state) return false
        const ids = [...state.changes.keys()]
        ids.forEach(id => commands.rejectChange(id))
        return true
      },
    }
  },

  addProseMirrorPlugins() {
    const author = this.options.author

    return [
      new Plugin<TrackChangesState>({
        key: trackChangesKey,

        state: {
          init: () => ({
            enabled: this.options.enabled,
            changes: new Map(),
            decorations: DecorationSet.empty,
          }),

          apply: (tr, pluginState, _oldState, newState) => {
            const meta = tr.getMeta(trackChangesKey)

            let { enabled, changes } = pluginState

            if (meta?.type === 'SET_ENABLED') {
              enabled = meta.enabled
            }

            if (meta?.type === 'REMOVE_CHANGE') {
              changes = new Map(changes)
              changes.delete(meta.changeId)
            }

            // Intercept content changes when tracking is enabled
            if (enabled && tr.docChanged) {
              changes = new Map(changes)

              tr.steps.forEach((step, stepIndex) => {
                const stepMap = step.getMap()
                stepMap.forEach((oldStart, oldEnd, newStart, newEnd) => {
                  if (newEnd > newStart) {
                    // Content was inserted
                    const id = uuid()
                    changes.set(id, {
                      id,
                      type: 'insertion',
                      author,
                      createdAt: Date.now(),
                      data: { from: newStart, to: newEnd },
                    })
                  }
                  if (oldEnd > oldStart) {
                    // Content was deleted
                    const id = uuid()
                    changes.set(id, {
                      id,
                      type: 'deletion',
                      author,
                      createdAt: Date.now(),
                    })
                  }
                })
              })
            }

            // Rebuild decorations
            const decorations = buildTrackingDecorations(newState.doc, changes)

            return { enabled, changes, decorations }
          },
        },

        props: {
          decorations(state) {
            return trackChangesKey.getState(state)?.decorations ?? DecorationSet.empty
          },
        },
      }),
    ]
  },
})

function buildTrackingDecorations(doc: any, changes: Map<string, TrackedChange>): DecorationSet {
  const decos: Decoration[] = []

  changes.forEach(change => {
    if (change.type === 'insertion' && change.data?.from != null && change.data?.to != null) {
      decos.push(
        Decoration.inline(change.data.from, change.data.to, {
          class: `tracked-insertion`,
          style: `border-bottom: 2px solid ${change.author.color}; background: ${change.author.color}22`,
          'data-change-id': change.id,
          'data-author': change.author.name,
        })
      )
    }
  })

  return DecorationSet.create(doc, decos)
}
```

### Step 2 – Track Changes UI panel

```tsx
interface TrackChangesPanelProps {
  editor: Editor
}

export function TrackChangesPanel({ editor }: TrackChangesPanelProps) {
  const [changes, setChanges] = useState<TrackedChange[]>([])

  useEffect(() => {
    const update = () => {
      const state = editor.state
      const pluginState = trackChangesKey.getState(state)
      setChanges([...(pluginState?.changes.values() ?? [])])
    }
    editor.on('transaction', update)
    return () => editor.off('transaction', update)
  }, [editor])

  return (
    <div className="track-changes-panel">
      <div className="track-changes-panel__header">
        <h3>Changes ({changes.length})</h3>
        <button onClick={() => editor.commands.acceptAllChanges()}>Accept All</button>
        <button onClick={() => editor.commands.rejectAllChanges()}>Reject All</button>
      </div>

      {changes.map(change => (
        <div key={change.id} className="track-change-item">
          <div
            className="track-change-item__avatar"
            style={{ backgroundColor: change.author.color }}
          >
            {change.author.name[0]}
          </div>
          <div className="track-change-item__body">
            <div className="track-change-item__author">{change.author.name}</div>
            <div className="track-change-item__type">{change.type}</div>
            <div className="track-change-item__date">
              {new Date(change.createdAt).toLocaleString()}
            </div>
          </div>
          <div className="track-change-item__actions">
            <button onClick={() => editor.commands.acceptChange(change.id)}>✓</button>
            <button onClick={() => editor.commands.rejectChange(change.id)}>✗</button>
          </div>
        </div>
      ))}
    </div>
  )
}
```

### Integrating with Y.js (collaborative track changes)

Store the change metadata in a Y.js Map so all users share the same change list:

```ts
// In the Hocuspocus provider setup
const yChanges = ydoc.getMap<TrackedChange>('trackChanges')

// When a change is created:
yChanges.set(change.id, change)

// When accepted/rejected:
yChanges.delete(changeId)

// Subscribe to remote changes:
yChanges.observe(() => {
  // Trigger a decoration rebuild
  editor.commands.refreshDecorations()
})
```

---

## 13. AI Content Generation Extension

The extension is **provider-agnostic**: it depends on an abstract `AIProvider` interface that you implement for OpenAI, Claude, or any other LLM.

### Provider interface

```ts
export interface AIStreamChunk {
  text: string
  done: boolean
}

export interface AIProvider {
  /**
   * Stream a completion given a prompt and optional document context.
   * Each call to the async generator yields a text chunk.
   */
  stream(options: {
    prompt: string
    context?: string        // surrounding text sent as context
    signal?: AbortSignal    // for cancellation
  }): AsyncGenerator<AIStreamChunk>

  /**
   * Non-streaming single-shot completion.
   */
  complete(options: {
    prompt: string
    context?: string
    signal?: AbortSignal
  }): Promise<string>
}
```

### OpenAI implementation example

```ts
export class OpenAIProvider implements AIProvider {
  constructor(private apiKey: string, private model = 'gpt-4o') {}

  async *stream({ prompt, context, signal }: {
    prompt: string
    context?: string
    signal?: AbortSignal
  }): AsyncGenerator<AIStreamChunk> {
    const response = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      signal,
      headers: {
        Authorization: `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: this.model,
        stream: true,
        messages: [
          { role: 'system', content: 'You are a helpful writing assistant.' },
          ...(context ? [{ role: 'user', content: `Context:\n${context}` }] : []),
          { role: 'user', content: prompt },
        ],
      }),
    })

    const reader = response.body!.getReader()
    const decoder = new TextDecoder()

    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      const lines = decoder.decode(value).split('\n').filter(l => l.startsWith('data: '))
      for (const line of lines) {
        const data = line.slice(6)
        if (data === '[DONE]') { yield { text: '', done: true }; return }
        const json = JSON.parse(data)
        const text = json.choices?.[0]?.delta?.content ?? ''
        if (text) yield { text, done: false }
      }
    }
  }

  async complete(opts: { prompt: string; context?: string; signal?: AbortSignal }) {
    let result = ''
    for await (const chunk of this.stream(opts)) {
      result += chunk.text
    }
    return result
  }
}
```

### Anthropic Claude implementation example

```ts
export class ClaudeProvider implements AIProvider {
  constructor(private apiKey: string, private model = 'claude-sonnet-4-6') {}

  async *stream({ prompt, context, signal }: {
    prompt: string
    context?: string
    signal?: AbortSignal
  }): AsyncGenerator<AIStreamChunk> {
    const response = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      signal,
      headers: {
        'x-api-key': this.apiKey,
        'anthropic-version': '2023-06-01',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: this.model,
        max_tokens: 2048,
        stream: true,
        messages: [
          {
            role: 'user',
            content: context
              ? `Context:\n${context}\n\n${prompt}`
              : prompt,
          },
        ],
      }),
    })

    const reader = response.body!.getReader()
    const decoder = new TextDecoder()

    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      const lines = decoder.decode(value).split('\n').filter(l => l.startsWith('data: '))
      for (const line of lines) {
        const json = JSON.parse(line.slice(6))
        if (json.type === 'content_block_delta') {
          yield { text: json.delta?.text ?? '', done: false }
        }
        if (json.type === 'message_stop') {
          yield { text: '', done: true }
          return
        }
      }
    }
  }

  async complete(opts: { prompt: string; context?: string; signal?: AbortSignal }) {
    let result = ''
    for await (const chunk of this.stream(opts)) result += chunk.text
    return result
  }
}
```

### AI TipTap extension

```ts
import { Extension } from '@tiptap/core'
import type { AIProvider } from './AIProvider'

declare module '@tiptap/core' {
  interface Commands<ReturnType> {
    ai: {
      /** Insert AI-generated text at the current cursor position (streaming) */
      aiWrite: (prompt: string) => ReturnType
      /** Replace selected text with AI-generated content */
      aiReplace: (prompt: string) => ReturnType
      /** Continue writing from the cursor position */
      aiContinue: () => ReturnType
      /** Open the AI prompt dialog (handled by UI layer) */
      openAIPrompt: () => ReturnType
      /** Cancel an in-flight AI request */
      aiCancel: () => ReturnType
    }
  }

  interface Storage {
    ai: {
      isGenerating: boolean
      abortController: AbortController | null
      generatedText: string
    }
  }
}

export interface AIExtensionOptions {
  provider: AIProvider
  /** Characters of context to send before the cursor (default 2000) */
  contextLength?: number
}

export const AI = Extension.create<AIExtensionOptions, {
  isGenerating: boolean
  abortController: AbortController | null
  generatedText: string
}>({
  name: 'ai',

  addOptions() {
    return {
      provider: null as any,
      contextLength: 2000,
    }
  },

  addStorage() {
    return {
      isGenerating: false,
      abortController: null,
      generatedText: '',
    }
  },

  addCommands() {
    return {
      aiWrite: (prompt: string) => ({ editor, commands }) => {
        if (this.storage.isGenerating) return false
        this._streamInsert(editor, prompt)
        return true
      },

      aiReplace: (prompt: string) => ({ editor }) => {
        if (this.storage.isGenerating) return false
        const { from, to } = editor.state.selection
        editor.chain().focus().deleteRange({ from, to }).run()
        this._streamInsert(editor, prompt)
        return true
      },

      aiContinue: () => ({ editor }) => {
        const context = editor.getText().slice(-this.options.contextLength!)
        return editor.commands.aiWrite(`Continue this text:\n${context}`)
      },

      openAIPrompt: () => () => {
        // Dispatch a custom event; the UI layer listens and shows a dialog
        document.dispatchEvent(new CustomEvent('tiptap:ai:open-prompt'))
        return true
      },

      aiCancel: () => () => {
        this.storage.abortController?.abort()
        this.storage.isGenerating = false
        return true
      },
    }
  },

  // Internal method (not a command)
  // @ts-ignore – private helper attached to `this`
  async _streamInsert(editor: Editor, prompt: string) {
    const controller = new AbortController()
    this.storage.abortController = controller
    this.storage.isGenerating = true
    this.storage.generatedText = ''

    const context = editor.getText().slice(-(this.options.contextLength ?? 2000))

    // Show a placeholder decoration while generating
    const { from } = editor.state.selection

    try {
      for await (const chunk of this.options.provider.stream({
        prompt,
        context,
        signal: controller.signal,
      })) {
        if (chunk.done) break
        // Insert each streamed chunk at the latest cursor position
        editor.chain().focus().insertContentAt(
          editor.state.selection.from,
          chunk.text
        ).run()
        this.storage.generatedText += chunk.text
      }
    } catch (err: any) {
      if (err.name !== 'AbortError') {
        console.error('AI generation failed', err)
      }
    } finally {
      this.storage.isGenerating = false
      this.storage.abortController = null
    }
  },
})
```

### AI Toolbar button component

```tsx
export function AIToolbarButton({ editor }: { editor: Editor }) {
  const [isOpen, setIsOpen] = useState(false)
  const [prompt, setPrompt] = useState('')
  const isGenerating = editor.storage.ai?.isGenerating ?? false

  return (
    <div className="ai-toolbar">
      {isGenerating ? (
        <button onClick={() => editor.commands.aiCancel()} className="ai-cancel-btn">
          ⏹ Stop
        </button>
      ) : (
        <button onClick={() => setIsOpen(true)} className="ai-btn">
          ✨ Ask AI
        </button>
      )}

      {isOpen && (
        <div className="ai-prompt-dialog">
          <textarea
            value={prompt}
            onChange={e => setPrompt(e.target.value)}
            placeholder="What should the AI write?"
            onKeyDown={e => {
              if (e.key === 'Enter' && !e.shiftKey) {
                e.preventDefault()
                editor.commands.aiWrite(prompt)
                setPrompt('')
                setIsOpen(false)
              }
            }}
          />
          <div className="ai-prompt-actions">
            <button onClick={() => {
              editor.commands.aiWrite(prompt)
              setPrompt('')
              setIsOpen(false)
            }}>Generate</button>
            <button onClick={() => editor.commands.aiContinue()}>Continue writing</button>
            {editor.state.selection.from !== editor.state.selection.to && (
              <button onClick={() => {
                editor.commands.aiReplace(prompt)
                setIsOpen(false)
              }}>Replace selection</button>
            )}
            <button onClick={() => setIsOpen(false)}>Cancel</button>
          </div>
        </div>
      )}
    </div>
  )
}
```

---

## 14. Custom Schema Reference

### Document schema (default TipTap structure)

```
doc
├── block+
│   ├── paragraph          (text inline*)
│   ├── heading            (text inline*, attrs: level 1-6)
│   ├── bulletList         (listItem+)
│   │   └── listItem       (paragraph block*)
│   ├── orderedList        (listItem+)
│   ├── blockquote         (block+)
│   ├── codeBlock          (text, attrs: language)
│   ├── horizontalRule
│   ├── image              (attrs: src, alt, title)
│   ├── table              (tableRow+)
│   │   └── tableRow       (tableCell | tableHeader)+
│   ├── callout            (block+, attrs: type)      ← custom
│   └── ...custom nodes
```

### Inline schema

```
inline
├── text         (with marks: bold, italic, underline, strike, code,
│                             highlight, link, textStyle, spoiler…)
├── hardBreak
├── mention      (attrs: id, label)
└── inlineMath   (attrs: formula)  ← custom
```

### Adding a node to an existing group

```ts
// Your custom node automatically inherits schema membership via the group property
const CustomBlock = Node.create({
  name: 'customBlock',
  group: 'block',          // ← makes it usable wherever "block" content is expected
  content: 'inline*',
  // ...
})
```

---

## 15. Command System

### Command helpers available inside `addCommands`

```ts
addCommands() {
  return {
    myCommand: (arg) => (helpers) => {
      const {
        tr,           // current transaction
        state,        // editor state
        view,         // editor view
        dispatch,     // dispatch fn — if null, this is a "can()" check
        editor,       // editor instance
        commands,     // all other commands
        chain,        // command chain builder
        can,          // can() checker
      } = helpers

      // Guard: only dispatch if not in "can()" check mode
      if (dispatch) {
        tr.insertText(arg)
      }

      return true   // return true = command ran successfully
    },
  }
}
```

### Command chaining

```ts
// Chain multiple commands — only dispatches once
editor
  .chain()
  .focus()
  .deleteRange({ from: 0, to: 10 })
  .insertContent('<h1>Hello</h1>')
  .setTextAlign('center')
  .run()

// Check if a command can run (without dispatching)
const canBold = editor.can().toggleBold()
```

### Built-in commands reference

```ts
// Content manipulation
editor.commands.insertContent(value)              // HTML string, JSON, or Node
editor.commands.insertContentAt(pos, value)
editor.commands.setContent(content, emitUpdate?)
editor.commands.clearContent(emitUpdate?)
editor.commands.deleteRange({ from, to })
editor.commands.deleteSelection()
editor.commands.deleteNode(typeOrName)

// Selection
editor.commands.setTextSelection({ from, to })
editor.commands.setNodeSelection(pos)
editor.commands.selectAll()
editor.commands.selectParentNode()
editor.commands.focus(position?, options?)

// Node transforms
editor.commands.setNode(typeOrName, attrs?)
editor.commands.toggleNode(typeOrName, fallback, attrs?)
editor.commands.wrapIn(typeOrName, attrs?)
editor.commands.lift(typeOrName)
editor.commands.liftEmptyBlock()
editor.commands.splitBlock(options?)
editor.commands.joinForward()
editor.commands.joinBackward()

// Mark operations
editor.commands.setMark(typeOrName, attrs?)
editor.commands.toggleMark(typeOrName, attrs?)
editor.commands.unsetMark(typeOrName)
editor.commands.unsetAllMarks()

// History
editor.commands.undo()
editor.commands.redo()
```

---

## 16. Events & Lifecycle Hooks

### Editor events (via `editor.on(event, handler)`)

```ts
editor.on('beforeCreate', ({ editor }) => {})
editor.on('create', ({ editor }) => {})
editor.on('update', ({ editor, transaction }) => {})
editor.on('selectionUpdate', ({ editor }) => {})
editor.on('transaction', ({ editor, transaction }) => {})
editor.on('focus', ({ editor, event }) => {})
editor.on('blur', ({ editor, event }) => {})
editor.on('destroy', () => {})
editor.on('contentError', ({ editor, error, disableCollaboration }) => {})

// Remove listener
editor.off('update', handler)
```

### Extension lifecycle hooks (inside extension config)

```ts
Extension.create({
  onBeforeCreate({ editor }) {
    // Fires before ProseMirror view is created
    // Good place to set up external resources
  },
  onCreate({ editor }) {
    // ProseMirror view is ready; safe to call editor.commands.*
  },
  onUpdate({ editor, transaction }) {
    // After every document change
  },
  onDestroy() {
    // Clean up timers, subscriptions, WebSockets, etc.
  },
  dispatchTransaction({ transaction, next }) {
    // Intercept every transaction before dispatch
    // Call next(transaction) to continue; or modify tr first
    next(transaction)
  },
})
```

---

## 17. Full Editor Bootstrap Example

```tsx
// editor/index.tsx
import { useEditor, EditorContent, EditorProvider } from '@tiptap/react'
import StarterKit from '@tiptap/starter-kit'
import { Highlight } from '@tiptap/extension-highlight'
import { Underline } from '@tiptap/extension-underline'
import { TextAlign } from '@tiptap/extension-text-align'
import { TextStyle } from '@tiptap/extension-text-style'
import { Color } from '@tiptap/extension-color'
import { Link } from '@tiptap/extension-link'
import { Image } from '@tiptap/extension-image'
import { Table } from '@tiptap/extension-table'
import { TableRow } from '@tiptap/extension-table-row'
import { TableCell } from '@tiptap/extension-table-cell'
import { TableHeader } from '@tiptap/extension-table-header'
import { Collaboration } from '@tiptap/extension-collaboration'
import { CollaborationCaret } from '@tiptap/extension-collaboration-caret'
import * as Y from 'yjs'
import { HocuspocusProvider } from '@hocuspocus/provider'

import { SlashCommand } from './extensions/SlashCommand'
import { Callout } from './extensions/Callout'
import { TrackChanges } from './extensions/TrackChanges'
import { AI } from './extensions/AI'
import { OpenAIProvider } from './ai/OpenAIProvider'
import { Toolbar } from './components/Toolbar'
import { SelectionBubbleMenu } from './components/SelectionBubbleMenu'
import { BlockInsertMenu } from './components/BlockInsertMenu'
import { TrackChangesPanel } from './components/TrackChangesPanel'

// Yjs & Hocuspocus setup
const ydoc = new Y.Doc()
const provider = new HocuspocusProvider({
  url: 'ws://localhost:1234',
  name: 'my-document',
  document: ydoc,
  token: 'user-token',
})

const currentUser = {
  id: 'user-1',
  name: 'Alice',
  color: '#3b82f6',
}

const aiProvider = new OpenAIProvider(import.meta.env.VITE_OPENAI_API_KEY)

export function RichEditor() {
  const editor = useEditor({
    extensions: [
      // Base: disable history because Collaboration manages it
      StarterKit.configure({
        history: false,
        heading: { levels: [1, 2, 3] },
      }),

      // Formatting marks
      Highlight.configure({ multicolor: true }),
      Underline,
      TextStyle,
      Color,
      Link.configure({
        openOnClick: false,
        HTMLAttributes: { rel: 'noopener noreferrer', target: '_blank' },
      }),

      // Layout
      TextAlign.configure({ types: ['heading', 'paragraph'] }),

      // Media
      Image.configure({ allowBase64: true }),

      // Tables
      Table.configure({ resizable: true }),
      TableRow,
      TableHeader,
      TableCell,

      // Custom nodes
      Callout,

      // Slash commands
      SlashCommand,

      // Collaboration
      Collaboration.configure({ document: ydoc }),
      CollaborationCaret.configure({
        provider,
        user: currentUser,
      }),

      // Track changes
      TrackChanges.configure({
        enabled: false,
        author: currentUser,
      }),

      // AI
      AI.configure({
        provider: aiProvider,
        contextLength: 2000,
      }),
    ],

    content: '',

    onUpdate({ editor }) {
      // Persist JSON to your database as a debounced side-effect
      const json = editor.getJSON()
      debouncedSave(json)
    },

    onContentError({ disableCollaboration }) {
      // Schema mismatch — load doc without collaboration to avoid sync corruption
      disableCollaboration()
    },
  })

  if (!editor) return null

  return (
    <div className="editor-layout">
      <Toolbar editor={editor} />

      <div className="editor-body">
        <div className="editor-area">
          <SelectionBubbleMenu editor={editor} />
          <BlockInsertMenu editor={editor} />
          <EditorContent editor={editor} className="editor-content" />
        </div>

        <TrackChangesPanel editor={editor} />
      </div>
    </div>
  )
}
```

---

## Quick Reference Card

### Create a custom block node
1. `Node.create({ name, group:'block', content:'inline*', parseHTML, renderHTML, addCommands })`
2. Add `addNodeView()` → `ReactNodeViewRenderer(MyComponent)` for rich UI
3. Register in `extensions: [MyNode]`

### Create a custom mark
1. `Mark.create({ name, parseHTML, renderHTML, addCommands, addKeyboardShortcuts })`
2. Augment `declare module '@tiptap/core' { interface Commands<R> { ... } }`

### Create a formatting extension (no schema)
1. `Extension.create({ name, addKeyboardShortcuts, addProseMirrorPlugins })`

### Add a slash command
1. Add entry to `SLASH_COMMANDS` array with `command: ({ editor, range }) => ...`

### Add a toolbar button
1. `editor.chain().focus().myCommand().run()` in `onClick`
2. `editor.isActive('myExtension')` for active state

### Enable collaboration
1. `StarterKit.configure({ history: false })`
2. Add `Collaboration.configure({ document: ydoc })`
3. Add `CollaborationCaret.configure({ provider, user })`

### Stream AI content
1. `editor.commands.aiWrite('your prompt')` — inserts at cursor
2. `editor.commands.aiReplace('prompt')` — replaces selection
3. `editor.commands.aiCancel()` — stops the stream
