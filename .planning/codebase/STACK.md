# Technology Stack

**Analysis Date:** 2026-03-26

## Languages

**Primary:**
- Python 3.10+ — Backend server, AI pipelines, SAM 3 microservice
- TypeScript ~5.9 — Frontend web application

**Secondary:**
- Shell (Bash) — Launch scripts (`start.sh`, `run_backend.sh`, `run_frontend.sh`, `run_sam.sh`)
- YAML — CI workflows (`sam3/.github/workflows/format.yml`)

## Runtime

**Environment:**
- Python 3.10+ (primary backend runtime)
- Node.js 18+ (frontend dev server)

**Package Managers:**
- **pip** — Backend dependencies via `requirements.txt` and local package installs
- **npm** — Frontend dependencies via `package.json` + `package-lock.json`
- **uv** — LTX-2 subpackages (`LTX-2/packages/ltx-core`, `ltx-pipelines`, `ltx-trainer`) use `uv` as their package manager with `pyproject.toml`
- **conda** — SAM 3 runs in a separate conda environment (`sam3_env`) via `/opt/miniconda3/bin/activate`

**Virtual Environments:**
- `milimov/` — Python venv at project root for main backend (LTX-2 + Flux 2 + FastAPI)
- `sam3_env` — Conda env for SAM 3 microservice (separate from main backend)

## Frameworks

**Core Backend:**
- FastAPI — REST API server (`backend/server.py`)
- Uvicorn — ASGI server (port 8000)
- SQLModel — ORM (SQLAlchemy-based) for data models (`backend/database.py`)
- SQLite — Database storage (file: `milimovideo.db`)
- sse-starlette — Server-Sent Events streaming for real-time progress

**Core Frontend:**
- React 19.2 — UI framework
- Vite 7.2 — Build tool and dev server
- Tailwind CSS 4.1 — Utility-first CSS (via `@tailwindcss/vite` plugin)
- Zustand 5.0 — State management (7-slice store architecture)
- zundo 2.3 — Undo/redo for Zustand
- Framer Motion 12.29 — Animations
- Lucide React 0.563 — Icon library

**AI/ML Frameworks:**
- PyTorch 2.10 — Core ML runtime (torch, torchvision, torchaudio)
- Transformers (HuggingFace) — Text encoder (Gemma 3, Qwen 3)
- Accelerate — Mixed precision and distributed training
- diffusers — VAE fallback for Flux 2
- timm — Image model backbone (SAM 3)
- huggingface_hub — Model downloading

**SAM 3 Microservice:**
- FastAPI + Uvicorn (port 8001)
- Runs separately from main backend

## Build Tools

**Frontend:**
- Vite 7.2 — Dev server, HMR, production builds
- `@vitejs/plugin-react` — React JSX transform
- TypeScript compiler (`tsc -b`) — Type checking before build
- PostCSS + Autoprefixer — CSS post-processing
- ESLint 9 — Linting (`eslint.config.js`, flat config)
  - `eslint-plugin-react-hooks`
  - `eslint-plugin-react-refresh`
  - `typescript-eslint`

**Backend:**
- setuptools — Build backend for sam3, flux2 packages
- uv_build — Build backend for ltx-core, ltx-pipelines
- hatchling — Build backend for ltx-trainer

**Linting (Python):**
- ruff — Linting and formatting (LTX-2, Flux 2)
- black — Code formatting (SAM 3)
- ufmt + usort — Import sorting (SAM 3)
- mypy — Type checking (SAM 3)

## Key Dependencies

**Critical Backend:**
- `torch==2.10.0` — Neural network inference
- `ltx-core` (local) — LTX-2 model core (transformer, VAE, text encoder)
- `ltx-pipelines` (local) — LTX-2 inference pipelines (ti2vid, ic_lora, keyframe)
- `flux2` (local) — Flux 2 image generation and inpainting
- `sam3` (local, separate env) — Segment Anything Model 3

**Critical Frontend:**
- `wavesurfer.js 7.12` — Audio waveform visualization
- `clsx 2.1` + `tailwind-merge 3.4` — Conditional CSS class merging
- `uuid 13.0` — ID generation

**Text Encoders / LLMs:**
- Gemma 3 12B (via ltx-core) — Video prompt enhancement (built-in)
- Ollama (optional external) — Alternative prompt enhancement via local LLM API
- Qwen 3 8B — Flux 2 text encoder
- OpenRouter API (optional) — Flux 2 prompt upsampling via Pixtral Large

## Configuration

**Environment Variables:**
- `MILIMO_LLM_PROVIDER` — LLM provider selection: `"gemma"` (default) or `"ollama"`
- `MILIMO_OLLAMA_URL` — Ollama API base URL (default: `http://localhost:11434`)
- `MILIMO_OLLAMA_MODEL` — Ollama model name (default: `llama3.1`)
- `MILIMO_OLLAMA_KEEP_ALIVE` — Ollama model keep-alive (default: `"0"`)
- `PYTORCH_ENABLE_MPS_FALLBACK` — Apple Silicon fallback for unsupported MPS ops
- `AE_MODEL_PATH` — VAE model path (set by `start.sh`)

**Build Config:**
- `web-app/vite.config.ts` — Vite plugins: react, tailwindcss
- `web-app/tsconfig.app.json` — TypeScript: ES2022 target, strict mode, bundler resolution
- `web-app/eslint.config.js` — ESLint flat config with TypeScript + React hooks

## Platform Requirements

**Development:**
- Python 3.10+
- Node.js 18+
- FFmpeg (for video processing, thumbnails, frame extraction)
- Apple Silicon (M1/M2/M3/M4) or NVIDIA GPU

**Production:**
- **Apple Silicon**: M1/M2/M3/M4 Max/Ultra, 32GB+ RAM recommended
- **NVIDIA**: 16GB+ VRAM recommended, CUDA 12.x
- MPS-first optimization: FP8 on CUDA, float32 fallback on MPS
- VAE decode CPU-offloaded on MPS to prevent black output

**External Dependencies:**
- FFmpeg — Video concatenation, thumbnail extraction, overlap trimming
- No Docker/containers detected
- No CI/CD pipeline for main project (SAM 3 has a basic `format.yml` GitHub Actions workflow)

---

*Stack analysis: 2026-03-26*
