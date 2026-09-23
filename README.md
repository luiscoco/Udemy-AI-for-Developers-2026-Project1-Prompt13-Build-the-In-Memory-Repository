# Build the In-Memory Repository

This README walks through the steps that were followed to satisfy Prompt 13: creating an in-memory data repository for the backend.

## Goal

Create `apps/backend/src/data/repository.ts` exposing a `createRepository()` factory that loads the fixture data once and serves it through a small set of read/write methods, without ever touching the fixture files on disk.

## Steps followed

1. **Explored the existing project structure** to understand conventions already in place:
   - Looked at `apps/backend/src/domain/reference.ts` and `workOrderLifecycle.ts` to see the existing code style (plain functions, no classes, `.js` extensions in relative imports).
   - Checked the fixture files at the repository root (`data/assets.json`, `data/technicians.json`, `data/work-orders.json`) to confirm their shape.
   - Checked `packages/contract/src/index.ts` and `types.gen.ts` for the shared `Asset`, `Technician`, `WorkOrder`, `Priority`, and `WorkOrderState` types generated from the OpenAPI contract, so the repository would use the same types as the rest of the app.
   - Checked `apps/backend/tsconfig.json` and the root `tsconfig.base.json`. Since `rootDir` is `src` and the fixture files live outside of it (at the repo root `data/` folder), importing the JSON files directly (`import assets from "../../../data/assets.json"`) was ruled out — it would break the `rootDir` constraint. Reading the files at runtime with Node's `fs` module was used instead.

2. **Implemented `createRepository()`** in `apps/backend/src/data/repository.ts`:
   - Loads `assets.json`, `technicians.json`, and `work-orders.json` **once**, at the time `createRepository()` is called (not per-request), using `fs.readFileSync` + `JSON.parse`. The file path is computed from `import.meta.url` so it works regardless of the current working directory.
   - Keeps the parsed arrays in memory inside a closure — this is the "in-memory" part of the repository.
   - Exposes the following methods:
     - `listAssets()` / `listTechnicians()` — return shallow copies of the arrays.
     - `listWorkOrders(filters?)` — supports optional `state` and `priority` filters and always returns a **new array of copied objects**, never the internal array itself.
     - `getWorkOrder(id)` — returns a copy of a single work order, or `undefined` if not found.
     - `saveWorkOrder(updated)` — updates an existing work order in memory (or inserts it if not found), by replacing the entry in the internal array.
     - `assetExists(id)` / `technicianExists(id)` — quick existence checks used for validating references.
     - `references()` — returns the list of existing work order reference codes (used elsewhere to generate the next sequential reference).

3. **Guaranteed the fixtures stay read-only**:
   - The JSON fixture files are only ever read with `readFileSync`, never written back to.
   - All data returned to callers is copied (`{ ...item }` / `.map(...)`), so mutating a returned object can never corrupt the in-memory store, and mutating the in-memory store can never reach the original JSON files on disk.
   - Any changes made through `saveWorkOrder()` live only in the process's memory and are lost when the server restarts — exactly as an in-memory repository is expected to behave.

## File created

- [apps/backend/src/data/repository.ts](apps/backend/src/data/repository.ts)

## Running the project (Windows terminal)

At this point in the course there is **no runnable server or frontend yet** — `apps/backend` only contains domain logic and the repository, and `apps/frontend` only has its `package.json`/`tsconfig.json` (no Vite app wired up yet, no Fastify server entry point yet). So there is no "start the app" command to give yet.

What you **can** run today is the backend test suite. From the repository root, in PowerShell or `cmd.exe`:

```powershell
cd apps\backend
npm install
npm test
```

This runs `vitest run`, which executes the existing tests (e.g. `reference.test.ts`, `workOrderLifecycle.test.ts`) and is the fastest way to check that nothing is broken after each prompt.

Once a Fastify server entry point and a Vite dev server are added in a later prompt, this section will be updated with the actual `npm run dev` (or equivalent) command to launch the app.
