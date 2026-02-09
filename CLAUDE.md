
# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface, Claude generates/modifies files via tool calls, and the result renders in a sandboxed iframe — all using an in-memory virtual file system (no files written to disk).

## Commands

```bash
npm run setup          # Install deps, generate Prisma client, run migrations
npm run dev            # Start dev server (Next.js + Turbopack) on :3000
npm run build          # Production build
npm run lint           # ESLint
npm run test           # Vitest (all tests)
npm run db:reset       # Reset SQLite database (prisma/dev.db)
```

Run a single test file: `npx vitest run src/lib/__tests__/file-system.test.ts`

## Architecture

### Request Flow

1. User types prompt in chat UI
2. `ChatContext` sends messages + serialized VirtualFileSystem to `POST /api/chat`
3. Route handler streams Claude responses using Vercel AI SDK (`streamText`)
4. Claude calls tools (`str_replace_editor`, `file_manager`) to create/modify files
5. Tool calls are handled client-side in `FileSystemContext`, updating VirtualFileSystem state
6. `PreviewFrame` transforms JSX via Babel standalone, creates import maps (esm.sh CDN), and renders in a sandboxed iframe

### Key Abstractions

- **VirtualFileSystem** (`src/lib/file-system.ts`): In-memory tree-based file system with serialize/deserialize for persistence. Core of the entire app — files exist only in state and database, never on disk.
- **ChatContext** (`src/lib/contexts/chat-context.tsx`): Wraps `useAIChat` from `@ai-sdk/react`. Manages messages, streaming, and anonymous work persistence.
- **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`): Manages VFS state and handles AI tool call execution (create, update, delete, rename files).
- **AI Provider** (`src/lib/provider.ts`): Returns Anthropic (`claude-haiku-4-5`) if `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that returns static demo components.
- **Tool definitions** (`src/lib/tools/`): Zod-validated tool schemas for `str_replace_editor` (view/create/str_replace/insert/undo_edit) and `file_manager` (rename/delete).
- **JSX Transformer** (`src/lib/transform/jsx-transformer.ts`): Client-side Babel compilation + import map generation for iframe preview.

### State & Data

- **Database**: SQLite via Prisma. Two models: `User` and `Project`. Project stores messages and file system as JSON strings.
- **Auth**: JWT sessions (7-day expiry) stored in `auth-token` httpOnly cookie. Anonymous users can work without auth; signing in migrates anonymous work to a persisted project.
- **Contexts** are provided at the `MainContent` component level, not in the root layout.

### UI Layout

Three-panel resizable layout (`react-resizable-panels`): chat on the left (35%), preview/code editor on the right (65%). Preview is the default tab; code editor uses Monaco.

## Tech Stack

Next.js 15 (App Router, Turbopack), React 19, TypeScript, Tailwind CSS v4, Prisma/SQLite, Vercel AI SDK with Anthropic, shadcn/ui (new-york style), Monaco Editor, Babel standalone.

## Testing

Vitest with jsdom environment, React Testing Library. Tests live in `__tests__/` directories next to source files.

## Path Alias

`@/*` maps to `./src/*` (configured in tsconfig.json).
