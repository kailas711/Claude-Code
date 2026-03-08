# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

This repo contains a single app: **`uigen/`** — an AI-powered React component generator with live preview, built with Next.js 15 App Router.

## Commands

All commands are run from `uigen/`:

```bash
# First-time setup
npm run setup          # npm install + prisma generate + prisma migrate dev

# Development
npm run dev            # Start dev server at localhost:3000

# Build & lint
npm run build
npm run lint

# Tests
npm test               # Run all tests with vitest
npx vitest run path/to/file.test.ts  # Run a single test file

# Database
npm run db:reset       # Reset and re-run all migrations (destructive)
```

## Architecture

### Request Flow

1. User types a message in **`ChatInterface`**
2. **`ChatProvider`** (`chat-context.tsx`) calls `useChat` from `@ai-sdk/react`, posting to `/api/chat` with the serialized `VirtualFileSystem` + message history
3. **`/api/chat/route.ts`** reconstructs the `VirtualFileSystem`, streams a response via `streamText`, and provides two tools to Claude:
   - `str_replace_editor` — view/create/edit files via string replacement or line insertion
   - `file_manager` — rename/delete files
4. Tool calls stream back to the client; `onToolCall` in `ChatProvider` dispatches them to `handleToolCall` in **`FileSystemContext`**, which mutates the in-memory `VirtualFileSystem`
5. **`PreviewFrame`** watches `refreshTrigger` from `FileSystemContext`, builds a browser-native ES module import map via `createImportMap()` (using Babel + blob URLs), and renders a sandboxed `<iframe>` with the generated HTML

### Key Abstractions

- **`VirtualFileSystem`** (`src/lib/file-system.ts`): In-memory tree, no disk writes. Serializes to/from a plain `Record<string, FileNode>` for API transport and Prisma storage.
- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`): React context wrapping the VFS. Single source of truth for file state on the client. Exposes `handleToolCall` which maps AI tool calls to VFS mutations.
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`): Wraps `useChat` from Vercel AI SDK, sending full VFS state on each message.
- **JSX Transformer** (`src/lib/transform/jsx-transformer.ts`): Uses `@babel/standalone` to transpile TSX/JSX in the browser. Builds an ES import map with blob URLs, resolves third-party packages via `https://esm.sh/`, and generates placeholder modules for missing local imports.
- **`getLanguageModel()`** (`src/lib/provider.ts`): Returns `anthropic("claude-haiku-4-5")` if `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that returns static components for offline/testing use.

### Auth

JWT-based sessions via `jose`, stored as an `httpOnly` cookie (`auth-token`). `src/middleware.ts` protects `/api/projects` and `/api/filesystem`. Anonymous users can use the app without sign-in; their work is tracked in `sessionStorage` via `anon-work-tracker.ts` and can be claimed upon sign-up.

### Data Model

SQLite via Prisma (schema at `prisma/schema.prisma`). Two models:
- `User`: email + bcrypt-hashed password
- `Project`: stores chat `messages` and VFS `data` as JSON strings; `userId` is nullable to support anonymous projects

Prisma client is generated to `src/generated/prisma/`.

### Environment

Copy `.env.example` or create `.env` in `uigen/`:
```
ANTHROPIC_API_KEY=   # Optional; app works without it using mock provider
JWT_SECRET=          # Optional; defaults to "development-secret-key"
```
