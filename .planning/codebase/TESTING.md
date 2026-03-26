# Testing Patterns

**Analysis Date:** 2026-03-26

## Current State: No Tests

**Critical finding:** The main milimovideo project (`web-app/` and `backend/`) has **zero test files**. There are no unit tests, integration tests, or end-to-end tests for the core application.

The `.gitignore` explicitly excludes backend test files (`backend/test_*.py`), confirming tests have been written at some point but are not committed.

## Test Frameworks Available

**Frontend (`web-app/`):**
- **No test runner installed** — No jest, vitest, or testing-library in `package.json` dependencies
- No test config files (`jest.config.*`, `vitest.config.*`)
- No `test` script in `package.json` (only `dev`, `build`, `lint`, `preview`)

**Backend (`backend/`):**
- **No test framework** in `requirements.txt`
- No pytest, unittest configuration
- No test files in `backend/` directory (excluded from git)

**SAM 3 (`sam3/`):**
- Test framework configured: `pytest` + `pytest-cov` in dev dependencies
- `pyproject.toml` test config:
  ```toml
  [tool.pytest.ini_options]
  testpaths = ["tests"]
  python_files = "test_*.py"
  python_classes = "Test*"
  python_functions = "test_*"
  ```
- No actual `tests/` directory found in the committed code

**Flux 2 (`flux2/`):**
- No test framework in dependencies
- No test configuration

**LTX-2 (`LTX-2/`):**
- `pytest ~=9.0` in dev dependencies
- Only `test_setup.py` — a smoke test that verifies imports work:
  ```python
  import ltx_core
  import ltx_pipelines
  import torch
  # Verifies MPS/CUDA/CPU detection
  ```
- No unit tests for pipeline logic

## Existing "Tests"

**Only file found:** `LTX-2/test_setup.py`

Purpose: Environment verification, not a real test suite.
```python
try:
    import ltx_core
    print("ltx_core imported successfully")
except ImportError as e:
    print(f"ltx_core import failed: {e}")
```

## What Needs Testing (Priority Order)

### Critical — Backend API Routes

**Files that need tests:**
- `backend/routes/projects.py` — CRUD operations, project save/load, shot sync logic, split shot
- `backend/routes/shots.py` — Partial shot updates, field validation
- `backend/routes/elements.py` — Element CRUD, SAM 3 tracking integration
- `backend/routes/jobs.py` — Job status polling, cancellation
- `backend/routes/storyboard.py` — Script parsing, scene commit, batch generation

**Pattern to follow (once framework is added):**
```python
# tests/test_routes_projects.py
import pytest
from fastapi.testclient import TestClient
from backend.server import app

client = TestClient(app)

def test_create_project():
    response = client.post("/projects", json={"name": "Test"})
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "Test"
    assert "id" in data

def test_get_project_not_found():
    response = client.get("/projects/nonexistent")
    assert response.status_code == 404
```

### Critical — Backend Services

**Files that need tests:**
- `backend/services/script_parser.py` — Screenplay, freeform, and numbered parsing modes
- `backend/services/element_matcher.py` — Element matching from script text
- `backend/job_utils.py` — Job queue, cancellation, progress broadcasting
- `backend/database.py` — Model definitions, session management

**Key functions to test:**
- `ScriptParser.parse_script()` with various input formats
- `ScriptParser._detect_mode()` auto-detection logic
- `ScriptParser._infer_shot_type()` keyword matching
- `clamp_resolution_for_device()` in `backend/tasks/video.py`

### Important — Frontend Store Logic

**Files that need tests:**
- `web-app/src/stores/slices/shotSlice.ts` — Shot CRUD, split, reorder, generate
- `web-app/src/stores/slices/projectSlice.ts` — Save/load/delete, snake_case mapping
- `web-app/src/stores/slices/trackSlice.ts` — Track mute/lock/hidden toggles
- `web-app/src/utils/snapEngine.ts` — Snap logic for timeline
- `web-app/src/utils/timelineUtils.ts` — Layout computation

**Pattern to follow:**
```typescript
// tests/stores/projectSlice.test.ts
import { describe, it, expect, vi } from 'vitest';
import { createProjectSlice } from '../../src/stores/slices/projectSlice';

describe('projectSlice', () => {
    it('should map snake_case to camelCase on load', () => {
        // Test the field mapping logic
    });

    it('should handle 404 on load gracefully', async () => {
        // Mock fetch, verify fallback to DEFAULT_PROJECT
    });
});
```

### Nice-to-Have — UI Components

**Components that would benefit from tests:**
- `web-app/src/components/ErrorBoundary.tsx` — Error boundary behavior
- `web-app/src/components/Timeline/VisualTimeline.tsx` — Auto-save hook, timeline layout
- `web-app/src/providers/SSEProvider.tsx` — Connection retry logic, event parsing

## Recommended Testing Setup

### Backend

```bash
# Add to requirements.txt or create requirements-dev.txt
pip install pytest pytest-asyncio httpx

# Run tests
cd backend && python -m pytest tests/ -v
```

**Test directory structure:**
```
backend/
└── tests/
    ├── conftest.py          # Fixtures: test DB, test client
    ├── test_routes/
    │   ├── test_projects.py
    │   ├── test_shots.py
    │   ├── test_elements.py
    │   └── test_jobs.py
    ├── test_services/
    │   ├── test_script_parser.py
    │   └── test_element_matcher.py
    └── test_utils/
        ├── test_job_utils.py
        └── test_file_utils.py
```

### Frontend

```bash
# Add to package.json devDependencies
npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom

# Add test script to package.json
"scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage"
}
```

**Test directory structure:**
```
web-app/
└── tests/
    ├── setup.ts             # jsdom setup
    ├── stores/
    │   ├── projectSlice.test.ts
    │   ├── shotSlice.test.ts
    │   └── uiSlice.test.ts
    ├── utils/
    │   ├── timelineUtils.test.ts
    │   └── snapEngine.test.ts
    └── components/
        ├── ErrorBoundary.test.tsx
        └── VisualTimeline.test.tsx
```

## Coverage

**Current coverage:** 0%

**No coverage requirements** are enforced anywhere in the project.

**No coverage tools** are configured (no `jest --coverage`, no `pytest-cov` in use, no Codecov/Coveralls integration).

## CI/CD Testing

**Current state:** No CI pipeline runs tests.

- SAM 3 CI only checks formatting (`ufmt`)
- Flux 2 CI only checks linting (`ruff`)
- LTX-2 has pytest in dev deps but no CI workflow
- Root project has no `.github/workflows/` directory

**Recommended CI additions:**
```yaml
# .github/workflows/test.yml
name: Tests
on: [push, pull_request]
jobs:
  backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r backend/requirements.txt pytest pytest-asyncio httpx
      - run: python -m pytest backend/tests/ -v --tb=short

  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm ci
        working-directory: web-app
      - run: npm run test:run
        working-directory: web-app
```

## Mocking Patterns (Recommended)

### Backend — Database Mocking

```python
# tests/conftest.py
import pytest
from sqlmodel import SQLModel, create_engine, Session
from sqlmodel.pool import StaticPool

@pytest.fixture
def session():
    engine = create_engine(
        "sqlite://",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    SQLModel.metadata.create_all(engine)
    with Session(engine) as session:
        yield session
```

### Frontend — Fetch Mocking

```typescript
// tests/setup.ts
import { vi } from 'vitest';

global.fetch = vi.fn();

beforeEach(() => {
    vi.mocked(fetch).mockClear();
});
```

## Test Types Status

| Type | Frontend | Backend | SAM 3 | Flux 2 | LTX-2 |
|------|----------|---------|-------|--------|-------|
| **Unit** | ❌ None | ❌ None | ❌ None | ❌ None | ❌ None |
| **Integration** | ❌ None | ❌ None | ❌ None | ❌ None | ❌ None |
| **E2E** | ❌ None | ❌ None | N/A | N/A | N/A |
| **Smoke** | ❌ None | ❌ None | ❌ None | ❌ None | ✅ `test_setup.py` |

---

*Testing analysis: 2026-03-26*
