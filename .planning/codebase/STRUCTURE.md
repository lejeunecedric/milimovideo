# Codebase Structure

**Analysis Date:** 2026-03-26

## Directory Layout

```
milimovideo/
├── backend/                    # FastAPI Python backend (port 8000)
│   ├── server.py               # FastAPI app entry point, CORS, static serving, SSE
│   ├── config.py               # Centralized paths, ports, defaults, presets
│   ├── database.py             # SQLModel ORM (Project, Scene, Shot, Element, Asset, Job)
│   ├── schemas.py              # Pydantic request/response schemas
│   ├── events.py               # SSE broadcast manager with backpressure
│   ├── model_engine.py         # LTX-2 pipeline manager (singleton)
│   ├── memory_manager.py       # GPU memory mutual exclusion (LTX vs Flux)
│   ├── job_utils.py            # GPU semaphore, job tracking, progress updates
│   ├── file_utils.py           # Path resolution, project workspace paths
│   ├── llm.py                  # Prompt enhancement (Gemma / Ollama dispatcher)
│   ├── worker.py               # Task re-export hub (not standalone in production)
│   ├── requirements.txt        # Python dependencies
│   ├── models_manifest.json    # Model registry manifest
│   ├── routes/                 # API route handlers
│   │   ├── __init__.py         # Unified router aggregation
│   │   ├── projects.py         # CRUD, save/load, render, split shot, LLM settings
│   │   ├── shots.py            # Partial shot update (PATCH)
│   │   ├── storyboard.py       # Script parse, commit, AI parse, batch generate, reorder
│   │   ├── elements.py         # Element CRUD, visualize, inpaint, SAM health, tracking
│   │   ├── jobs.py             # Job status, cancel, GPU status, generate endpoints
│   │   ├── assets.py           # Asset upload/management
│   │   └── models.py           # Model registry, download, delete, LoRA management
│   ├── tasks/                  # Background GPU generation tasks
│   │   ├── __init__.py
│   │   ├── video.py            # Video generation (single-shot + chained dispatch)
│   │   ├── image.py            # Image generation via Flux 2
│   │   └── chained.py          # Autoregressive multi-chunk video generation
│   ├── services/               # Business logic services
│   │   ├── __init__.py
│   │   ├── ai_storyboard.py    # Gemma 3 powered script-to-storyboard parsing
│   │   ├── element_matcher.py  # @Trigger word matching to project elements
│   │   ├── script_parser.py    # Regex-based screenplay parser (fallback)
│   │   ├── model_registry.py   # Model manifest + disk scan status
│   │   ├── model_downloader.py # HuggingFace model download with progress
│   │   ├── model_loader.py     # Model loading helpers
│   │   ├── model_search.py     # Model search/filtering
│   │   └── model_settings.py   # Model configuration management
│   ├── managers/               # Domain state managers
│   │   ├── element_manager.py  # Element CRUD, visual generation, prompt injection
│   │   ├── inpainting_manager.py # Flux RePaint inpainting orchestration
│   │   └── tracking_manager.py # SAM 3 video tracking session coordination
│   ├── models/                 # AI model wrappers
│   │   ├── __init__.py
│   │   └── flux_wrapper.py     # FluxInpainter, FluxAEWrapper, IP-Adapter projector
│   ├── storyboard/             # Storyboard generation engine
│   │   ├── __init__.py
│   │   └── manager.py          # StoryboardManager: chunking, overlap, shot prep
│   ├── utils/                  # Shared utilities
│   │   └── render_utils.py     # FFmpeg complex filtergraph builder for NLE render
│   └── migrations/             # Database migration scripts
│       └── migrate_to_project_workspaces.py
├── web-app/                    # React + TypeScript frontend (Vite, port 5173)
│   ├── package.json            # Dependencies: React 19, Zustand, Tailwind CSS 4, framer-motion
│   ├── vite.config.ts          # Vite config (React + Tailwind plugins)
│   ├── tsconfig.json           # TypeScript configuration
│   ├── index.html              # SPA entry HTML
│   ├── public/                 # Static assets (logo, favicons)
│   └── src/
│       ├── main.tsx            # App mount: ErrorBoundary → SSEProvider → App
│       ├── App.tsx             # Root: Layout + CinematicPlayer + Toast container
│       ├── config.ts           # API_BASE_URL (localhost:8000), getAssetUrl helper
│       ├── components/         # React components
│       │   ├── Layout.tsx      # Main layout shell
│       │   ├── Player/         # Video playback
│       │   │   ├── CinematicPlayer.tsx
│       │   │   └── PlaybackEngine.tsx
│       │   ├── Timeline/       # NLE timeline tracks
│       │   │   ├── VisualTimeline.tsx
│       │   │   ├── TimelineTrack.tsx
│       │   │   ├── TimelineClip.tsx
│       │   │   ├── AudioClip.tsx
│       │   │   ├── Playhead.tsx
│       │   │   └── TimeDisplay.tsx
│       │   ├── Storyboard/     # Storyboard view
│       │   │   ├── StoryboardView.tsx
│       │   │   ├── StoryboardSceneGroup.tsx
│       │   │   ├── StoryboardShotCard.tsx
│       │   │   ├── ScriptInput.tsx
│       │   │   └── ElementBadge.tsx
│       │   ├── Inspector/      # Right panel editors
│       │   │   ├── InspectorPanel.tsx
│       │   │   ├── ShotParameters.tsx
│       │   │   ├── AdvancedSettings.tsx
│       │   │   ├── ConditioningEditor.tsx
│       │   │   └── NarrativeDirector.tsx
│       │   ├── Editor/         # Masking & tracking overlays
│       │   │   ├── MaskingCanvas.tsx
│       │   │   ├── SegmentationOverlay.tsx
│       │   │   └── TrackingPanel.tsx
│       │   ├── Library/        # Media & element management
│       │   │   ├── ElementManager.tsx
│       │   │   ├── ElementPanel.tsx
│       │   │   └── MediaLibrary.tsx
│       │   ├── Elements/       # Elements view
│       │   │   └── ElementsView.tsx
│       │   ├── Images/         # Image gallery view
│       │   │   └── ImagesView.tsx
│       │   ├── ModelLibrary/   # Model/LoRA management
│       │   │   ├── ModelLibrary.tsx
│       │   │   ├── ModelSettings.tsx
│       │   │   ├── LoRAManager.tsx
│       │   │   └── index.ts
│       │   ├── Controls.tsx    # Playback controls
│       │   ├── ExportModal.tsx  # Export/render dialog
│       │   ├── MediaUploader.tsx
│       │   ├── MediaLibrary.tsx
│       │   ├── ProjectManager.tsx
│       │   ├── LLMSettings.tsx
│       │   ├── VideoPlayer.tsx
│       │   ├── Toggle.tsx
│       │   └── ErrorBoundary.tsx
│       ├── stores/             # Zustand state management
│       │   ├── timelineStore.ts # Composite store (8 slices + persist + zundo)
│       │   ├── types.ts        # TypeScript type definitions
│       │   └── slices/         # Store slices
│       │       ├── projectSlice.ts
│       │       ├── shotSlice.ts
│       │       ├── playbackSlice.ts
│       │       ├── uiSlice.ts
│       │       ├── trackSlice.ts
│       │       ├── elementSlice.ts
│       │       ├── serverSlice.ts
│       │       └── modelSlice.ts
│       ├── hooks/              # Custom React hooks
│       │   ├── useEventSource.ts
│       │   └── useSSE.ts
│       ├── providers/          # Context providers
│       │   └── SSEProvider.tsx
│       ├── contexts/           # React contexts
│       │   └── SSEContext.ts
│       └── utils/              # Shared utilities
│           ├── GlobalAudioManager.ts
│           ├── jobPoller.ts
│           ├── snapEngine.ts
│           └── timelineUtils.ts
├── LTX-2/                      # LTX-Video 2 — video generation model (forked)
│   ├── pyproject.toml          # uv workspace config
│   ├── packages/
│   │   ├── ltx-core/           # Core model, VAE, text encoders, components
│   │   │   └── src/ltx_core/
│   │   │       ├── components/ # Diffusion steps, guiders, noisers, schedulers
│   │   │       ├── conditioning/
│   │   │       ├── guidance/
│   │   │       ├── loader/     # Model/LoRA loading
│   │   │       ├── model/      # Video VAE, audio VAE, upsampler
│   │   │       ├── text_encoders/ # Gemma text encoder
│   │   │       ├── tools.py
│   │   │       ├── types.py
│   │   │       └── utils.py
│   │   ├── ltx-pipelines/      # Generation pipelines
│   │   │   └── src/ltx_pipelines/
│   │   │       ├── ti2vid_two_stages.py  # Main pipeline (two-stage upscaling)
│   │   │       ├── ti2vid_one_stage.py   # Single-stage alternative
│   │   │       ├── ic_lora.py            # IC-LoRA conditioning pipeline
│   │   │       ├── keyframe_interpolation.py
│   │   │       ├── distilled.py
│   │   │       └── utils/      # Media I/O, helpers, constants
│   │   └── ltx-trainer/        # Training utilities (not used at runtime)
│   └── models/                 # Model weights (not in git)
│       ├── checkpoints/        # ltx-2-19b-distilled.safetensors, distilled-lora-384
│       ├── upscalers/          # spatial-upscaler-x2, temporal-upscaler-x2
│       └── text_encoders/gemma3/
├── flux2/                      # Flux 2 Klein — image generation model (forked)
│   ├── pyproject.toml
│   └── src/flux2/
│       ├── model.py            # Flux flow transformer (9B params)
│       ├── autoencoder.py      # Native AE encoder/decoder
│       ├── sampling.py         # Denoising sampler
│       ├── text_encoder.py     # Qwen 3 text encoder
│       ├── openrouter_api_client.py # OpenRouter API for prompt enhancement
│       ├── system_messages.py  # System prompts
│       ├── util.py
│       └── watermark.py
├── sam3/                       # SAM 3 — segmentation & tracking model
│   ├── pyproject.toml
│   ├── start_sam_server.py     # FastAPI microservice entry point (port 8001)
│   ├── sam3/
│   │   ├── model/              # SAM 3 model, video predictor, processor
│   │   ├── agent/
│   │   ├── eval/
│   │   └── model_builder.py    # Model construction
│   └── scripts/                # Utility scripts
├── milimov/                    # Python virtual environment (venv)
│   ├── bin/python              # Python interpreter for backend
│   └── lib/                    # Installed packages
├── assets/                     # Static images for README/docs
│   ├── logo_milimo.png
│   ├── milimo_video_timeline.png
│   ├── milimo_video_image_generation.png
│   ├── milimo_video_elements.png
│   ├── Milimo_Video_Studio_Architecture_infograph.png
│   └── favicon_io/
├── docs/                       # Technical documentation
│   ├── 01_system_architecture.md
│   ├── 02_data_models.md
│   ├── 03_ai_pipelines.md
│   ├── 04_frontend_state.md
│   ├── 05_execution_flow.md
│   ├── 06_file_dependency.md
│   ├── CODEBASE_ANALYSIS.md
│   ├── flux2-bible.md
│   ├── ltx2-bible.md
│   └── sam3-bible.md
├── skills/                     # OpenCode skill definitions
│   ├── skills/                 # 21 skill modules including milimo-specific ones
│   ├── spec/
│   └── template/
├── start.sh                    # Launch backend + frontend
├── run_backend.sh              # Launch backend only (port 8000)
├── run_frontend.sh             # Launch frontend only (port 5173)
├── run_sam.sh                  # Launch SAM 3 service (port 8001)
└── README.md                   # Project overview and setup guide
```

## Directory Purposes

**`backend/`** — FastAPI application server. All business logic, AI orchestration, database access, and API endpoints.

**`web-app/`** — React SPA with NLE-style UI. State managed via Zustand with 8 slices. Tailwind CSS 4 for styling.

**`LTX-2/`** — Forked Lightricks LTX-Video 2 model. Three sub-packages: `ltx-core` (model/VAE), `ltx-pipelines` (generation), `ltx-trainer` (training). Installed as editable pip packages into `milimov/` venv.

**`flux2/`** — Forked Flux 2 Klein 9B image model. Handles image generation, inpainting, IP-Adapter conditioning. Installed as editable pip package.

**`sam3/`** — SAM 3 segmentation model. Runs as separate microservice on port 8001 (separate conda env `sam3_env`). Provides text-prompted segmentation, click-to-segment, and video object tracking.

**`milimov/`** — Python venv for backend + LTX-2 + Flux 2. Created with `python3 -m venv`. SAM 3 uses a separate conda env.

**`assets/`** — Static image assets for documentation and README. Not served by the application.

**`docs/`** — Comprehensive technical documentation covering architecture, data models, AI pipelines, frontend state, execution flow, and per-model integration guides.

**`skills/`** — OpenCode skill definitions including milimo-specific skills (`milimo-ai-pipeline-expert`, `milimo-frontend-state-manager`, `milimo-storyboard-analyst`, `milimo-system-architect`).

## Key File Locations

**Entry Points:**
- `backend/server.py` — Backend API server entry point
- `web-app/src/main.tsx` — Frontend SPA entry point
- `sam3/start_sam_server.py` — SAM 3 microservice entry point
- `start.sh` — Orchestrator: launches backend + frontend

**Configuration:**
- `backend/config.py` — All paths, ports, defaults, presets, LLM settings
- `web-app/src/config.ts` — Frontend API base URL and asset URL helper
- `web-app/vite.config.ts` — Vite build configuration

**Core Logic:**
- `backend/model_engine.py` — LTX-2 ModelManager (singleton)
- `backend/models/flux_wrapper.py` — Flux 2 FluxInpainter wrapper
- `backend/memory_manager.py` — GPU memory mutual exclusion
- `backend/storyboard/manager.py` — Autoregressive chunk generation
- `backend/tasks/video.py` — Main video generation task (808 lines)
- `backend/tasks/chained.py` — Multi-chunk chained generation (498 lines)

**State Management:**
- `web-app/src/stores/timelineStore.ts` — Composite Zustand store
- `web-app/src/stores/slices/` — 8 domain slices

**Testing:**
- Not detected — no test files or test configuration found

## Naming Conventions

**Files:**
- Backend Python: `snake_case.py` (e.g., `model_engine.py`, `job_utils.py`)
- Frontend TypeScript: `camelCase.ts` for utilities, `PascalCase.tsx` for components
- Routes match domain: `projects.py`, `shots.py`, `storyboard.py`, `elements.py`

**Directories:**
- Backend: lowercase plural for domain grouping (`routes/`, `services/`, `managers/`, `tasks/`, `models/`)
- Frontend: PascalCase for component directories (`Timeline/`, `Storyboard/`, `Player/`, `Inspector/`)
- Store slices: `camelCaseSlice.ts` pattern

## Where to Add New Code

**New API Endpoint:**
- Route handler: `backend/routes/{domain}.py`
- Schema: `backend/schemas.py`
- Business logic: `backend/services/` or `backend/managers/`

**New Background Task:**
- Task function: `backend/tasks/{type}.py`
- Register in: `backend/worker.py` (re-export)
- Queue via: `job_utils.queue_video_task` or `BackgroundTasks.add_task`

**New Frontend Component:**
- Component: `web-app/src/components/{Category}/{Name}.tsx`
- Store slice: `web-app/src/stores/slices/{domain}Slice.ts`
- Hook: `web-app/src/hooks/use{Name}.ts`

**New AI Model Integration:**
- Wrapper: `backend/models/{model}_wrapper.py`
- Pipeline loading: `backend/model_engine.py`
- Memory slot: `backend/memory_manager.py` (add conflict rules)

**New Service:**
- Implementation: `backend/services/{name}.py`
- Wire into routes as needed

## Special Directories

**`milimov/`** — Python virtual environment. Generated by `python3 -m venv`. Committed? No (in `.gitignore`).

**`backend/models/`** — Model weight storage. Large files not in git. Contains `models_manifest.json` for registry.

**`backend/projects/`** — Per-project workspace directories. Created at runtime. Each has `generated/`, `thumbnails/`, `assets/`, `workspace/` subdirs.

**`skills/`** — OpenCode skill definitions. Committed. 21 skill modules including 4 milimo-specific skills.

**`docs/`** — Reference documentation. Committed. 10 markdown files covering all subsystems.

---

*Structure analysis: 2026-03-26*
