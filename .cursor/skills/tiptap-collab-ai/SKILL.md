---
name: tiptap-collab-ai
description: Add real-time collaboration with Hocuspocus/Y.js, conflict-free multi-user editing, presence cursors, track-changes/suggestion mode, and AI-assisted writing to a TipTap editor. Use when implementing collaboration, offline sync, change tracking, or AI content generation.
---

# TipTap — Collaboration, Track Changes & AI Extension

## When to use this skill
- "Add real-time collaboration to the editor"
- "Set up Hocuspocus / Y.js"
- "Show live cursors for other users"
- "Add track changes / suggestion mode"
- "Integrate AI writing / generation"
- "Stream AI content into the editor"

---

## Part 1 — Real-Time Collaboration (Hocuspocus + Y.js)

### Installation

```bash
npm install @tiptap/extension-collaboration @tiptap/extension-collaboration-caret
npm install @hocuspocus/provider yjs
```

### Client setup

```tsx
// CollaborativeEditor.tsx
import { useEditor, EditorContent } from '@tiptap/react'
import StarterKit from '@tiptap/starter-kit'
import { Collaboration } from '@tiptap/extension-collaboration'
import { CollaborationCaret } from '@tiptap/extension-collaboration-caret'
import { HocuspocusProvider } from '@hocuspocus/provider'
import * as Y from 'yjs'
import { useEffect, useState } from 'react'

interface CollabEditorProps {
  documentId: string
  user: { name: string; color: string }
  token?: string
}

export function CollaborativeEditor({ documentId, user, token }: CollabEditorProps) {
  const [provider, setProvider] = useState<HocuspocusProvider | null>(null)
  const [ydoc] = useState(() => new Y.Doc())

  useEffect(() => {
    const p = new HocuspocusProvider({
      url: 'wss://your-hocuspocus-server.com',
      name: documentId,
      document: ydoc,
      token,                              // JWT / auth token
      onConnect: () => console.log('connected'),
      onDisconnect: () => console.log('disconnected'),
      onSynced: () => console.log('document synced'),
      onAuthenticationFailed: ({ reason }) => console.error('auth failed', reason),
    })

    setProvider(p)
    return () => p.destroy()
  }, [documentId, ydoc, token])

  const editor = useEditor({
    extensions: [
      StarterKit.configure({
        history: false,  // ← CRITICAL: Y.js manages history, not StarterKit
      }),

      Collaboration.configure({
        document: ydoc,
        field: 'default',   // Y.js XML field name (use different names for separate docs)
      }),

      CollaborationCaret.configure({
        provider,
        user,                // { name: string, color: string }
        render: user => {    // Custom caret/cursor label
          const cursor = document.createElement('span')
          cursor.classList.add('collaboration-cursor')
          cursor.style.setProperty('--caret-color', user.color)
          cursor.setAttribute('data-name', user.name)
          return cursor
        },
      }),
    ],
    immediatelyRender: false,
    onContentError({ disableCollaboration }) {
      // Schema mismatch — safely fall back to local mode
      disableCollaboration()
    },
  }, [provider])  // re-create editor when provider changes

  if (!editor) return null

  return <EditorContent editor={editor} />
}
```

### Hocuspocus server (Node.js)

```ts
// server.ts
import { Server } from '@hocuspocus/server'
import { Database } from '@hocuspocus/extension-database'
import { Logger } from '@hocuspocus/extension-logger'

const server = Server.configure({
  port: 1234,

  extensions: [
    new Logger(),

    new Database({
      fetch: async ({ documentName }) => {
        // Return stored Uint8Array state or null for new documents
        const doc = await db.findOne({ name: documentName })
        return doc?.state ?? null
      },
      store: async ({ documentName, state }) => {
        // state is a Uint8Array — store as binary
        await db.upsert({ name: documentName }, { state })
      },
    }),
  ],

  async onAuthenticate({ token }) {
    // Verify JWT / session token
    // Throw to deny access; return user data to allow
    const user = await verifyToken(token)
    if (!user) throw new Error('Unauthorized')
    return user
  },

  async onConnect({ documentName, context }) {
    // context = value returned from onAuthenticate
    console.log(`${context.name} connected to ${documentName}`)
  },
})

server.listen()
```

### Offline / local-first with IndexedDB

```ts
import { IndexeddbPersistence } from 'y-indexeddb'

const persistence = new IndexeddbPersistence(documentId, ydoc)
persistence.on('synced', () => {
  console.log('Loaded from IndexedDB')
})
// Y.js merges offline edits automatically when reconnecting
```

### Y.js awareness (custom presence data)

```ts
// Set any user data — all clients receive it in real time
provider.setAwarenessField('user', { name: 'Alice', color: '#3b82f6', avatar: '...' })
provider.setAwarenessField('cursor', { anchor: 12, head: 18 })

// Read all connected users
provider.awareness.getStates().forEach((state, clientId) => {
  console.log(clientId, state.user)
})

// Subscribe to changes
provider.awareness.on('change', ({ added, updated, removed }) => {
  // update presence UI
})
```

---

## Part 2 — Track Changes / Suggestion Mode

### Installation

```bash
npm install @tiptap-pro/extension-comments         # For comment annotations
# OR for native track-changes (TipTap Pro):
npm install @tiptap-pro/extension-track-changes
```

### Custom suggestion mode (open-source approach)

This pattern implements suggest/reject inline changes using marks:

```ts
// extensions/SuggestionMark.ts
import { Mark, mergeAttributes } from '@tiptap/core'

declare module '@tiptap/core' {
  interface Commands<ReturnType> {
    suggestionInsert: {
      acceptSuggestion: (id: string) => ReturnType
      rejectSuggestion: (id: string) => ReturnType
    }
  }
}

export const SuggestionInsert = Mark.create({
  name: 'suggestionInsert',
  inclusive: true,
  excludes: '',

  addAttributes() {
    return {
      id: { default: null },
      author: { default: null },
      createdAt: { default: null },
    }
  },

  parseHTML() { return [{ tag: 'ins[data-suggestion]' }] },

  renderHTML({ HTMLAttributes }) {
    return ['ins', mergeAttributes(HTMLAttributes, { 'data-suggestion': '', class: 'suggestion-insert' }), 0]
  },

  addCommands() {
    return {
      acceptSuggestion: id => ({ editor, tr }) => {
        // Remove the mark but keep the text
        editor.view.state.doc.descendants((node, pos) => {
          node.marks.forEach(mark => {
            if (mark.type.name === 'suggestionInsert' && mark.attrs.id === id) {
              tr.removeMark(pos, pos + node.nodeSize, mark.type)
            }
          })
        })
        return true
      },
      rejectSuggestion: id => ({ editor, tr }) => {
        // Remove both mark and text
        editor.view.state.doc.descendants((node, pos) => {
          node.marks.forEach(mark => {
            if (mark.type.name === 'suggestionInsert' && mark.attrs.id === id) {
              tr.delete(pos, pos + node.nodeSize)
            }
          })
        })
        return true
      },
    }
  },
})
```

### Enable suggestion mode via extension

```ts
import { Extension } from '@tiptap/core'

export const SuggestionMode = Extension.create({
  name: 'suggestionMode',

  addStorage() {
    return { enabled: false, currentUserId: '', currentUserName: '' }
  },

  dispatchTransaction({ transaction, next }) {
    if (!this.storage.enabled || !transaction.docChanged) {
      return next(transaction)
    }

    // Wrap inserted text with suggestionInsert mark
    let tr = transaction
    transaction.steps.forEach((step) => {
      // Mark new insertions — advanced: inspect ReplaceStep
    })
    next(tr)
  },
})

// Toggle suggestion mode:
editor.storage.suggestionMode.enabled = true
editor.storage.suggestionMode.currentUserId = 'user-123'
editor.storage.suggestionMode.currentUserName = 'Alice'
```

---

## Part 3 — AI Extension (Streaming Content Generation)

### Installation

```bash
npm install @tiptap-pro/extension-ai  # TipTap Pro AI
# OR build custom (see below)
```

### Custom AI extension (bring-your-own model)

```ts
// extensions/AIExtension.ts
import { Extension } from '@tiptap/core'
import { Decoration, DecorationSet } from '@tiptap/pm/view'
import { Plugin, PluginKey } from '@tiptap/pm/state'

declare module '@tiptap/core' {
  interface Commands<ReturnType> {
    ai: {
      aiGenerate: (prompt: string) => ReturnType
      aiComplete: () => ReturnType
      aiAccept: () => ReturnType
      aiReject: () => ReturnType
    }
  }
}

const aiKey = new PluginKey<{ streaming: boolean; previewText: string }>('ai')

export const AIExtension = Extension.create<
  { endpoint: string; headers?: Record<string, string> },
  { abortController: AbortController | null; streaming: boolean }
>({
  name: 'ai',

  addOptions() {
    return { endpoint: '/api/ai', headers: {} }
  },

  addStorage() {
    return { abortController: null, streaming: false }
  },

  addCommands() {
    return {
      aiGenerate: (prompt) => ({ editor, commands }) => {
        const { from } = editor.state.selection
        this.storage.streaming = true
        this.storage.abortController = new AbortController()

        streamAI({
          endpoint: this.options.endpoint,
          headers: this.options.headers,
          prompt,
          signal: this.storage.abortController.signal,
          onChunk: (text) => {
            // Insert streamed text at cursor
            editor.view.dispatch(
              editor.state.tr.insertText(text, editor.state.selection.from)
            )
          },
          onDone: () => {
            this.storage.streaming = false
          },
          onError: (err) => {
            if (err.name !== 'AbortError') console.error('AI error:', err)
            this.storage.streaming = false
          },
        })

        return true
      },

      aiComplete: () => ({ editor, commands }) => {
        const { from, to } = editor.state.selection
        const selectedText = editor.state.doc.textBetween(from, to)
        const beforeCursor = editor.state.doc.textBetween(0, from)
        return commands.aiGenerate(`Continue this text: ${beforeCursor}`)
      },

      aiAccept: () => ({ editor }) => {
        // Already inserted — just stop streaming
        this.storage.abortController?.abort()
        this.storage.streaming = false
        return true
      },

      aiReject: () => ({ editor, commands }) => {
        this.storage.abortController?.abort()
        this.storage.streaming = false
        commands.undo()
        return true
      },
    }
  },

  addKeyboardShortcuts() {
    return {
      'Mod-Enter': () => this.editor.commands.aiComplete(),
      Escape: () => {
        if (this.storage.streaming) {
          return this.editor.commands.aiReject()
        }
        return false
      },
    }
  },

  onDestroy() {
    this.storage.abortController?.abort()
  },
})

// Streaming helper
async function streamAI({
  endpoint,
  headers = {},
  prompt,
  signal,
  onChunk,
  onDone,
  onError,
}: {
  endpoint: string
  headers?: Record<string, string>
  prompt: string
  signal: AbortSignal
  onChunk: (text: string) => void
  onDone: () => void
  onError: (err: Error) => void
}) {
  try {
    const res = await fetch(endpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', ...headers },
      body: JSON.stringify({ prompt }),
      signal,
    })

    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    if (!res.body) throw new Error('No response body')

    const reader = res.body.getReader()
    const decoder = new TextDecoder()

    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      const chunk = decoder.decode(value, { stream: true })
      // Handle SSE format: "data: ...\n\n"
      chunk.split('\n').forEach(line => {
        if (line.startsWith('data: ')) {
          const data = line.slice(6).trim()
          if (data === '[DONE]') return
          try {
            const json = JSON.parse(data)
            const text = json.choices?.[0]?.delta?.content ?? json.text ?? ''
            if (text) onChunk(text)
          } catch {}
        }
      })
    }

    onDone()
  } catch (err) {
    onError(err as Error)
  }
}
```

### API route (Next.js App Router)

```ts
// app/api/ai/route.ts
import { NextRequest } from 'next/server'
import Anthropic from '@anthropic-ai/sdk'

const client = new Anthropic()

export async function POST(req: NextRequest) {
  const { prompt } = await req.json()

  const stream = await client.messages.stream({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    messages: [{ role: 'user', content: prompt }],
  })

  const encoder = new TextEncoder()
  const readable = new ReadableStream({
    async start(controller) {
      for await (const event of stream) {
        if (event.type === 'content_block_delta' && event.delta.type === 'text_delta') {
          const data = `data: ${JSON.stringify({ text: event.delta.text })}\n\n`
          controller.enqueue(encoder.encode(data))
        }
      }
      controller.enqueue(encoder.encode('data: [DONE]\n\n'))
      controller.close()
    },
  })

  return new Response(readable, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      Connection: 'keep-alive',
    },
  })
}
```

### AI toolbar button

```tsx
<button
  onClick={() => {
    const prompt = window.prompt('What should I write?')
    if (prompt) editor.commands.aiGenerate(prompt)
  }}
  disabled={editor.storage.ai?.streaming}
>
  {editor.storage.ai?.streaming ? 'Generating…' : '✨ AI Write'}
</button>
```

---

## Presence UI — Connected Users List

```tsx
import { useState, useEffect } from 'react'
import type { HocuspocusProvider } from '@hocuspocus/provider'

export function PresenceList({ provider }: { provider: HocuspocusProvider | null }) {
  const [users, setUsers] = useState<Array<{ name: string; color: string; clientId: number }>>([])

  useEffect(() => {
    if (!provider) return
    const update = () => {
      const states = provider.awareness.getStates()
      const list: typeof users = []
      states.forEach((state, clientId) => {
        if (state.user) list.push({ ...state.user, clientId })
      })
      setUsers(list)
    }
    provider.awareness.on('change', update)
    update()
    return () => provider.awareness.off('change', update)
  }, [provider])

  return (
    <div className="presence-list">
      {users.map(u => (
        <div key={u.clientId} className="presence-avatar" style={{ backgroundColor: u.color }} title={u.name}>
          {u.name[0].toUpperCase()}
        </div>
      ))}
    </div>
  )
}
```

---

## Common Mistakes & Fixes

| Mistake | Fix |
|---------|-----|
| Undo/redo doesn't work in collab mode | Set `StarterKit.configure({ history: false })` — Y.js manages undo |
| Two editors show different content | Make sure both use the same `Y.Doc` instance and same `field` name |
| Provider connects but document empty | Check `onAuthenticate` doesn't throw for valid tokens; check `fetch` in Database extension |
| `onContentError` fires on load | Schema mismatch between stored doc and current extensions — call `disableCollaboration()` and reload |
| AI stream inserts at wrong position | Capture `editor.state.selection.from` before the async call, not inside `onChunk` |
| AbortError thrown visibly | Check `err.name !== 'AbortError'` before logging/showing error |
| Streaming text duplicates on re-render | Each `onChunk` should dispatch a single `insertText` — don't batch |
| Collaboration caret color not showing | Ensure `CollaborationCaret` is after `Collaboration` in the extensions array |
