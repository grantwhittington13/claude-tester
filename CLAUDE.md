# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack (http://localhost:3000)
npm run build        # Production build
npm run lint         # ESLint via Next.js
npm run test         # Run all Vitest tests
npm run db:reset     # Reset and re-migrate the SQLite database
```

Run a single test file:
```bash
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx
```

If `npm` is not found (e.g. when Node is managed by nvm), source nvm first:
```bash
export NVM_DIR="$HOME/.nvm" && [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh" && npm run dev
```

## Architecture

UIGen is a Next.js 15 App Router app that generates React components via Claude AI with live preview.

### Core Data Flow

1. User types a prompt in `ChatInterface` → sent to `POST /api/chat`
2. The API route streams a response from Claude (via Vercel AI SDK) using two tools:
   - `str_replace_editor` — `create`, `str_replace`, and `insert` commands that write to the VFS
   - `file_manager` — `rename` and `delete` commands
3. Tool call results are dispatched to `handleToolCall` in `FileSystemContext`, which mutates the `VirtualFileSystem` and triggers a re-render via `refreshTrigger`
4. The preview iframe (`PreviewFrame`) calls `createImportMap()` which Babel-transpiles every VFS file, creates blob URLs, and wires them into an ESM import map. Third-party packages resolve automatically via `esm.sh`; missing local imports get empty placeholder modules. Tailwind CSS is loaded from CDN inside the iframe.
5. The preview entry point defaults to `/App.jsx`; the `@/` alias maps to the VFS root `/`
6. For authenticated users, the full state (messages + VFS) is serialized as JSON and persisted to SQLite via Prisma at the end of each `streamText` call

### Key Modules

- **`src/lib/file-system.ts`** — `VirtualFileSystem` class: in-memory tree with `serialize()` / `deserializeFromNodes()` for JSON round-trips. `serialize()` strips `Map` children; `deserializeFromNodes()` rebuilds them.
- **`src/lib/contexts/file-system-context.tsx`** — React context wrapping the VFS. `handleToolCall` is the single dispatch point for all AI tool results.
- **`src/lib/contexts/chat-context.tsx`** — Thin wrapper around `useAIChat` (Vercel AI SDK). Sends `fileSystem.serialize()` and `projectId` with every request. Also calls `setHasAnonWork` to persist anonymous session state in `sessionStorage`.
- **`src/app/api/chat/route.ts`** — Streaming AI endpoint. Reconstructs VFS from request body, streams with up to 40 tool steps, and saves to DB in `onFinish`.
- **`src/lib/tools/`** — Tool schemas (`str_replace_editor`, `file_manager`) passed to Claude. Each tool is built as a factory that closes over the server-side `VirtualFileSystem` instance.
- **`src/lib/transform/jsx-transformer.ts`** — Babel transpilation pipeline and `createPreviewHTML()`. Handles CSS stripping, `@/` alias resolution, and placeholder generation for missing imports.
- **`src/lib/provider.ts`** — Returns `anthropic("claude-haiku-4-5")` when `ANTHROPIC_API_KEY` is set, otherwise returns `MockLanguageModel` (simulates tool calls for "counter", "form", "card" prompts with `maxSteps: 4`).
- **`src/lib/prompts/generation.tsx`** — System prompt for Claude, cached via Anthropic's `cacheControl: ephemeral`.
- **`src/lib/auth.ts`** — JWT session (7-day expiry, cookie-stored). `getSession()` is called server-side; `useAuth()` is the client hook.
- **`src/lib/anon-work-tracker.ts`** — `sessionStorage`-backed utility to preserve anonymous work across the auth dialog flow.
- **`src/actions/`** — Server actions for auth (`getUser`, sign-in, sign-up) and project CRUD.

### Routing

- `/` — Anonymous users see the editor directly. Authenticated users are redirected to their most recent project, or a new one is created automatically.
- `/:projectId` — Project view; requires authentication, redirects to `/` otherwise.

### Database

SQLite via Prisma. Two models: `User` and `Project`. Projects store `messages` and `data` (VFS state) as serialized JSON strings. Prisma client is generated to `src/generated/prisma/`.

### Auth

JWT-based. Middleware at `src/middleware.ts` reads the session cookie. Anonymous users can generate components but cannot save projects. `useAuth()` hook (`src/hooks/use-auth.ts`) provides auth state client-side.

### Testing

Tests live in `__tests__/` directories co-located with source. Uses Vitest + jsdom + React Testing Library.
