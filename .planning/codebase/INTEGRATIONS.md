# External Integrations

**Analysis Date:** 2026-03-26

## APIs & External Services

**Ollama (Optional LLM Provider):**
- Purpose: Prompt enhancement for video and image generation
- SDK/Client: `requests` (direct HTTP calls)
- Auth: None (local service)
- Config: `MILIMO_OLLAMA_URL` env var (default: `http://localhost:11434`)
- Implementation: `backend/llm.py` → `enhance_prompt_ollama()`
- API: POST to `/api/generate` with JSON payload
- Model management: `MILIMO_OLLAMA_KEEP_ALIVE` controls VRAM release after use

**OpenRouter API (Flux 2 Prompt Upsampling):**
- Purpose: Image prompt upsampling for Flux 2 generation
- SDK/Client: `openai` Python SDK (OpenAI-compatible)
- Auth: OPENROUTER_API_KEY env var
- Implementation: `flux2/src/flux2/openrouter_api_client.py` → `OpenRouterAPIClient`
- Model: `mistralai/pixtral-large-2411` (default)
- Status: Optional, used when Flux 2 prompt upsampling is enabled

**HuggingFace Hub:**
- Purpose: Model checkpoint downloading (SAM 3, LTX-2, Flux 2)
- SDK/Client: `huggingface_hub`
- Implementation: `backend/services/model_downloader.py`
- Model manifest: `backend/models_manifest.json` (759 lines, 10+ models cataloged)
- Key repos:
  - `Lightricks/LTX-2` — LTX-2 checkpoints and LoRAs
  - `facebook/sam3` — SAM 3 segmentation model
  - `google/gemma-3-12b-it-qat-q4_0-unquantized` — Gemma 3 text encoder
- Progress streaming: tqdm_class override → SSE broadcast to frontend

## Data Storage

**Databases:**
- SQLite (file-based)
  - Connection: `sqlite:///backend/milimovideo.db`
  - Client: SQLModel + SQLAlchemy
  - Pool config: `pool_size=20, max_overflow=40`
  - Schema: `backend/database.py`
  - Tables: Project, Scene, Shot, Element, Asset, Job

**File Storage:**
- Local filesystem only
- Projects: `backend/projects/` (primary asset storage)
- Uploads: `backend/uploads/` (legacy)
- Generated: `backend/generated/` (legacy)
- Models: `backend/models/` (AI model checkpoints)
- LTX-2 models: `LTX-2/models/checkpoints/`, `LTX-2/models/upscalers/`, `LTX-2/models/text_encoders/`

**Caching:**
- None (no Redis, Memcached, or disk cache detected)

## Authentication & Identity

**Auth Provider:**
- None — Application runs local-first with no authentication
- CORS configured as wide-open: `allow_origins=["*"]`

## Communication Patterns

**Frontend ↔ Backend:**
- REST API: `http://localhost:8000` (FastAPI)
- SSE (Server-Sent Events): `GET /events` via `EventSource` for real-time updates
  - Events: job progress, generation status, model download progress
  - Backpressure: Max queue size 100, message TTL 30 seconds
  - Implementation: `backend/events.py` → `EventManager`, `web-app/src/providers/SSEProvider.tsx`
- Job polling fallback: `GET /status/{job_id}` for state recovery after page refresh

**Backend ↔ SAM 3 Microservice:**
- HTTP REST to `http://localhost:8001`
- Endpoints:
  - `POST /predict/mask` — Point-based mask prediction
  - `POST /detect` — Text-prompted object detection
  - `POST /segment/text` — Text-prompted segmentation (merged mask PNG)
  - `POST /track/start` — Start video tracking session
  - `POST /track/prompt` — Add tracking prompt
  - `POST /track/propagate` — Propagate tracking through video frames
  - `POST /track/remove_object` — Remove tracked object
  - `POST /track/stop` — Close tracking session
  - `GET /health` — Health check

**Backend API Routes:**
- `backend/routes/projects.py` — Project CRUD
- `backend/routes/shots.py` — Shot management and generation
- `backend/routes/jobs.py` — Job status and management
- `backend/routes/assets.py` — Asset upload/management
- `backend/routes/storyboard.py` — Script parsing and storyboard
- `backend/routes/elements.py` — Character/location/object elements
- `backend/routes/models.py` — Model registry and download management

## Media Processing

**FFmpeg:**
- Purpose: Video encoding, thumbnail extraction, frame extraction, concat demuxer
- Invocation: `subprocess` calls from `backend/tasks/video.py`
- Operations: overlap trimming, segment concatenation, thumbnail generation

**PyTorch Operations:**
- VAE encode/decode (video latents)
- Diffusion inference (LTX-2 transformer, Flux 2 flow model)
- Memory management: `gc.collect()` + `torch.mps.empty_cache()` / `torch.cuda.empty_cache()`
- Device auto-detection: CUDA → MPS → CPU

## Monitoring & Observability

**Error Tracking:**
- None (no Sentry, Datadog, or similar)

**Logs:**
- Backend: Python `logging` with two handlers
  - File: `server.log` — Rotating (10MB max, 3 backups), JSON-structured
  - Console: Human-readable format
- SAM 3: Basic `logging.basicConfig(level=logging.INFO)`
- LLM: Standard Python logger in `backend/llm.py`

**Metrics:**
- None (no Prometheus, StatsD, or similar)

## CI/CD & Deployment

**Hosting:**
- Local-first — No cloud hosting detected
- Runs via shell scripts: `start.sh`, `run_backend.sh`, `run_frontend.sh`, `run_sam.sh`

**CI Pipeline:**
- SAM 3 only: `.github/workflows/format.yml`
  - Trigger: Pull requests to `main`
  - Action: `ufmt` formatting check (black + usort + ruff-api)
  - No automated tests, no build, no deployment
- Main project: No CI/CD detected

**Deployment:**
- Manual process — no Docker, no Kubernetes, no cloud deployment configs detected
- No `.env` files detected (env vars set inline in shell scripts)

## Environment Configuration

**Required Configuration:**
- Model checkpoints manually downloaded to `backend/models/` and `LTX-2/models/`
- Python venv `milimov/` created and populated via pip
- SAM 3 conda env `sam3_env` created separately
- Node.js dependencies installed via `npm install` in `web-app/`

**Optional Configuration:**
- `MILIMO_LLM_PROVIDER` — Default `"gemma"`, alternative `"ollama"`
- `MILIMO_OLLAMA_URL` — Default `"http://localhost:11434"`
- `MILIMO_OLLAMA_MODEL` — Default `"llama3.1"`
- `MILIMO_OLLAMA_KEEP_ALIVE` — Default `"0"` (unload immediately)

## Secrets Location

- No `.env` files detected
- No `credentials.*` or `secrets.*` files detected
- Ollama is local (no auth needed)
- OpenRouter API key expected as env var (not found configured)
- HuggingFace models: some gated (may require HF token)

## Webhooks & Callbacks

**Incoming:**
- None detected

**Outgoing:**
- SSE broadcasts from backend to all connected frontend clients
- HTTP calls to SAM 3 microservice endpoints
- HTTP calls to Ollama API (when configured)
- HTTP calls to OpenRouter API (when Flux 2 upsampling enabled)

---

*Integration audit: 2026-03-26*
