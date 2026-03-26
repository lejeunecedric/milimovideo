# Coding Conventions

**Analysis Date:** 2026-03-26

## Project Overview

Milimo Video is a monorepo with **four distinct sub-projects**, each with its own tooling:
- `web-app/` — React/TypeScript frontend (Vite)
- `backend/` — Python FastAPI server (custom, no linting config)
- `sam3/` — Python package (Meta's SAM 3, uses black/ufmt/usort)
- `flux2/` — Python package (Black Forest Labs, uses ruff)
- `LTX-2/` — Python workspace (Lightricks, uses ruff with extensive rules)

## Naming Patterns

**Files:**
- Frontend: `camelCase.ts` for stores/utils/hooks (e.g., `timelineStore.ts`, `jobPoller.ts`, `useSSE.ts`)
- Frontend: `PascalCase.tsx` for components (e.g., `VisualTimeline.tsx`, `ErrorBoundary.tsx`, `CinematicPlayer.tsx`)
- Frontend store slices: `camelCaseSlice.ts` (e.g., `projectSlice.ts`, `shotSlice.ts`, `playbackSlice.ts`)
- Backend: `snake_case.py` throughout (e.g., `job_utils.py`, `file_utils.py`, `model_engine.py`)

**Functions:**
- Frontend: `camelCase` (e.g., `createProjectSlice`, `pollJobStatus`, `getAssetUrl`)
- Backend: `snake_case` (e.g., `update_job_db`, `broadcast_progress`, `clamp_resolution_for_device`)

**Variables:**
- Frontend: `camelCase` (e.g., `selectedShotId`, `assetRefreshVersion`, `lastJobId`)
- Backend: `snake_case` (e.g., `project_id`, `negative_prompt`, `num_frames`)

**Types (Frontend):**
- `PascalCase` interfaces (e.g., `TimelineState`, `Shot`, `Project`, `ConditioningItem`)
- `PascalCase` type aliases (e.g., `ConditioningType`, `ShotType`)

**Types (Backend):**
- `PascalCase` Pydantic/SQLModel classes (e.g., `ShotConfig`, `ProjectState`, `ParsedScene`)

## Code Style — Frontend (`web-app/`)

**Formatting:**
- No explicit Prettier config detected
- Vite as bundler (`vite.config.ts`)
- 4-space indentation observed consistently
- Single quotes for imports, double quotes in JSX attributes

**Linting:**
- ESLint v9 flat config at `web-app/eslint.config.js`
- Extends: `js.configs.recommended`, `tseslint.configs.recommended`, `reactHooks.configs.flat.recommended`, `reactRefresh.configs.vite`
- Ignores `dist/` directory
- Target: `ecmaVersion: 2020`
- Run command: `npm run lint`

**TypeScript Configuration (`web-app/tsconfig.app.json`):**
- `strict: true` — Full strict mode enabled
- `noUnusedLocals: true`
- `noUnusedParameters: true`
- `noFallthroughCasesInSwitch: true`
- `noUncheckedSideEffectImports: true`
- `erasableSyntaxOnly: true`
- `verbatimModuleSyntax: true`
- Target: `ES2022`, JSX: `react-jsx`
- Module resolution: `bundler`

**Import Order (Frontend):**
1. React / external library imports
2. Internal components and stores
3. Types (using `import type`)
4. Utility/config imports

Example pattern from `projectSlice.ts`:
```typescript
import type { StateCreator } from 'zustand';
import type { TimelineState, ProjectSlice, Project } from '../types';
import { v4 as uuidv4 } from 'uuid';
import { getAssetUrl } from '../../config';
```

## Code Style — Backend (`backend/`)

**Formatting:**
- No explicit formatter config (no black, ruff, or similar)
- 4-space indentation
- Double quotes for strings
- No type annotations on many functions (some use `Optional`, `List` from `typing`)

**Import Order (Backend):**
1. Standard library (`os`, `sys`, `logging`, `json`, etc.)
2. Third-party (`fastapi`, `torch`, `sqlmodel`, etc.)
3. Local imports (`config`, `database`, `events`, etc.)

Example pattern from `server.py`:
```python
import sys
import os
import logging
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request
import config
from database import init_db, engine, Job
```

**Naming:**
- Module-level constants: `UPPER_SNAKE_CASE` (e.g., `API_PORT`, `MEDIA_EXTENSIONS`, `CORS_HEADERS`)
- Class attributes: `snake_case`
- Private methods: Not prefixed (no leading underscore convention strictly followed)

## Code Style — Sub-packages

**SAM 3 (`sam3/pyproject.toml`):**
- Formatter: `black` (line-length 88) + `usort` + `ufmt` with `ruff-api`
- Type checking: `mypy` with `strict` annotations (`disallow_untyped_defs: true`)
- CI: GitHub Actions runs `ufmt` check on PRs to main

**Flux 2 (`flux2/pyproject.toml`):**
- Linter/Formatter: `ruff` v0.6.8 (line-length 110)
- Pre-commit hooks: `ruff --fix`, `ruff --select I` (isort), `ruff-format`, `yamlfmt`
- CI: GitHub Actions runs `ruff check`, `ruff check --select I`, `ruff format --check`

**LTX-2 (`LTX-2/pyproject.toml`):**
- Linter/Formatter: `ruff` v0.14.3+ (line-length 120)
- Extensive rule selection: E, F, W, I, N, ANN, B, A, COM, C4, DTZ, EXE, PIE, T20, PT, SIM, ARG, PTH, ERA, RUF, PL
- CI: `pytest ~=9.0` and `pre-commit >=4.3.0` in dev deps
- Isort config: `known-first-party = ["ltx_core", "ltx_pipelines", "ltx_trainer"]`

## CSS/Styling

**Framework:** Tailwind CSS v4 via `@tailwindcss/vite` plugin

**Patterns:**
- Utility-first Tailwind classes directly in JSX
- Arbitrary values with bracket syntax: `bg-[#050505]`, `z-[100]`
- Conditional classes via template literals with ternary operators:
```tsx
className={`${toast.type === 'error' ? 'bg-red-500/90 text-white' : 'bg-green-500/90 text-white'}`}
```
- Opacity modifiers: `bg-red-900/20`, `border-red-500/50`, `text-white/70`

## State Management

**Pattern:** Zustand with slice architecture

**Store location:** `web-app/src/stores/timelineStore.ts`

**Architecture:**
- Single store (`TimelineState`) composed of 8 slices
- Slices: `projectSlice`, `shotSlice`, `playbackSlice`, `uiSlice`, `trackSlice`, `elementSlice`, `serverSlice`, `modelSlice`
- Each slice uses `StateCreator<TimelineState, [], [], SliceType>` typing
- Middleware: `temporal` (zundo for undo/redo, limit 20) wrapping `persist` (localStorage)
- Persisted data: Only `project` state via `partialize`
- Use `useShallow` for granular selectors to avoid unnecessary re-renders

**Naming convention for slice files:**
```
web-app/src/stores/slices/
├── elementSlice.ts
├── modelSlice.ts
├── playbackSlice.ts
├── projectSlice.ts
├── serverSlice.ts
├── shotSlice.ts
├── trackSlice.ts
└── uiSlice.ts
```

## Error Handling — Backend

**Strategy:** Exception-based with HTTP status codes

**Pattern in routes:**
```python
@router.get("/projects/{project_id}")
def get_project(project_id: str, session: Session = Depends(get_session)):
    project = session.get(Project, project_id)
    if not project:
        raise HTTPException(status_code=404, detail="Project not found")
```

**Pattern in background tasks:**
```python
try:
    # ... generation logic ...
except Exception as e:
    logger.error(f"Render fatally failed: {e}")
    await emit_progress(0, 100, str(e), status="failed")
```

**Pattern in job utils:**
```python
try:
    with Session(engine) as session:
        # ... database operations ...
except Exception as e:
    logger.error(f"Failed to update job DB for {job_id}: {e}")
```

**Key characteristics:**
- `logger.error()` for all caught exceptions
- Graceful degradation — DB failures in helpers are logged, not raised
- Background tasks emit failure status via SSE rather than crashing

## Error Handling — Frontend

**Strategy:** Optimistic updates + toast notifications

**Pattern:**
```typescript
try {
    const res = await fetch(`http://localhost:8000/...`);
    if (!res.ok) throw new Error("Operation failed");
    // handle success
    addToast("Success message", "success");
} catch (e) {
    console.error("Operation failed", e);
    addToast("User-friendly error", "error");
}
```

**Error Boundaries:**
- `ErrorBoundary` wraps entire app in `main.tsx` — catches fatal errors, shows reload button
- `PanelErrorBoundary` for individual UI panels — shows retry button, doesn't crash app
- Both log errors via `console.error`

**Toast system:** Built into Zustand UI slice, types: `'success' | 'error' | 'info'`

## Logging

**Backend:**
- Standard Python `logging` module
- Structured JSON logging to `server.log` via `StructuredFormatter` (rotating, 10MB max, 3 backups)
- Human-readable console output for development
- Log levels: `INFO` for normal operations, `WARNING` for recoverable issues, `ERROR` for failures

**Frontend:**
- `console.log` for dev/debug info (e.g., `[SSE] Connecting...`, `Auto-Saving Project...`)
- `console.error` for all error cases
- `console.warn` for non-critical issues (e.g., `Failed to save last project ID`)

## Documentation

**Root README:** Comprehensive 233-line README with architecture overview, features, setup instructions, key bindings

**Docs directory (`docs/`):**
- `01_system_architecture.md`
- `02_data_models.md`
- `03_ai_pipelines.md`
- `04_frontend_state.md`
- `05_execution_flow.md`
- `06_file_dependency.md`
- `flux2-bible.md`, `ltx2-bible.md`, `sam3-bible.md` — Deep integration docs

**Inline documentation:**
- Backend: Section comments with `# ── Section Name ──` separators (e.g., `# ── GPU Job Limiter ──`, `# ── SSE Backpressure Settings ──`)
- Backend: Docstrings on public functions/classes (e.g., `"""Resolve element image_path..."""`)
- Frontend: JSDoc-style comments on exported functions (e.g., `/** Syncs the status of a specific job ONCE. */`)
- TypeScript interfaces are well-documented with inline comments explaining field purposes

## CI/CD

**SAM 3:** GitHub Actions workflow at `sam3/.github/workflows/format.yml`
- Runs `ufmt` check (black + usort + ruff-api) on PRs to main

**Flux 2:** GitHub Actions workflow at `flux2/.github/workflows/ci.yaml`
- Runs on every push: `ruff check`, `ruff check --select I`, `ruff format --check`

**LTX-2:** `pre-commit` in dev dependencies, `pytest` configured

**Main project (milimovideo):** No CI/CD pipelines detected at root level

## Pre-commit Hooks

**Flux 2 only:** `.pre-commit-config.yaml` with:
- `ruff` linter with `--fix`
- `ruff` isort with `--fix`
- `ruff-format`
- `yamlfmt` for YAML files

**No pre-commit hooks** detected for `web-app/`, `backend/`, or root project.

## Key Conventions Summary

| Aspect | Frontend (`web-app/`) | Backend (`backend/`) |
|--------|----------------------|---------------------|
| **Formatter** | None configured | None configured |
| **Linter** | ESLint v9 (flat config) | None configured |
| **Type safety** | TypeScript strict mode | Pydantic/SQLModel (partial) |
| **Naming files** | camelCase / PascalCase | snake_case |
| **Naming vars** | camelCase | snake_case |
| **State** | Zustand slices + persist | SQLModel + SQLite |
| **Error handling** | try/catch + toasts | try/catch + HTTPException |
| **Logging** | console.* | Python logging (structured) |
| **Indentation** | 4 spaces | 4 spaces |
| **Quotes** | Single (imports) / Double (JSX) | Double |

---

*Convention analysis: 2026-03-26*
