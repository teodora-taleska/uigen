# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # Install deps + generate Prisma client + run migrations
npm run dev          # Start dev server at http://localhost:3000 (Turbopack)
npm run build        # Production build
npm start            # Start production server
npm test             # Run all Vitest tests
npm run db:reset     # Reset SQLite database
```

Run a single test file:
```bash
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx
```

## Environment

Add `ANTHROPIC_API_KEY=your-key` to `.env`. Without it, the app uses a `MockLanguageModel` that generates static demo components — useful for UI development without burning API credits.

## Architecture

**UIGen** is a Next.js 15 App Router app where users describe React components in chat and Claude generates them with live preview.

### Data Flow

```
User (Chat) → ChatContext → POST /api/chat → Claude (streamText)
  → tool calls (str_replace_editor / file_manager)
  → VirtualFileSystem (in-memory, no disk)
  → FileSystemContext (state)
  → CodeEditor + PreviewFrame (iframe with Babel + import maps)
  → DB persistence (authenticated users only)
```

### Key Modules

- **`src/app/api/chat/route.ts`** — Streaming API endpoint. Uses Claude Haiku (`claude-haiku-4-5`). Exposes two tools to Claude: `str_replace_editor` (view/create/replace/insert in files) and `file_manager` (rename/delete). Saves project to DB when stream completes.

- **`src/lib/file-system.ts`** — In-memory virtual file system (tree structure). All file I/O goes through this; nothing is written to disk. Serialize/deserialize for DB persistence.

- **`src/lib/contexts/file-system-context.tsx`** — React context wrapping the virtual FS. All components read/write files through this.

- **`src/lib/contexts/chat-context.tsx`** — Uses Vercel AI SDK `useChat`. Passes serialized FS state to the API. Handles tool call callbacks that update the virtual FS.

- **`src/app/main-content.tsx`** — Three-panel layout: chat (left 35%), preview/code toggle (right 65%).

- **`src/components/preview/PreviewFrame.tsx`** — Renders generated JSX in a sandboxed iframe. Transpiles JSX with Babel and injects an import map for `react`, `react-dom`, and Tailwind CDN. Auto-detects entry point (`App.jsx` → `index.jsx` → first file).

- **`src/lib/provider.ts`** — Returns the Anthropic client if `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel`.

- **`src/lib/prompts/generation.tsx`** — System prompt for Claude that governs component generation behavior.

- **`src/actions/`** — Server actions for auth (`signUp`, `signIn`, `signOut`, `getUser`) and project CRUD.

### Database (Prisma + SQLite)

```
User: id, email, password, createdAt, updatedAt
Project: id, name, userId, messages (JSON), data (JSON), createdAt, updatedAt
```

Projects store the entire message history and virtual FS snapshot as JSON blobs.

### Auth

JWT sessions (7-day expiry, stored in an httpOnly cookie), bcrypt password hashing. Anonymous users can generate components but projects are not persisted. `src/middleware.ts` protects `/[projectId]` routes.

## Tech Stack

Next.js 15, React 19, TypeScript, Tailwind CSS v4, Prisma + SQLite, Vercel AI SDK, Anthropic Claude, Monaco Editor, shadcn/ui + Radix UI, Vitest + React Testing Library.
