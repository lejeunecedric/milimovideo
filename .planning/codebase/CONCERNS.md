# Codebase Concerns

**Analysis Date:** 2026-03-26

---

## 1. CRITICAL SECURITY ISSUES

### 1.1 Code Injection via `eval()`

**Risk:** Remote Code Execution (RCE)

**File:** `backend/routes/elements.py:119`
```python
mask_path = await inpainting_manager.get_mask_from_sam(req.image_path, eval(req.points))
```

The `eval()` call executes arbitrary Python from user-supplied `points` string. An attacker can send malicious code through the API and achieve RCE on the server.

**Fix approach:** Replace `eval(req.points)` with `json.loads(req.points)` and validate the resulting structure contains only numeric lists.

---

### 1.2 CORS Wide Open (`allow_origins=["*"]`)

**Risk:** Any origin can make authenticated requests to the API.

**File:** `backend/server.py:112-117`
```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=False,
    allow_methods=["*"],
    allow_headers=["*"],
    ...
)
```

**Impact:** In production, any malicious website can make API calls to the backend. Combined with the lack of authentication (see 1.3), this is a complete security bypass.

**Fix approach:** Restrict `allow_origins` to the frontend URL. Use environment variable for configuration.

---

### 1.3 No Authentication/Authorization

**Risk:** All endpoints publicly accessible without any auth.

**Files:**
- `backend/routes/projects.py` — full CRUD on projects
- `backend/routes/jobs.py` — can trigger GPU-intensive generation
- `backend/routes/models.py:359-375` — can read/write HF tokens
- `backend/routes/elements.py` — can execute SAM operations

**Impact:** Anyone who can reach the server can create/delete projects, consume GPU resources (DoS vector), and exfiltrate HF tokens.

**Fix approach:** At minimum, add API key middleware. For multi-user, add JWT or session-based auth.

---

### 1.4 HuggingFace Token Exposed in Process Memory

**File:** `backend/routes/models.py:366`
```python
os.environ["HF_TOKEN"] = request.token
```

**Risk:** The HF token is stored in the process environment, visible to any code running in the same process. It's also ephemeral — lost on restart.

**Fix approach:** Store encrypted in a secrets file or use a secrets manager. Never store in `os.environ` from an HTTP endpoint.

---

## 2. HIGH SEVERITY CODE QUALITY

### 2.1 Pervasive `any` Type in TypeScript (63+ instances)

**Impact:** Defeats the purpose of TypeScript. Runtime type errors propagate silently.

**Files with most violations:**
- `web-app/src/components/Editor/TrackingPanel.tsx` (12 instances)
- `web-app/src/stores/slices/shotSlice.ts` (7 instances)
- `web-app/src/components/Player/CinematicPlayer.tsx` (7 instances)
- `web-app/src/stores/slices/projectSlice.ts` (5 instances)
- `web-app/src/components/Images/ImagesView.tsx` (5 instances)

**Pattern:**
```typescript
const mapShot = (s: any): Shot => ({  // input untyped
const handleServerEvent: (type: string, data: any) => {  // event data untyped
```

**Fix approach:** Define proper interfaces for API responses. Use Zod or similar runtime validation for incoming data.

---

### 2.2 Bare `except` Clauses (9 instances)

**Impact:** Swallows all exceptions including `KeyboardInterrupt`, `SystemExit`, and real bugs.

**Files:**
- `backend/tasks/video.py:424,649,802`
- `backend/tasks/chained.py:173`
- `backend/routes/assets.py:214,284,327,347`
- `backend/file_utils.py:57`

**Pattern:**
```python
except:
    pass  # Silent failure
```

**Fix approach:** Always catch specific exceptions. At minimum use `except Exception:` if broad catch is needed.

---

### 2.3 Massive Monolithic Files

**Files exceeding 500 lines:**

| File | Lines | Purpose |
|------|-------|---------|
| `backend/models/flux_wrapper.py` | 823 | Flux model wrapper + IP-Adapter + generation |
| `backend/tasks/video.py` | 808 | Video generation with Flux fallback |
| `backend/routes/projects.py` | 689 | Project CRUD + render + split |
| `backend/routes/storyboard.py` | 682 | Storyboard operations |
| `backend/routes/models.py` | 583 | Model management + downloads |
| `backend/routes/elements.py` | 507 | Element CRUD + SAM integration + tracking |
| `web-app/src/components/Editor/TrackingPanel.tsx` | 835 | SAM tracking UI |

**Impact:** Hard to understand, test, and maintain. Single responsibility principle violated.

**Fix approach:** Extract concerns: separate Flux generation logic from IP-Adapter from VAE wrapper in `flux_wrapper.py`. Extract render logic from `projects.py` into a service.

---

### 2.4 Duplicated Prompt Enhancement Logic

**File:** `backend/tasks/video.py`

The same prompt enhancement pattern (Gemma text encoder → LLM enhance → fallback) is repeated verbatim for:
- `ti2vid` pipeline (lines 354-399)
- `ic_lora` pipeline (lines 448-471)
- `keyframe` pipeline (lines 492-514)
- `chained.py` (lines 165-225)

Each copy has ~20 lines of identical logic with slight variations.

**Fix approach:** Extract into a shared function `enhance_pipeline_prompt(pipeline, prompt, input_images, ...)`.

---

### 2.5 Duplicate Import Statement

**File:** `backend/tasks/chained.py:8,15`
```python
import config  # Line 8
...
import config  # Line 15 (duplicate)
```

**Fix approach:** Remove the second import.

---

## 3. HIGH SEVERITY ARCHITECTURE CONCERNS

### 3.1 SQLite with Oversized Connection Pool

**File:** `backend/database.py:131`
```python
engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False}, pool_size=20, max_overflow=40)
```

**Issue:** SQLite doesn't support true concurrent writes. A pool_size of 20 with max_overflow of 40 (total 60 connections) is counterproductive — it will cause `database is locked` errors under load.

**Fix approach:** Use `pool_size=5` with `max_overflow=0` for SQLite, or migrate to PostgreSQL for production.

---

### 3.2 JSON Stored as String in Database

**Files:**
- `backend/database.py:54` — `matched_elements: Optional[str] = None`
- `backend/database.py:69` — `timeline: Optional[str] = None`

**Pattern requiring JSON parsing everywhere:**
- `backend/storyboard/manager.py:312` — `json.loads(shot.matched_elements)`
- `backend/storyboard/manager.py:367` — `json.loads(shot.timeline)`
- `backend/routes/storyboard.py:350,362` — `json.loads(shot.timeline)`, `json.loads(shot.matched_elements)`
- `backend/routes/projects.py:269,283` — serializing/deserializing timeline

**Impact:** Every access requires try/except for JSON parsing. No query ability on fields. Risk of inconsistent data.

**Fix approach:** Use SQLAlchemy JSON column type: `Column(JSON)` instead of `Optional[str]`.

---

### 3.3 No Input Validation on Patch Endpoints

**File:** `backend/routes/projects.py:335-354`
```python
@router.patch("/shots/{shot_id}")
async def update_shot_partial(shot_id: str, updates: dict, session: Session = Depends(get_session)):
    for k, v in updates.items():
        if k == "id" or k == "project_id": continue
        if hasattr(shot, k):
            setattr(shot, k, v)  # Arbitrary field injection
```

**Risk:** Client can set any field on a Shot including `status`, `created_at`, or internal state fields. No type checking.

**Fix approach:** Use a Pydantic model for the update body. Whitelist allowed fields.

---

### 3.4 No Database Migration System

**Issue:** Schema changes require manual intervention or destructive recreation. The codebase includes a one-time migration script (`backend/migrate_v1_v2.py`) but no systematic migration framework.

**Impact:** Deployments risk data loss. Schema evolution is untracked.

**Fix approach:** Integrate Alembic for SQLAlchemy migrations.

---

## 4. FRONTEND CONCERNS

### 4.1 Hardcoded `localhost:8000` URLs (74 instances)

**Files:**
- `web-app/src/config.ts:1` — `export const API_BASE_URL = "http://localhost:8000";`
- `web-app/src/stores/slices/shotSlice.ts` — 11 instances
- `web-app/src/stores/slices/elementSlice.ts` — 8 instances
- `web-app/src/stores/slices/projectSlice.ts` — 4 instances
- `web-app/src/components/Images/ImagesView.tsx` — 14 instances
- `web-app/src/components/Editor/MaskingCanvas.tsx` — uses hardcoded base

**Impact:** Cannot deploy to any environment other than localhost. Some files use `API_BASE_URL` from config, others inline the URL directly — inconsistent pattern.

**Fix approach:** Use `API_BASE_URL` from config consistently. Configure via `VITE_API_URL` environment variable.

---

### 4.2 Missing API_BASE_URL Usage

**Files that define their own constant instead of using config:**
- `web-app/src/components/ModelLibrary/LoRAManager.tsx:9`
- `web-app/src/components/ModelLibrary/ModelSettings.tsx:10`
- `web-app/src/components/Editor/MaskingCanvas.tsx:13`

```typescript
// Inconsistent: Some files define their own
const API_BASE_URL = import.meta.env.VITE_API_URL || 'http://localhost:8000';
// While config.ts already exports API_BASE_URL
```

**Fix approach:** Import from `../../config` instead of redefining.

---

### 4.3 No Error Boundaries on Critical Paths

**File:** `web-app/src/components/ErrorBoundary.tsx` exists but is not wrapped around the main application routes or generation views. A single render error crashes the entire app.

**Fix approach:** Wrap each major view (StoryboardView, ImagesView, TrackingPanel) in an ErrorBoundary.

---

## 5. PERFORMANCE CONCERNS

### 5.1 Synchronous HTTP Calls Inside Async Handlers

**Files:**
- `backend/routes/elements.py:56,178,205,231` — uses `requests.get/post` (blocking) inside FastAPI async handlers
- `backend/routes/projects.py:664` — `requests.get` for Ollama

```python
res = http_requests.get(f"http://localhost:{config.SAM_SERVICE_PORT}/health", timeout=3)
```

**Impact:** Blocks the event loop during SAM/Ollama calls. Under concurrent requests, the entire server stalls.

**Fix approach:** Use `httpx.AsyncClient` or `aiohttp` for async HTTP. Or use `asyncio.to_thread()` to wrap synchronous calls.

---

### 5.2 Full File Read for Media Serving

**File:** `backend/server.py:221-223`
```python
with open(full_path, "rb") as f:
    data = f.read()  # Reads entire file into memory
```

When no Range header is present, the entire file is read into memory. For large video files (hundreds of MB), this causes OOM.

**Fix approach:** Use streaming response (`StreamingResponse`) or `FileResponse` for non-range requests.

---

### 5.3 No Caching for Repeated API Calls

**Issue:** Every `/status/{job_id}` poll creates a new DB session. Every `/projects/{project_id}` reload queries all shots.

**Impact:** Under heavy frontend polling (default SSE + fallback polling), database is hammered.

**Fix approach:** Implement in-memory cache with TTL for frequently accessed but rarely changed data.

---

## 6. RELIABILITY CONCERNS

### 6.1 Global Mutable State for Job Tracking

**File:** `backend/job_utils.py:55`
```python
active_jobs: Dict[str, Dict[str, Any]] = {}
```

**Risk:** All job state is in-process memory. Server restart loses all active job tracking. No recovery mechanism for in-flight GPU work.

**Fix approach:** Use Redis or a persistent job queue (Celery, RQ) for production.

---

### 6.2 No Graceful Shutdown for GPU Work

**File:** `backend/server.py:99-104`
```python
count = job_utils.cancel_all_jobs()
if count > 0:
    await asyncio.sleep(2.0)  # Arbitrary 2s wait
```

**Risk:** GPU operations (model inference) can't be interrupted cleanly. The 2-second sleep is a guess — actual inference may take 45+ seconds.

**Fix approach:** Implement proper cancellation tokens in the inference loop. Use `threading.Event` for cooperative cancellation.

---

### 6.3 Circular Import Risk via `sys.path` Manipulation

**Files:**
- `backend/config.py:49-54` — `sys.path.append` and `sys.path.insert`
- `backend/models/flux_wrapper.py:13-14` — `sys.path.append`
- `backend/model_engine.py:9` — calls `config.setup_paths()`

```python
def setup_paths():
    if LTX_CORE_DIR not in sys.path:
        sys.path.append(LTX_CORE_DIR)
    if LTX_PIPELINES_DIR not in sys.path:
        sys.path.insert(0, LTX_PIPELINES_DIR)
    if LTX_CORE_DIR not in sys.path:  # DUPLICATE CHECK
        sys.path.insert(0, LTX_CORE_DIR)
```

**Issue:** The duplicate check on line 53 is a copy-paste bug (checks LTX_CORE_DIR twice instead of checking LTX_PIPELINES_DIR).

**Fix approach:** Remove duplicate check. Consider using proper Python packaging (pip install -e) instead of sys.path hacks.

---

### 6.4 Frontend SSE Connection Not Resilient

**File:** `web-app/src/providers/SSEProvider.tsx:41`
```typescript
const es = new EventSource('http://localhost:8000/events');
```

**Issues:**
- Hardcoded URL (can't configure for different environments)
- No exponential backoff on reconnection
- No event deduplication (replayed events may be processed twice)

**Fix approach:** Use `API_BASE_URL`, implement exponential backoff, add event ID tracking.

---

## 7. TECHNICAL DEBT

### 7.1 Legacy Path Handling Complexity

**File:** `backend/file_utils.py:60-108`

The `resolve_path` function handles 5 different path formats:
1. Full URLs: `http://localhost:8000/projects/...`
2. Project paths: `/projects/{id}/generated/...`
3. Legacy uploads: `/uploads/...`
4. Legacy generated: `/generated/...`
5. Absolute filesystem paths

**Impact:** Every place that works with paths must understand all formats. Bug-prone and hard to test.

**Fix approach:** Standardize on one format (project-relative URLs). Create a migration to convert legacy paths.

---

### 7.2 Video Path Resolution Duplicated Across 3 Files

**Pattern appears in:**
- `backend/routes/elements.py:290-301` — `track_start` URL resolution
- `backend/file_utils.py:76-105` — `resolve_path` function
- `backend/tasks/video.py:97-136` — input image resolution with fallback chains

Each has slightly different logic for the same URL → filesystem path conversion.

**Fix approach:** Use `resolve_path` everywhere. Remove inline resolution logic.

---

### 7.3 Prompt Enhancement Repeated 4+ Times

**Pattern:**
1. `backend/tasks/video.py:354-399` (ti2vid)
2. `backend/tasks/video.py:448-471` (ic_lora)
3. `backend/tasks/video.py:492-514` (keyframe)
4. `backend/tasks/chained.py:165-225` (chained/chunk 0)

Each copy follows the same ~25-line pattern: get text_encoder → enhance prompt → handle reference → cleanup.

**Fix approach:** Extract `enhance_with_fallback(pipeline, prompt, images, ...)` function.

---

### 7.4 `server_last_request.json` and `server_last_error.txt` Written to CWD

**File:** `backend/tasks/video.py:702-703`
```python
with open("server_last_request.json", "w") as f:
    json.dump(params, f, indent=2)
```

**File:** `backend/tasks/video.py:799-801`
```python
with open("server_last_error.txt", "w") as f:
    f.write(f"Error: {e}\n")
```

**Issue:** Debug files are written to the current working directory (undefined if server starts from different location). Should be in a dedicated debug/logs directory.

---

### 7.5 French Comments in Shell Scripts

**File:** `start.sh:7,15`
```bash
# Fonction de nettoyage
# Intercepter Ctrl+C et kill
```

**Issue:** Mixed languages in codebase (English code, French comments). Inconsistent with the rest of the project.

---

## 8. MISSING BEST PRACTICES

### 8.1 No Test Suite

**Finding:** Zero test files found for the backend. No `pytest.ini`, `conftest.py`, or test configuration.

**Impact:** Any change risks breaking existing functionality with no safety net.

**Fix approach:** Add pytest with FastAPI's TestClient. Start with API endpoint tests.

---

### 8.2 No Rate Limiting

**Issue:** No rate limiting on any endpoint. The GPU-intensive generation endpoints (`/generate/advanced`, `/generate/image`) can be abused.

**Fix approach:** Add `slowapi` or similar rate limiter. Limit generation endpoints to N requests per minute.

---

### 8.3 No Request Size Limits

**Issue:** No explicit request body size limits configured. Large payloads (e.g., base64 images in tracking) could exhaust memory.

**Fix approach:** Configure FastAPI/uvicorn `--limit-request-body` or use middleware.

---

### 8.4 No Health Check Endpoint

**Issue:** No `/health` or `/ready` endpoint for the main backend. The SAM service has one, but the core API doesn't.

**Fix approach:** Add `@app.get("/health")` returning service status, DB connectivity, and GPU availability.

---

## 9. SCALING LIMITS

### 9.1 Single-Process GPU Architecture

**File:** `backend/job_utils.py:16-23`
```python
_gpu_semaphore = asyncio.Semaphore(1)  # Only 1 GPU job at a time
```

**Current capacity:** 1 video/image generation at a time. Queue is in-memory.

**Limit:** Cannot scale beyond single GPU without architecture changes.

**Scaling path:** Migrate to Celery/Redis with GPU worker pool. Multiple workers can serve multiple GPUs.

---

### 9.2 SQLite as Primary Database

**Current:** SQLite with file-based storage.

**Limit:** Single-writer limitation. Cannot run multiple backend instances. No replication.

**Scaling path:** Migrate to PostgreSQL for multi-instance deployment.

---

## 10. DEPENDENCY HEALTH

### 10.1 Unpinned Python Dependencies

**File:** `backend/requirements.txt`

Most dependencies are unpinned:
```
fastapi          # No version
uvicorn          # No version
numpy            # No version
Pillow           # No version
```

Only PyTorch is pinned: `torch==2.10.0`

**Risk:** Builds are not reproducible. Breaking changes in dependencies can cause failures.

**Fix approach:** Pin all versions or use `pip freeze` output. Use a lock file.

---

### 10.2 Missing `requests` in requirements.txt

**File:** `backend/requirements.txt`

The `requests` library is used in multiple files:
- `backend/routes/elements.py:52,174,201,227`
- `backend/routes/projects.py:662`
- `backend/llm.py:9`
- `backend/services/model_downloader.py` (via huggingface_hub)

But `requests` is not listed in `requirements.txt`. It works because `huggingface_hub` depends on it transitively.

**Fix approach:** Add `requests` explicitly to requirements.txt.

---

## SUMMARY

| Severity | Count | Key Items |
|----------|-------|-----------|
| CRITICAL | 4 | `eval()` injection, CORS open, no auth, token exposure |
| HIGH | 8 | `any` types, bare excepts, monoliths, duplicated code, sync in async |
| MEDIUM | 8 | SQLite pool, JSON strings, no validation, hardcoded URLs, no migrations |
| LOW | 6 | Unpinned deps, French comments, debug files, missing health check |

**Top 3 Immediate Actions:**
1. **Fix `eval()` call** in `backend/routes/elements.py:119` — replace with `json.loads()`
2. **Restrict CORS** — set `allow_origins` to frontend URL only
3. **Add authentication middleware** — at minimum API key check

---

*Concerns audit: 2026-03-26*
