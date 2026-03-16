---
name: tiptap-ui
description: Build TipTap editor UI — fixed toolbars, bubble menus on text selection, floating menus on empty lines, and Notion-style slash command menus. Use when creating any editor toolbar, menu, or command palette.
---

# TipTap — Toolbar, Bubble Menu, Floating Menu & Slash Commands

## When to use this skill
- "Add a toolbar to the editor"
- "Show a formatting menu when text is selected"
- "Show a block type menu on empty lines"
- "Add slash commands like Notion"
- "Create a command palette triggered by /"

---

## Fixed Toolbar

```tsx
// components/Toolbar.tsx
import type { Editor } from '@tiptap/core'

interface ToolbarProps { editor: Editor }

export function Toolbar({ editor }: ToolbarProps) {
  // Helper: button with active state and disabled state
  const Btn = ({
    label,
    onClick,
    active = false,
    disabled = false,
  }: {
    label: string
    onClick: () => void
    active?: boolean
    disabled?: boolean
  }) => (
    <button
      onClick={onClick}
      disabled={disabled}
      className={`toolbar-btn ${active ? 'is-active' : ''}`}
    >
      {label}
    </button>
  )

  return (
    <div className="toolbar">
      {/* ── Marks ─────────────────────────────────────── */}
      <Btn label="B" onClick={() => editor.chain().focus().toggleBold().run()}
        active={editor.isActive('bold')}
        disabled={!editor.can().chain().focus().toggleBold().run()} />

      <Btn label="I" onClick={() => editor.chain().focus().toggleItalic().run()}
        active={editor.isActive('italic')} />

      <Btn label="U" onClick={() => editor.chain().focus().toggleUnderline().run()}
        active={editor.isActive('underline')} />

      <Btn label="S̶" onClick={() => editor.chain().focus().toggleStrike().run()}
        active={editor.isActive('strike')} />

      <Btn label="H" onClick={() => editor.chain().focus().toggleHighlight().run()}
        active={editor.isActive('highlight')} />

      <Btn label="Code" onClick={() => editor.chain().focus().toggleCode().run()}
        active={editor.isActive('code')} />

      <div className="toolbar-divider" />

      {/* ── Headings ───────────────────────────────────── */}
      {([1, 2, 3] as const).map(level => (
        <Btn
          key={level}
          label={`H${level}`}
          onClick={() => editor.chain().focus().toggleHeading({ level }).run()}
          active={editor.isActive('heading', { level })}
        />
      ))}

      <div className="toolbar-divider" />

      {/* ── Alignment ─────────────────────────────────── */}
      {(['left', 'center', 'right', 'justify'] as const).map(align => (
        <Btn
          key={align}
          label={align[0].toUpperCase()}
          onClick={() => editor.chain().focus().setTextAlign(align).run()}
          active={editor.isActive({ textAlign: align })}
        />
      ))}

      <div className="toolbar-divider" />

      {/* ── Lists ─────────────────────────────────────── */}
      <Btn label="• List" onClick={() => editor.chain().focus().toggleBulletList().run()}
        active={editor.isActive('bulletList')} />

      <Btn label="1. List" onClick={() => editor.chain().focus().toggleOrderedList().run()}
        active={editor.isActive('orderedList')} />

      <div className="toolbar-divider" />

      {/* ── Blocks ────────────────────────────────────── */}
      <Btn label="❝" onClick={() => editor.chain().focus().toggleBlockquote().run()}
        active={editor.isActive('blockquote')} />

      <Btn label="<>" onClick={() => editor.chain().focus().toggleCodeBlock().run()}
        active={editor.isActive('codeBlock')} />

      <Btn label="──" onClick={() => editor.chain().focus().setHorizontalRule().run()} />

      <div className="toolbar-divider" />

      {/* ── History ───────────────────────────────────── */}
      <Btn label="↩" onClick={() => editor.chain().focus().undo().run()}
        disabled={!editor.can().undo()} />

      <Btn label="↪" onClick={() => editor.chain().focus().redo().run()}
        disabled={!editor.can().redo()} />
    </div>
  )
}
```

---

## Bubble Menu (Appears on Text Selection)

```tsx
// components/SelectionMenu.tsx
import { BubbleMenu } from '@tiptap/react'
import type { Editor } from '@tiptap/core'

export function SelectionMenu({ editor }: { editor: Editor }) {
  return (
    <BubbleMenu
      editor={editor}
      tippyOptions={{
        duration: [100, 50],
        placement: 'top',
      }}
      shouldShow={({ state, from, to }) => {
        // Show only for non-empty text selection
        // Hide inside code blocks
        const { $from } = state.selection
        return (
          from !== to &&
          $from.parent.type.name !== 'codeBlock'
        )
      }}
    >
      <div className="bubble-menu">
        <button
          onClick={() => editor.chain().focus().toggleBold().run()}
          className={editor.isActive('bold') ? 'is-active' : ''}
        >B</button>
        <button
          onClick={() => editor.chain().focus().toggleItalic().run()}
          className={editor.isActive('italic') ? 'is-active' : ''}
        >I</button>
        <button
          onClick={() => editor.chain().focus().toggleUnderline().run()}
          className={editor.isActive('underline') ? 'is-active' : ''}
        >U</button>
        <button
          onClick={() => editor.chain().focus().toggleHighlight().run()}
          className={editor.isActive('highlight') ? 'is-active' : ''}
        >H</button>
        <button
          onClick={() => {
            const url = prompt('URL:')
            if (url) editor.chain().focus().setLink({ href: url }).run()
          }}
          className={editor.isActive('link') ? 'is-active' : ''}
        >🔗</button>
        {editor.isActive('link') && (
          <button onClick={() => editor.chain().focus().unsetLink().run()}>unlink</button>
        )}
      </div>
    </BubbleMenu>
  )
}
```

### BubbleMenu props reference

```tsx
<BubbleMenu
  editor={editor}
  tippyOptions={{
    duration: 100,           // animation duration ms
    placement: 'top',        // tippy placement
    maxWidth: 'none',
  }}
  updateDelay={100}          // debounce in ms before repositioning
  shouldShow={({ editor, view, state, from, to }) => boolean}
  // appendTo: where in DOM to mount — useful for z-index/portal issues
  appendTo={() => document.body}
/>
```

---

## Floating Menu (Appears on Empty Line)

```tsx
// components/BlockInsertMenu.tsx
import { FloatingMenu } from '@tiptap/react'
import type { Editor } from '@tiptap/core'

export function BlockInsertMenu({ editor }: { editor: Editor }) {
  return (
    <FloatingMenu
      editor={editor}
      tippyOptions={{ duration: 100, placement: 'left' }}
      shouldShow={({ state }) => {
        const { $from } = state.selection
        const isEmptyLine = $from.parent.content.size === 0
        const isDefaultNode = $from.parent.type.name === 'paragraph'
        return isEmptyLine && isDefaultNode
      }}
    >
      <div className="floating-menu">
        <button onClick={() => editor.chain().focus().toggleHeading({ level: 1 }).run()}>H1</button>
        <button onClick={() => editor.chain().focus().toggleHeading({ level: 2 }).run()}>H2</button>
        <button onClick={() => editor.chain().focus().toggleBulletList().run()}>List</button>
        <button onClick={() => editor.chain().focus().toggleOrderedList().run()}>1. List</button>
        <button onClick={() => editor.chain().focus().toggleCodeBlock().run()}>Code</button>
        <button onClick={() => editor.chain().focus().toggleBlockquote().run()}>Quote</button>
        <button onClick={() => editor.chain().focus().setHorizontalRule().run()}>Divider</button>
      </div>
    </FloatingMenu>
  )
}
```

---

## Notion-Style Slash Commands

### Step 1 — Define the command list

```ts
// slash/commands.ts
import type { Editor, Range } from '@tiptap/core'

export interface SlashItem {
  title: string
  description: string
  icon: string
  keywords?: string[]   // extra search terms
  command: (props: { editor: Editor; range: Range }) => void
}

export const SLASH_ITEMS: SlashItem[] = [
  {
    title: 'Text',
    description: 'Plain paragraph',
    icon: 'T',
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).setParagraph().run(),
  },
  {
    title: 'Heading 1',
    description: 'Large heading',
    icon: 'H1',
    keywords: ['h1', 'title', 'big'],
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).setHeading({ level: 1 }).run(),
  },
  {
    title: 'Heading 2',
    description: 'Medium heading',
    icon: 'H2',
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).setHeading({ level: 2 }).run(),
  },
  {
    title: 'Heading 3',
    description: 'Small heading',
    icon: 'H3',
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).setHeading({ level: 3 }).run(),
  },
  {
    title: 'Bullet List',
    description: 'Unordered list',
    icon: '•',
    keywords: ['ul', 'unordered'],
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).toggleBulletList().run(),
  },
  {
    title: 'Numbered List',
    description: 'Ordered list',
    icon: '1.',
    keywords: ['ol', 'ordered', 'numbered'],
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).toggleOrderedList().run(),
  },
  {
    title: 'Callout',
    description: 'Highlighted info block',
    icon: 'ℹ',
    keywords: ['info', 'tip', 'warn', 'alert'],
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).setCallout({ type: 'info' }).run(),
  },
  {
    title: 'Code Block',
    description: 'Code with syntax highlighting',
    icon: '<>',
    keywords: ['code', 'pre', 'snippet'],
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).setCodeBlock().run(),
  },
  {
    title: 'Blockquote',
    description: 'Indented quote',
    icon: '❝',
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).setBlockquote().run(),
  },
  {
    title: 'Divider',
    description: 'Horizontal rule',
    icon: '──',
    keywords: ['hr', 'rule', 'line'],
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).setHorizontalRule().run(),
  },
  {
    title: 'Image',
    description: 'Upload or embed image',
    icon: '🖼',
    command: ({ editor, range }) => {
      const url = prompt('Image URL:')
      if (url) editor.chain().focus().deleteRange(range).setImage({ src: url }).run()
    },
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
    keywords: ['ai', 'gpt', 'generate', 'write'],
    command: ({ editor, range }) =>
      editor.chain().focus().deleteRange(range).openAIPrompt().run(),
  },
]

export function filterSlashItems(query: string): SlashItem[] {
  const q = query.toLowerCase()
  return SLASH_ITEMS.filter(
    item =>
      item.title.toLowerCase().includes(q) ||
      item.description.toLowerCase().includes(q) ||
      item.keywords?.some(k => k.includes(q))
  ).slice(0, 10)
}
```

### Step 2 — The dropdown React component

```tsx
// slash/SlashList.tsx
import { forwardRef, useEffect, useImperativeHandle, useState } from 'react'
import type { SlashItem } from './commands'

export interface SlashListRef {
  onKeyDown: (event: KeyboardEvent) => boolean
}

interface Props {
  items: SlashItem[]
  command: (item: SlashItem) => void
}

export const SlashList = forwardRef<SlashListRef, Props>(({ items, command }, ref) => {
  const [index, setIndex] = useState(0)

  useEffect(() => setIndex(0), [items])

  useImperativeHandle(ref, () => ({
    onKeyDown(event: KeyboardEvent): boolean {
      if (event.key === 'ArrowUp') {
        setIndex(i => (i - 1 + items.length) % items.length)
        return true
      }
      if (event.key === 'ArrowDown') {
        setIndex(i => (i + 1) % items.length)
        return true
      }
      if (event.key === 'Enter') {
        if (items[index]) command(items[index])
        return true
      }
      return false
    },
  }))

  if (!items.length) {
    return (
      <div className="slash-menu">
        <div className="slash-menu__empty">No commands found</div>
      </div>
    )
  }

  return (
    <div className="slash-menu">
      {items.map((item, i) => (
        <button
          key={item.title}
          className={`slash-menu__item ${i === index ? 'is-selected' : ''}`}
          onClick={() => command(item)}
          onMouseEnter={() => setIndex(i)}
        >
          <span className="slash-menu__icon">{item.icon}</span>
          <div className="slash-menu__text">
            <div className="slash-menu__title">{item.title}</div>
            <div className="slash-menu__desc">{item.description}</div>
          </div>
        </button>
      ))}
    </div>
  )
})
SlashList.displayName = 'SlashList'
```

### Step 3 — The TipTap extension

```ts
// slash/SlashCommand.ts
import { Extension } from '@tiptap/core'
import { Suggestion } from '@tiptap/suggestion'
import { ReactRenderer } from '@tiptap/react'
import tippy, { type Instance as TippyInstance } from 'tippy.js'
import { filterSlashItems, type SlashItem } from './commands'
import { SlashList, type SlashListRef } from './SlashList'

export const SlashCommand = Extension.create({
  name: 'slashCommand',

  addProseMirrorPlugins() {
    return [
      Suggestion<SlashItem>({
        editor: this.editor,

        char: '/',                    // trigger character
        startOfLine: false,
        allowSpaces: false,
        allowedPrefixes: [' '],       // only trigger after space or line start

        items: ({ query }) => filterSlashItems(query),

        command: ({ editor, range, props }) => {
          props.command({ editor, range })
        },

        render: () => {
          let component: ReactRenderer<SlashListRef>
          let popup: TippyInstance[]

          return {
            onStart(props) {
              component = new ReactRenderer(SlashList, {
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
              popup[0]?.setProps({ getReferenceClientRect: props.clientRect as any })
            },

            onKeyDown(props) {
              if (props.event.key === 'Escape') {
                popup[0]?.hide()
                return true
              }
              return component.ref?.onKeyDown(props.event) ?? false
            },

            onExit() {
              popup[0]?.destroy()
              component.destroy()
            },
          }
        },
      }),
    ]
  },
})
```

### Install tippy.js

```bash
npm install tippy.js
```

---

## Putting It All Together

```tsx
// App.tsx
import { useEditor, EditorContent } from '@tiptap/react'
import StarterKit from '@tiptap/starter-kit'
import { Toolbar } from './components/Toolbar'
import { SelectionMenu } from './components/SelectionMenu'
import { BlockInsertMenu } from './components/BlockInsertMenu'
import { SlashCommand } from './slash/SlashCommand'

export function App() {
  const editor = useEditor({
    extensions: [StarterKit, SlashCommand],
    content: '<p>Type / for commands…</p>',
    immediatelyRender: false,
  })

  if (!editor) return null

  return (
    <div className="editor-layout">
      <Toolbar editor={editor} />

      <div className="editor-area">
        <SelectionMenu editor={editor} />
        <BlockInsertMenu editor={editor} />
        <EditorContent editor={editor} />
      </div>
    </div>
  )
}
```

---

## Common Mistakes & Fixes

| Mistake | Fix |
|---------|-----|
| Bubble menu shows on node selection | Add `from !== to` check in `shouldShow` |
| Slash menu doesn't close on Escape | Return `true` from `onKeyDown` when key is `'Escape'` and call `popup[0].hide()` |
| Slash command executes twice | Make sure `deleteRange(range)` is inside the command, not outside |
| Floating menu on non-empty line | Check `$from.parent.content.size === 0` in `shouldShow` |
| Toolbar button always disabled | Don't call `.run()` inside `editor.can().chain().focus().toggleX().run()` — it's not needed for the check |
| `editor.isActive` returns wrong value | Pass attribute object: `editor.isActive({ textAlign: 'center' })` |
| BubbleMenu flickers | Increase `updateDelay` to 150+ ms |
| SlashList `ref.onKeyDown` is undefined | Ensure `forwardRef` + `useImperativeHandle` are both present in the component |
