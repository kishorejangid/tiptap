---
name: tiptap-setup
description: Bootstrap a TipTap rich-text editor in React/TypeScript. Use when starting a new TipTap project, configuring the editor, installing packages, or setting up the base EditorContent component with extensions.
---

# TipTap — Project Setup & Editor Initialization

## When to use this skill
- "Set up a TipTap editor in React"
- "Install TipTap"
- "Configure the editor with these extensions"
- "Create the base editor component"

---

## Step 1 — Install packages

```bash
# Core (always required)
npm install @tiptap/core @tiptap/react @tiptap/pm

# Starter extensions (bold, italic, headings, lists, code, blockquote, etc.)
npm install @tiptap/starter-kit

# Common formatting extensions
npm install @tiptap/extension-highlight @tiptap/extension-underline
npm install @tiptap/extension-text-align @tiptap/extension-color @tiptap/extension-text-style
npm install @tiptap/extension-link @tiptap/extension-image

# Tables
npm install @tiptap/extension-table @tiptap/extension-table-row @tiptap/extension-table-cell @tiptap/extension-table-header

# Menus & dropdowns
npm install @tiptap/extension-bubble-menu @tiptap/extension-floating-menu

# Slash command engine
npm install @tiptap/suggestion

# Drag handles (React)
npm install @tiptap/extension-drag-handle-react

# Collaboration (add only when needed)
npm install @tiptap/extension-collaboration @tiptap/extension-collaboration-caret
npm install @hocuspocus/provider yjs

# Code blocks with syntax highlighting (optional)
npm install @tiptap/extension-code-block-lowlight lowlight
```

---

## Step 2 — Minimal editor component

```tsx
// src/components/Editor.tsx
import { useEditor, EditorContent } from '@tiptap/react'
import StarterKit from '@tiptap/starter-kit'
import { Highlight } from '@tiptap/extension-highlight'
import { Underline } from '@tiptap/extension-underline'
import { TextAlign } from '@tiptap/extension-text-align'
import { Link } from '@tiptap/extension-link'

interface EditorProps {
  initialContent?: string
  onChange?: (json: object) => void
  editable?: boolean
}

export function Editor({ initialContent = '<p>Start writing…</p>', onChange, editable = true }: EditorProps) {
  const editor = useEditor({
    extensions: [
      StarterKit.configure({
        heading: { levels: [1, 2, 3] },
        // history: false  ← uncomment ONLY when using Collaboration
      }),
      Highlight.configure({ multicolor: true }),
      Underline,
      TextAlign.configure({ types: ['heading', 'paragraph'] }),
      Link.configure({ openOnClick: false }),
    ],

    content: initialContent,
    editable,

    // IMPORTANT: set false for Next.js / SSR to avoid hydration mismatch
    immediatelyRender: false,

    onCreate({ editor }) {
      // Editor is ready; safe to call editor.commands.*
    },

    onUpdate({ editor }) {
      onChange?.(editor.getJSON())
    },

    onContentError({ disableCollaboration }) {
      // Schema mismatch — load without collab to avoid data loss
      disableCollaboration()
    },
  })

  // Always guard for null during initialization
  if (!editor) return null

  return (
    <div className="editor-container">
      <EditorContent editor={editor} className="editor-content" />
    </div>
  )
}
```

---

## Full EditorOptions Reference

```ts
useEditor({
  extensions: [],           // REQUIRED: Node/Mark/Extension array
  content: '',              // HTML string, TipTap JSON, or ''
  editable: true,           // false = read-only
  autofocus: false,         // 'start' | 'end' | 'all' | pos | boolean
  enableInputRules: true,   // markdown-style shortcuts while typing
  enablePasteRules: true,   // pattern transforms on paste
  immediatelyRender: true,  // false for SSR
  injectCSS: true,          // auto-inject base ProseMirror CSS
  parseOptions: {},         // ProseMirror ParseOptions
  editorProps: {            // ProseMirror EditorProps
    attributes: { class: 'prose max-w-none' },  // attrs on the editable div
    handleDOMEvents: { /* raw DOM event handlers */ },
    handleKeyDown: (view, event) => false,
    handlePaste: (view, event, slice) => false,
    handleDrop: (view, event, slice, moved) => false,
  },

  // Lifecycle — all optional
  onCreate({ editor }) {},
  onUpdate({ editor, transaction }) {},
  onSelectionUpdate({ editor }) {},
  onTransaction({ editor, transaction }) {},
  onFocus({ editor, event }) {},
  onBlur({ editor, event }) {},
  onDestroy() {},
  onContentError({ editor, error, disableCollaboration }) {},
})
```

---

## StarterKit — What's included & how to disable

```ts
StarterKit.configure({
  // Nodes
  document: false,        // rarely disabled
  paragraph: {},
  heading: { levels: [1, 2, 3] },
  blockquote: {},
  bulletList: {},
  orderedList: {},
  codeBlock: {},
  horizontalRule: {},
  hardBreak: {},

  // Marks
  bold: {},
  italic: {},
  strike: {},
  code: {},
  link: {},
  underline: {},

  // Extensions
  dropcursor: { color: '#3b82f6' },
  gapcursor: {},
  history: {},            // set to false when using Collaboration!
})
```

---

## Content Serialization

```ts
// Get content as TipTap JSON (best for persistence)
const json = editor.getJSON()

// Get content as HTML string
const html = editor.getHTML()

// Get plain text
const text = editor.getText()
const text = editor.getText({ blockSeparator: '\n' })

// Load content programmatically
editor.commands.setContent('<h1>Hello</h1><p>World</p>')
editor.commands.setContent({ type: 'doc', content: [...] })  // JSON
editor.commands.clearContent()

// Read-only mode
editor.setEditable(false)
editor.setEditable(true)
```

---

## Multiple Editor Instances

Each call to `useEditor` creates an independent instance. For shared state, lift `editor` to a context:

```tsx
// EditorContext.tsx
import { createContext, useContext } from 'react'
import type { Editor } from '@tiptap/core'

const EditorContext = createContext<Editor | null>(null)

export function useCurrentEditor() {
  const editor = useContext(EditorContext)
  if (!editor) throw new Error('useCurrentEditor must be used inside EditorProvider')
  return editor
}

export function EditorProvider({ editor, children }: { editor: Editor; children: React.ReactNode }) {
  return <EditorContext.Provider value={editor}>{children}</EditorContext.Provider>
}
```

---

## TypeScript tsconfig.json (minimum)

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}
```
