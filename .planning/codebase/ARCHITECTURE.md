# Architecture

**Analysis Date:** 2026-03-26

## Pattern Overview

**Overall:** Multi-process client-server with embedded AI model pipelines and a separate segmentation microservice.

**Key Characteristics:**
- **Monolithic FastAPI backend** with in-process background tasks (not a task queue like Celery)
- **SPA frontend** (React/Vite) communicating via REST API + SSE (Server-Sent Events)
- **Separate SAM 3 microservice** on port 8001 for segmentation/tracking isolation
- **Local-first execution** — all AI models run on the user's machine (Apple Silicon or CUDA GPU)
- **Project-scoped workspaces** — each project gets its own directory for generated assets

## Layers

**Presentation Layer (Frontend):**
- Purpose: NLE-style video editing UI with storyboard, timeline, and inspector panels
- Location: `web-app/src/`
- Contains: React components, Zustand stores, SSE providers, utility functions
- Depends on: Backend REST API (port 8000), SSE event stream
- Used by: Browser

**API Layer (Backend Routes):**
- Purpose: HTTP endpoints for CRUD operations, generation triggers, and model management
- Location: `backend/routes/`
- Contains: `projects.py`, `shots.py`, `storyboard.py`, `elements.py`, `jobs.py`, `assets.py`, `models.py`
- Depends on: `database.py`, `schemas.py`, `services/`, `managers/`
- Used by: Frontend, SAM 3 service (indirectly)

**Service Layer:**
- Purpose: Business logic for AI storyboard parsing, element matching, model registry, script parsing
- Location: `backend/services/`
- Contains: `ai_storyboard.py`, `element_matcher.py`, `script_parser.py`, `model_registry.py`, `model_downloader.py`, `model_loader.py`, `model_search.py`, `model_settings.py`
- Depends on: `model_engine.py`, `llm.py`, LTX-2 text encoders
- Used by: Routes, Managers

**Manager Layer:**
- Purpose: Domain-specific state management for elements, inpainting, and tracking
- Location: `backend/managers/`
- Contains: `element_manager.py`, `inpainting_manager.py`, `tracking_manager.py`
- Depends on: `database.py`, `models/flux_wrapper.py`, SAM 3 service
- Used by: Routes, Tasks

**Task/Worker Layer:**
- Purpose: Long-running GPU-bound generation tasks executed as FastAPI BackgroundTasks
- Location: `backend/tasks/`
- Contains: `video.py`, `image.py`, `chained.py`
- Depends on: `model_engine.py`, LTX-2 pipelines, Flux 2, `job_utils.py`, `events.py`
- Used by: Routes (`jobs.py`, `storyboard.py`)

**Model Engine Layer:**
- Purpose: AI model lifecycle management — loading, unloading, pipeline selection
- Location: `backend/model_engine.py` (LTX-2), `backend/models/flux_wrapper.py` (Flux 2)
- Contains: `ModelManager` singleton, `FluxInpainter`, `FluxAEWrapper`, IP-Adapter projector
- Depends on: LTX-2 packages (`ltx-core`, `ltx-pipelines`), Flux 2 source (`flux2/src/`)
- Used by: Tasks, Memory Manager

**Storyboard Engine:**
- Purpose: Autoregressive multi-chunk video generation with overlap handling
- Location: `backend/storyboard/manager.py`
- Contains: `StoryboardManager` class, chunk calculation, overlap frame extraction
- Depends on: `config.py`, LTX-2 pipeline, FFmpeg
- Used by: Tasks (`chained.py`), Routes (`storyboard.py`, `jobs.py`)

**Data Layer:**
- Purpose: Persistence via SQLModel (SQLAlchemy) with SQLite
- Location: `backend/database.py`
- Contains: `Project`, `Scene`, `Shot`, `Element`, `Asset`, `Job` models
- Depends on: SQLite file at `backend/milimovideo.db`
- Used by: All routes, managers, tasks, job_utils

**Cross-Cutting Services:**
- `backend/memory_manager.py` — Mutual exclusion between LTX-2 and Flux 2 to prevent OOM
- `backend/events.py` — SSE broadcast system with backpressure
- `backend/job_utils.py` — GPU semaphore (1 concurrent job), job tracking, progress updates
- `backend/llm.py` — Prompt enhancement dispatcher (Gemma or Ollama)
- `backend/config.py` — Centralized configuration, paths, defaults

## Data Flow

**Video Generation Flow (Text-to-Video):**

1. Frontend sends `POST /generate/advanced` with `ShotConfig` → `backend/routes/jobs.py`
2. Route creates `Job` record in SQLite, queues via `BackgroundTasks.add_task(queue_video_task, ...)`
3. `job_utils.queue_video_task` wraps task in `gpu_job_wrapper` (asyncio Semaphore(1) — one GPU job at a time)
4. `tasks/video.py::generate_video_task` loads LTX-2 pipeline via `ModelManager.load_pipeline()`
5. `MemoryManager.prepare_for("video")` unloads Flux 2 if loaded
6. Pipeline runs TI2VidTwoStagesPipeline → generates video frames → encodes to MP4
7. Progress broadcast via `events.py::event_manager.broadcast("progress", ...)` → SSE to frontend
8. On completion: updates `Job` status in DB, updates `Shot.video_url`, broadcasts `job_complete`

**Storyboard Flow (Script-to-Video):**

1. User pastes script → Frontend calls `POST /projects/{id}/storyboard/ai-parse`
2. `ai_storyboard.py` uses Gemma 3 text encoder to parse into scenes/shots
3. `element_matcher.py` matches `@Trigger` words to project Elements
4. User commits → `POST /projects/{id}/storyboard/commit` saves to DB
5. Per-shot generation: `POST /shots/{shot_id}/generate` → `StoryboardManager.prepare_shot_generation()`
6. For shots > 121 frames: `generate_chained_video_task` splits into chunks with 25-frame overlap
7. Each chunk conditions on last frame of previous chunk for continuity

**Image Generation Flow:**

1. Frontend calls `POST /generate/image` or `POST /elements/{id}/visualize`
2. `tasks/image.py::generate_image_task` loads Flux 2 pipeline via `FluxInpainter`
3. `MemoryManager.prepare_for("image")` unloads LTX-2 if loaded
4. Flux 2 generates image → saves to project workspace
5. IP-Adapter conditioning with element reference images

**Inpainting Flow:**

1. Frontend gets mask via SAM 3 (`POST http://localhost:8001/predict/mask` or `/segment/text`)
2. Frontend sends mask + prompt to `POST /elements/{id}/inpaint`
3. `inpainting_manager.py` loads Flux 2, runs RePaint iterative denoising
4. Masked regions blended back → result saved to project

**SAM 3 Tracking Flow:**

1. Frontend → `POST http://localhost:8001/track/start` with video path
2. `POST /track/prompt` at specific frame with text/points/box
3. `POST /track/propagate` → streams per-frame masks back
4. `backend/managers/tracking_manager.py` coordinates session lifecycle on backend side

**State Management (Frontend):**

1. Zustand store (`timelineStore.ts`) composed of 8 slices (project, shot, playback, ui, track, element, server, model)
2. `zundo` middleware provides undo/redo (20-step history)
3. `persist` middleware saves project state to localStorage
4. SSE via `SSEProvider` → `SSEContext` → store updates on events (progress, job_complete, etc.)

## Entry Points

**Backend Server:**
- Location: `backend/server.py`
- Trigger: `./run_backend.sh` → `./milimov/bin/python backend/server.py`
- Port: 8000
- Responsibilities: FastAPI app, CORS, static file serving (Range requests for Safari), SSE endpoint, router inclusion, startup/shutdown lifecycle

**SAM 3 Microservice:**
- Location: `sam3/start_sam_server.py`
- Trigger: `./run_sam.sh` → `conda activate sam3_env && python start_sam_server.py`
- Port: 8001
- Responsibilities: SAM 3 model loading, mask prediction, text segmentation, video tracking

**Frontend:**
- Location: `web-app/src/main.tsx`
- Trigger: `./run_frontend.sh` → `cd web-app && npm run dev`
- Port: 5173 (Vite default)
- Responsibilities: React app mount, ErrorBoundary, SSEProvider wrapping

**Worker Module:**
- Location: `backend/worker.py`
- Trigger: Imported by server (not run standalone in production)
- Responsibilities: Re-exports tasks, model manager, and utilities for centralized imports

## Error Handling

**Strategy:** Try/except at task level with DB status updates + SSE error broadcasts

**Patterns:**
- GPU tasks: Catch exceptions → update `Job.status = "failed"` in DB → broadcast `job_failed` via SSE
- Zombie job cleanup: On server startup, any jobs stuck in `"processing"` are marked `"failed"` with "Server Restarted"
- Job cancellation: `active_jobs[job_id]["cancelled"] = True` flag checked periodically in generation loops
- SAM 3 service: Returns HTTP 503 if model not loaded, 500 with error detail on inference failure

## Cross-Cutting Concerns

**Logging:** Custom `StructuredFormatter` for JSON file logging + human-readable console logging. Rotating file handler (10MB, 3 backups).
**Validation:** Pydantic `BaseModel` schemas in `backend/schemas.py` for all API request/response types
**Authentication:** None — local-first application with no user accounts
**Memory Management:** `MemoryManager` singleton enforces mutual exclusion between LTX-2 ("video") and Flux 2 ("image") slots. Prevents OOM on Apple Silicon unified memory.
**Concurrency:** `asyncio.Semaphore(1)` limits GPU to one job at a time. FIFO queue via semaphore wait.

---

*Architecture analysis: 2026-03-26*
