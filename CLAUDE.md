# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Build and start production
npm run build && npm start

# Lint
npm run lint

# Run all tests
npm run test

# Reset database
npm run db:reset
```

Tests use Vitest — to run a single test file:
```bash
npx vitest src/components/chat/__tests__/ChatInterface.test.tsx
```

## Architecture

UIGen is a Next.js 15 App Router application. Users describe React components in chat; Claude generates them using tool calls that operate on an **in-memory virtual file system** (never writes to disk). A live preview iframe renders the result via Babel Standalone.

### Data flow for component generation

1. User submits prompt in `ChatInterface`
2. `ChatContext` (`src/lib/contexts/chat-context.tsx`) sends POST to `/api/chat` with the current serialized file system
3. `/api/chat/route.ts` calls Claude via Vercel AI SDK `streamText` with two tools: `str_replace_editor` and `file_manager`
4. Claude emits tool calls; results stream back to the client
5. `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) processes tool results and updates the `VirtualFileSystem`
6. `PreviewFrame` detects file changes and re-renders via Babel JSX transform
7. On finish, if the user is authenticated, the API saves messages + file state to SQLite via Prisma

### Virtual file system

`src/lib/file-system.ts` — in-memory only, never touches disk. Shared between client (state) and server (tool execution). Serialized to JSON and sent with every chat request so the server is stateless.

### AI provider

`src/lib/provider.ts` — if `ANTHROPIC_API_KEY` is set, uses real Claude (`claude-sonnet-4-5`); otherwise falls back to a `MockLanguageModel` that returns static example components. Mock mode has `maxSteps: 4`, real Claude uses `maxSteps: 40`.

### Tools

- `src/lib/tools/str-replace.ts` — `str_replace_editor` tool: `view`, `create`, `str_replace`, `insert` commands
- `src/lib/tools/file-manager.ts` — `file_manager` tool: `rename`, `delete` commands

System prompt for Claude is in `src/lib/prompts/generation.tsx`.

### State management

Two React contexts wrap the entire app:
- **FileSystemContext** — owns `VirtualFileSystem` instance; processes tool call results
- **ChatContext** — wraps Vercel AI SDK's `useChat`; sends file state with each request

### Authentication

JWT stored in an HTTP-only cookie (7-day expiry). `src/lib/auth.ts` handles session creation/verification. Server actions in `src/actions/index.ts` handle sign-up/sign-in/sign-out. Anonymous users can use the app without signing in; their work is tracked in localStorage via `src/lib/anon-work-tracker.ts`.

Middleware at `src/middleware.ts` protects `/api/projects` and `/api/filesystem` routes.

### Database

Prisma + SQLite (`prisma/dev.db`). Two models: `User` and `Project`. `Project.messages` and `Project.data` are JSON strings storing serialized chat history and file system state respectively. Generated client lives at `src/generated/prisma`.

## Environment

- `ANTHROPIC_API_KEY` — leave empty for mock/demo mode
- `JWT_SECRET` — defaults to `"development-secret-key"` in dev (change in production)
