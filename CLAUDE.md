# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack at http://localhost:3000
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run Vitest tests
npm run db:reset     # Reset SQLite database
```

To run a single test file: `npx vitest run src/lib/__tests__/your-file.test.ts`

## Architecture

**UIGen** is an AI-powered React component generator with live preview. Users describe components in chat; Claude generates them into a virtual file system; results render live in an iframe.

### Key Data Flow

1. User sends a message → `ChatContext` posts to `/api/chat`
2. Chat route streams a response using the Vercel AI SDK (`streamText`) with two tools: `str_replace_editor` (view/create/edit files) and `file_manager` (rename/delete)
3. AI tool calls are forwarded to `FileSystemContext.handleToolCall()`, which mutates the in-memory `VirtualFileSystem`
4. `PreviewFrame` watches `refreshTrigger`, transforms all virtual files to blob URLs via Babel + `@babel/standalone`, generates an import map, and injects everything into an iframe's `srcdoc`
5. On stream completion, if authenticated, the full message history + serialized file system are persisted to SQLite via Prisma

### AI Provider (`src/lib/provider.ts`)

- **With `ANTHROPIC_API_KEY`**: uses `claude-haiku-4-5` via `@ai-sdk/anthropic`
- **Without key**: falls back to a `MockLanguageModel` that returns static placeholder components

### Virtual File System (`src/lib/file-system.ts`)

All generated code lives in memory only — nothing is written to disk. `VirtualFileSystem` is a Map-based tree. It serializes to/from plain JSON for database storage and iframe generation.

### Authentication (`src/lib/auth.ts`)

JWT-based via `jose`. Sessions stored in an httpOnly cookie (`auth-token`, 7-day expiry). `JWT_SECRET` env var defaults to `"development-secret-key"`. Anonymous users have their work tracked in `sessionStorage` only.

### Database

SQLite via Prisma. Two models: `User` (email + bcrypt password) and `Project` (stores `messages` and `data` as JSON strings). Prisma client is generated into `src/generated/prisma`.

### Component Layout

Three-panel UI: **Chat** (left, 35%) | **Preview / Code Editor** (right, 65%). The right panel tabs between a live iframe preview and a Monaco-based code editor with a file tree.

### Server vs. Client boundary

- `src/app/page.tsx` — server component; handles redirect logic and project auto-creation
- `src/app/main-content.tsx` — `"use client"`; all interactive state lives here
- `src/actions/` — `"use server"` server actions for auth and project CRUD
- `src/app/api/chat/route.ts` — streaming API route; the only place AI is called
