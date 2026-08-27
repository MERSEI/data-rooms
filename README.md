# Acme Data Rooms

[![CI](https://github.com/MERSEI/data-rooms/actions/workflows/ci.yml/badge.svg)](https://github.com/MERSEI/data-rooms/actions/workflows/ci.yml)

**Live**: https://acme-data-rooms.vercel.app

A Data Room MVP — an organized repository for securely storing and browsing due-diligence documents, in the spirit of Google Drive / Dropbox. Built as a frontend single-page application with a mocked backend.

## Features

- **Data rooms** — create, rename, delete multiple top-level data rooms, each with its own document tree
- **Folders** — create nested folders to any depth, rename, cascade delete (with a confirmation that counts what's inside)
- **Files** — upload PDFs via button or drag & drop (multi-file), preview in an in-app viewer, rename, download, delete
- **Search** — instant client-side search across the whole data room, results show each item's location
- **Persistence** — everything (including file contents) survives a page reload
- **Polish** — loading skeletons, empty states with calls to action, error states with retry, toasts, optimistic updates with rollback, deep-linkable URLs, responsive layout

A demo data room with folders and openable PDFs is seeded on first launch so the app is populated out of the box.

## Getting started

```bash
npm install
npm run dev      # http://localhost:5173
npm run test     # unit tests (vitest)
npm run build    # type-check + production build
```

No environment variables or backend required.

## Tech stack

React 19 · TypeScript · Vite · Tailwind CSS v4 · shadcn/ui (Base UI) · Zustand · React Router v7 · idb (IndexedDB) · Vitest

## Architecture

```
src/
  api/        # mock "server": async CRUD over IndexedDB with simulated latency
  stores/     # Zustand stores: UI-facing state, optimistic updates + rollback
  lib/        # pure logic: tree operations, validation, formatting (unit-tested)
  features/   # UI by domain: rooms / browser / upload / preview
  components/ # shared building blocks (dialogs, empty/error states) + shadcn ui
```

### Design decisions

**The mock backend is a real API layer, not scattered state.** Components never touch storage directly — they call `api/*` functions that are async, add 150–300 ms of simulated latency, and throw typed `ApiError`s. This forced the UI to handle loading, failure, and race conditions for real (stale responses after fast navigation are ignored), and it means swapping in an actual backend is a matter of reimplementing one directory with `fetch`.

**Flat data model, like a database table.** Every folder/file is an `Entry { id, roomId, parentId, type, name, size, timestamps }` row rather than a nested JSON tree. Renames are O(1), moves would be a one-field update, cascade delete is a subtree walk done in a single IndexedDB transaction, and the schema maps 1:1 onto a future SQL/blob backend. Tree shape (children, breadcrumbs, descendant counts) is derived in `lib/tree.ts` — pure functions covered by unit tests.

**IndexedDB over in-memory.** The task allows storing files in browser memory; IndexedDB (via `idb`) keeps metadata *and* PDF blobs, so uploads survive reloads and deep links keep working. Blobs live in a separate object store so listing a folder never deserializes file contents.

**Optimistic where it's safe, pessimistic where it isn't.** Rename and delete apply instantly and roll back on API failure; create and upload wait for the server since they need server-issued state. Failures surface as toasts.

**Native PDF rendering.** The preview is an `<iframe>` over a blob URL — the browser's built-in PDF viewer is excellent, and it keeps a heavy dependency (pdf.js) out of the bundle. Object URLs are revoked when the preview closes.

### Edge cases handled

- **Duplicate names on upload** → auto-renamed `report.pdf` → `report (1).pdf` (case-insensitive, like Drive), with a toast explaining what happened; multi-file uploads never fail midway
- **Duplicate names on create/rename** → blocked with an inline error in the dialog
- Name validation: trimmed, non-empty, ≤ 255 chars
- Only PDFs up to 50 MB accepted; rejected files are reported (with reasons) without aborting the rest of the batch
- Deleting a folder shows what will be lost ("…and everything inside it (2 folders, 3 files)") in a proper `AlertDialog`
- Dead URLs (deleted/unknown room or folder) → dedicated not-found states with a way back
- Deep folder chains → breadcrumbs collapse the middle with an ellipsis; long names truncate with tooltips
- Storage quota errors → human-readable message instead of a crash

### Testing

`lib/` (tree operations, name collision resolution, validation) is covered by 16 unit tests — that's where the logic with real branching lives. UI flows were verified end-to-end in the browser: create → nest → upload (duplicates, non-PDFs, multi-file) → preview → rename → cascade delete → reload → back/forward navigation.

## Trade-offs & what I'd do next

- **Real backend** — the `api/` layer is the seam: replace IndexedDB calls with `fetch` to a REST/tRPC service, store blobs in S3-style object storage (upload via presigned URLs), keep the same `Entry` schema in Postgres with a `parent_id` index.
- **Auth & permissions** — data rooms are naturally multi-tenant; per-room access (owner / viewer) would come before anything else in a real due-diligence product.
- **Move & bulk operations** — the flat model already supports moving (`parentId` update); it's cut for scope, not for architecture.
- **Full-text search inside PDFs** — extract text at upload time (pdf.js) and index it; the search UI wouldn't change.
- **Bundle size** — a single ~540 kB chunk (React, router, UI kit) is fine for an internal tool MVP; code-splitting the preview and virtualizing very large folders are the obvious next optimizations.
