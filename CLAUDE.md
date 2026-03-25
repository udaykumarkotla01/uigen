# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface, and the AI generates React code that is rendered in real-time.

## Tech Stack

- Next.js 15 with App Router
- React 19
- TypeScript
- Tailwind CSS v4
- Prisma with SQLite
- Anthropic Claude AI (via Vercel AI SDK)
- shadcn/ui components
- Monaco Editor for code editing
- Babel Standalone for JSX transformation

## Development Commands

```bash
# Install dependencies and initialize database
npm run setup

# Start development server (uses Turbopack)
npm run dev

# Build for production
npm run build

# Run linting
npm run lint

# Run tests (Vitest)
npm test

# Reset database
npm run db:reset
```

## Architecture

### Virtual File System

The core of the application is a `VirtualFileSystem` class (`src/lib/file-system.ts`) that maintains an in-memory file tree. Files are never written to disk; everything happens in memory.

- Files are stored as `FileNode` objects with type, name, path, and content
- The VFS supports CRUD operations, directory creation, and file renaming
- Files are serialized/deserialized for persistence in the database

### AI Integration

The AI chat flow (`src/app/api/chat/route.ts`):

1. Receives messages and current file system state from the client
2. Reconstructs the `VirtualFileSystem` from serialized data
3. Streams responses from Claude using the Vercel AI SDK
4. AI can invoke tools (`str_replace_editor`, `file_manager`) to modify files
5. On completion, saves the updated state to the database (if authenticated)

**Provider Selection** (`src/lib/provider.ts`):
- Uses Anthropic Claude if `ANTHROPIC_API_KEY` is set
- Falls back to a `MockLanguageModel` that returns static component templates

### Preview System

The preview (`src/components/preview/PreviewFrame.tsx`) renders generated components in an iframe:

1. Transpiles JSX/TSX files using Babel Standalone (`src/lib/transform/jsx-transformer.ts`)
2. Creates blob URLs for each file
3. Generates an import map for module resolution (supports `@/` aliases)
4. Loads third-party dependencies from esm.sh
5. Renders the result in a sandboxed iframe with error boundaries

### State Management

React Context is used for state management:

- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`): Manages the virtual file system, selected file, and handles tool calls from the AI
- `ChatContext` (`src/lib/contexts/chat-context.tsx`): Wraps the Vercel AI SDK's `useChat` hook, sends file system state with each request

### Authentication

JWT-based authentication (`src/lib/auth.ts`):
- Uses `jose` for JWT signing/verification
- HTTP-only cookies for session storage
- Middleware (`src/middleware.ts`) protects authenticated routes

### Database

Prisma with SQLite:
- `User` model: email, password (bcrypt hashed)
- `Project` model: stores serialized messages and file system data
- Generated client outputs to `src/generated/prisma`

## Key File Locations

| Purpose | Path |
|---------|------|
| Virtual File System | `src/lib/file-system.ts` |
| AI Chat API | `src/app/api/chat/route.ts` |
| AI Provider Selection | `src/lib/provider.ts` |
| System Prompt | `src/lib/prompts/generation.tsx` |
| JSX Transformer | `src/lib/transform/jsx-transformer.ts` |
| File System Context | `src/lib/contexts/file-system-context.tsx` |
| Chat Context | `src/lib/contexts/chat-context.tsx` |
| Auth Utilities | `src/lib/auth.ts` |
| Prisma Schema | `prisma/schema.prisma` |

## AI Tools

The AI has access to two tool categories:

1. **str_replace_editor**: File editing operations
   - `create`: Create a new file with content
   - `str_replace`: Replace text in a file
   - `insert`: Insert text at a specific line
   - `view`: View file contents (with optional line range)

2. **file_manager**: File system operations
   - `rename`: Rename/move files
   - `delete`: Delete files

## Testing

Tests use Vitest with jsdom environment and React Testing Library:

```bash
# Run all tests
npm test

# Run a specific test file
npm test -- src/lib/__tests__/file-system.test.ts
```

## Important Notes

- **Node.js Compatibility**: `node-compat.cjs` removes broken `localStorage`/`sessionStorage` globals in Node 25+ during SSR
- **Entry Point**: Generated projects must have a root `/App.jsx` file as the entry point
- **Import Aliases**: The preview system supports `@/` aliases mapping to the root directory
- **CSS**: Tailwind CSS is loaded via CDN in the preview iframe; CSS imports in user code are collected and injected
