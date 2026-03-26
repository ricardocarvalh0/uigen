# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + DB migration)
npm run setup

# Development server (Turbopack)
npm run dev

# Build
npm run build

# Lint
npm run lint

# Tests (Vitest + jsdom)
npm test

# Run a single test file
npx vitest src/components/chat/__tests__/ChatInterface.test.tsx

# Reset database
npm run db:reset

# After editing prisma/schema.prisma
npx prisma migrate dev
npx prisma generate
```

## Environment

Copy `.env.example` to `.env` and add `ANTHROPIC_API_KEY`. Without it, a mock provider returns static code (useful for local dev without API costs). `JWT_SECRET` defaults to a dev key.

## Architecture

UIGen is a Next.js 15 App Router app where users describe React components in a chat, Claude generates them, and a live preview renders them in a sandboxed iframe — no files are written to disk.

### Data flow

1. **User sends message** → `ChatContext` (`src/lib/contexts/chat-context.tsx`) calls Vercel AI SDK's `useChat` hook, POSTing to `/api/chat` with current messages + serialized virtual file system.

2. **API route** (`src/app/api/chat/route.ts`) reconstructs the `VirtualFileSystem`, streams a response from Claude using two tools:
   - `str_replace_editor` — create/edit files via string replacement or insert
   - `file_manager` — rename/delete files

3. **Tool calls stream back to client** → `ChatContext.onToolCall` → `FileSystemContext.handleToolCall` applies mutations to the in-memory `VirtualFileSystem`.

4. **Preview** (`src/components/preview/PreviewFrame.tsx`) re-renders whenever `refreshTrigger` increments. It calls `createImportMap` which uses Babel standalone to transpile JSX/TSX in-browser, creates blob URLs for each file, and builds an `<importmap>` pointing local imports to blob URLs and unknown npm packages to `esm.sh`. The resulting HTML is set as `srcdoc` on a sandboxed iframe.

5. **Persistence**: On stream finish, if `projectId` is present and the user is authenticated, the chat messages and serialized file system are saved to the `Project` row in SQLite via Prisma.

### Key abstractions

- **`VirtualFileSystem`** (`src/lib/file-system.ts`) — in-memory tree of `FileNode` objects. Supports CRUD, rename, serialize/deserialize. This is the single source of truth for all generated files.

- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — wraps `VirtualFileSystem` in React state, exposes `handleToolCall` to apply AI tool calls, and manages `selectedFile`.

- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK's `useChat`, wires tool calls to `FileSystemContext`, and tracks anonymous work.

- **JSX transformer** (`src/lib/transform/jsx-transformer.ts`) — client-side Babel transform pipeline: transpiles files, resolves `@/` aliases, creates blob URLs, handles CSS imports, generates import maps, and produces the full preview HTML string.

### Auth

JWT-based sessions stored in httpOnly cookies. `src/lib/auth.ts` is `server-only`. The middleware (`src/middleware.ts`) protects `/api/projects` and `/api/filesystem`. Users can use the app anonymously; anonymous work is tracked in `src/lib/anon-work-tracker.ts` and can be claimed on sign-up.

### Database

SQLite via Prisma. Schema: `User` (email + bcrypt password) → `Project` (name, messages JSON, data JSON). Prisma client is generated to `src/generated/prisma/`.

### UI

shadcn/ui components in `src/components/ui/`. Layout uses `react-resizable-panels` for the chat/preview/editor split. Monaco editor for the code view.

### Testing

Tests live in `__tests__/` subdirectories next to the code they test. Uses Vitest + jsdom + React Testing Library.
