# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server (Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest unit tests
npm run setup        # Install deps + Prisma generate + DB migrate
npm run db:reset     # Reset database
```

Run a single test file: `npx vitest run src/lib/__tests__/file-system.test.ts`

## Architecture

**UIGen** is an AI-powered React component generator with live preview. It's a Next.js 15 App Router application.

### Data Flow

```
User prompt → ChatProvider (useChat) → POST /api/chat
→ Claude Haiku via AI SDK
→ AI calls str_replace_editor / file_manager tools
→ VirtualFileSystem (in-memory, no disk writes)
→ FileSystemContext triggers refresh
→ Monaco editor + iframe preview update
→ On finish: Prisma saves project (authenticated users only)
```

### Key Abstractions

**Virtual File System** (`src/lib/file-system.ts`) — In-memory file tree (never written to disk). The AI operates on this through two tools:
- `str_replace_editor` (`src/lib/tools/str-replace.ts`) — view, create, str_replace, insert commands
- `file_manager` (`src/lib/tools/file-manager.ts`) — rename, delete commands

**Live Preview** (`src/lib/transform/jsx-transformer.ts`) — Generates HTML with an ES module import map so virtual files can reference each other in the browser, then uses Babel standalone to transpile JSX client-side in a sandboxed iframe.

**Language Model Factory** (`src/lib/provider.ts`) — Returns Claude Haiku 4.5 if `ANTHROPIC_API_KEY` is set, otherwise falls back to a `MockLanguageModel` that generates static React code.

**Contexts** (`src/lib/contexts/`) — Two providers wrap the app:
- `FileSystemProvider` — holds VirtualFileSystem state, exposes file CRUD
- `ChatProvider` — wraps Vercel AI SDK's `useChat`, serializes fileSystem into each request, processes tool calls from AI responses

### Main UI Layout (`src/app/main-content.tsx`)

3-panel resizable layout via `react-resizable-panels`:
1. Left (35%): Chat interface
2. Right (65%): Toggle between Preview (`PreviewFrame` in iframe) and Code (FileTree + Monaco editor)

### Authentication

JWT stored in httpOnly cookie `auth-token` (7-day expiry). Middleware (`src/middleware.ts`) protects `/api/projects` and `/api/filesystem`. Anonymous users can use the app but work is not persisted — tracked in localStorage via `anon-work-tracker.ts`.

### Database

SQLite via Prisma. Schema has two models: `User` and `Project` (with optional `userId` for anonymous). Client generated to `src/generated/prisma`. Run `npx prisma studio` to inspect data.

### Path Alias

`@/*` maps to `./src/*`.

### Tech Stack

- **Frontend**: React 19, Next.js 15, TypeScript, Tailwind CSS v4, shadcn/ui (new-york theme), Monaco Editor, Lucide icons
- **AI**: Vercel AI SDK, Claude Haiku 4.5 (`claude-haiku-4-5`)
- **Backend**: Prisma (SQLite), jose (JWT), bcrypt
- **Testing**: Vitest + @testing-library (jsdom environment)

## Environment Variables

- `ANTHROPIC_API_KEY` — Optional; without it the app uses MockLanguageModel
- `JWT_SECRET` — Defaults to `"development-secret-key"` in dev
