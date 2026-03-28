# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup (installs deps, generates Prisma client, runs migrations)
npm run setup

# Development (Windows — uses `set NODE_OPTIONS=...` for compatibility)
npm run dev

# Build
npm run build

# Run all tests
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Reset database
npm run db:reset
```

## Environment

Copy `.env` and set `ANTHROPIC_API_KEY`. Without it, the app runs in mock mode using pre-built sample components (Counter, ContactForm, Card).

The `node-compat.cjs` file is required at startup — it removes `globalThis.localStorage/sessionStorage` to prevent SSR crashes on Node.js 25+. This is already wired into all npm scripts.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat interface; Claude generates code into a **virtual (in-memory) file system**; the result is Babel-transformed and rendered live in an iframe.

### Request Flow

1. User sends message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. Route calls `streamText()` (Vercel AI SDK) with the Claude model and two tools:
   - `str_replace_editor` — creates/edits files via string replacement
   - `file_manager` — renames/deletes files
3. AI writes to the virtual file system (serialized as JSON in the request body)
4. Client receives streaming updates, reflects changes in the editor and preview
5. On finish, if authenticated, the project (messages + VFS data) is saved to SQLite via Prisma

### Virtual File System (`src/lib/file-system.ts`)

All generated code lives in memory — nothing is written to disk. The `VirtualFileSystem` class manages files/directories with path normalization. It serializes to/from JSON for persistence and API transport. The AI always targets `/App.jsx` as the entry point.

### AI Provider (`src/lib/provider.ts`)

Returns a `claude-haiku-4-5` model when `ANTHROPIC_API_KEY` is set, or a `MockLanguageModel` otherwise. The mock returns realistic hardcoded components and is useful for UI/layout development without API costs.

### Authentication (`src/lib/auth.ts`)

JWT-based sessions stored in httpOnly cookies (7-day expiry) using the `jose` library. Server actions in `src/actions/index.ts` handle sign-up, sign-in, and sign-out. Anonymous users can use the app freely; their in-progress work is tracked in `sessionStorage` (`src/lib/anon-work-tracker.ts`) and can be migrated on sign-up.

### Database (`prisma/schema.prisma`)

SQLite via Prisma. Two models:
- `User` — email + bcrypt password
- `Project` — stores `messages` (JSON) and `data` (serialized VFS JSON); `userId` is nullable to support anonymous projects

### Live Preview (`src/components/preview/PreviewFrame.tsx`)

Generated JSX/TSX is transformed in-browser using `@babel/standalone` (`src/lib/transform/jsx-transformer.ts`) and rendered inside a sandboxed iframe. Import alias `@/` in generated code resolves to other files in the virtual file system.

### Key Contexts

- `src/lib/contexts/chat-context.tsx` — manages chat messages, streams AI responses, syncs VFS state
- `src/lib/contexts/file-system-context.tsx` — wraps `VirtualFileSystem`, exposes file CRUD to components

### Routing

- `/` — home; redirects authenticated users to their most recent project
- `/[projectId]` — project view; validates ownership before rendering
- `/api/chat` — streaming POST endpoint for AI generation

### UI Layout

Three-panel resizable layout (`src/app/main-content.tsx`):
- Left: Chat (`src/components/chat/`)
- Center/Right tabs: Preview iframe | Code editor (Monaco) + file tree
