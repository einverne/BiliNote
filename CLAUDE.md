# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

BiliNote is an AI-powered video note-taking assistant that automatically generates structured Markdown notes from video platforms (Bilibili, YouTube, Douyin, local files). The system consists of a FastAPI backend for video processing and AI transcription, and a React/TypeScript frontend with Tauri support for desktop applications.

## Development Commands

### Backend (FastAPI - Python)

```bash
# Navigate to backend
cd backend

# Install dependencies
pip install -r requirements.txt

# Run development server (default: http://0.0.0.0:8483)
python main.py
```

### Frontend (React + TypeScript + Vite)

```bash
# Navigate to frontend
cd BillNote_frontend

# Install dependencies
pnpm install

# Run development server (default: http://localhost:3015)
pnpm dev

# Build for production
pnpm build

# Lint code
pnpm lint

# Preview production build
pnpm preview
```

### Docker Deployment

```bash
# Standard deployment
docker-compose up -d

# GPU-accelerated deployment (for faster audio transcription)
docker-compose -f docker-compose.gpu.yml up -d
```

### Environment Configuration

Copy `.env.example` to `.env` and configure:
- `BACKEND_PORT`: Backend server port (default: 8483)
- `VITE_API_BASE_URL`: Frontend API connection URL
- `TRANSCRIBER_TYPE`: Audio transcription engine (fast-whisper/bcut/kuaishou/mlx-whisper/groq)
- `WHISPER_MODEL_SIZE`: Whisper model size (base/small/medium/large)

## Architecture Overview

### Backend Architecture (`backend/`)

**Core Processing Pipeline** (orchestrated by `NoteGenerator` in `app/services/note.py`):
1. **Download**: Platform-specific downloaders fetch video/audio
2. **Transcribe**: Audio-to-text conversion using Whisper or alternative engines
3. **Generate**: LLM providers generate structured notes
4. **Enhance**: Optional screenshot insertion and timestamp linking

**Key Components**:

- **Downloaders** (`app/downloaders/`): Platform-specific video/audio fetchers
  - `BilibiliDownloader`: Bilibili platform
  - `YoutubeDownloader`: YouTube (via yt-dlp)
  - `DouyinDownloader`: Douyin/TikTok support
  - `LocalDownloader`: Local video file processing
  - Base class: `Downloader` with standard interface `download(url, quality) -> AudioDownloadResult`

- **Transcribers** (`app/transcriber/`): Audio-to-text conversion
  - `WhisperTranscriber`: Local faster-whisper implementation
  - `GroqTranscriber`: Groq API-based transcription
  - `BcutTranscriber`, `KuaishouTranscriber`: Alternative services
  - `MLXWhisperTranscriber`: Apple Silicon optimized (M1/M2/M3)
  - Base class: `Transcriber` with `transcribe(audio_path) -> TranscriptResult`
  - Factory pattern via `get_transcriber(transcriber_type)` with singleton caching

- **GPT Integration** (`app/gpt/`): LLM provider abstraction
  - `GPTFactory`: Creates provider instances via factory pattern
  - `OpenAI_compatible_provider`: Base provider supporting OpenAI-compatible APIs
  - Platform-specific implementations: `deepseek_gpt.py`, `qwen_gpt.py`, `universal_gpt.py`
  - `prompt_builder.py`: Constructs structured prompts with format/style/extras parameters
  - All providers extend `GPT` base class with `generate(sources, format, style) -> str` interface

- **Database** (`app/db/`): SQLite-based persistence
  - `models/`: SQLAlchemy ORM models (`video_tasks.py`, `providers.py`)
  - `*_dao.py`: Data access objects for CRUD operations
  - `video_task_dao`: Manages task lifecycle (insert, update status, retrieve by video_id)
  - `provider_dao`: Manages LLM provider configurations with default seeding

- **API Routes** (`app/routers/`):
  - `note.py`: Core video processing endpoints (`/process`, `/upload`, `/proxy`)
  - `provider.py`: LLM provider management (CRUD for model configurations)
  - `model.py`: Model selection and configuration
  - `config.py`: System configuration endpoints

**Note Generation Flow**:
```
VideoRequest → NoteGenerator.generate()
  ├─ download_audio() → Downloader → AudioDownloadResult
  ├─ transcribe() → Transcriber → TranscriptResult (segments with timestamps)
  ├─ generate_screenshots() → Optional[List[str]] (video frame captures)
  ├─ GPTFactory.create() → GPT instance
  ├─ gpt.generate() → Markdown content
  ├─ replace_content_markers() → Insert screenshots/links
  └─ save_note_to_file() → NoteResult with status tracking
```

### Frontend Architecture (`BillNote_frontend/src/`)

**State Management** (Zustand stores in `store/`):
- `taskStore`: Video processing task lifecycle and polling
- `providerStore`: LLM provider configurations
- `modelStore`: Model selection state
- `configStore`: Application settings

**Key Components**:
- `pages/HomePage/`: Main video processing interface with form and task history
- `pages/SettingPage/`: Configuration pages (models, transcribers, prompts)
- `components/Form/`: Dynamic form components for provider/model configuration
- `services/`: API client layer with Axios HTTP requests
- `hooks/useTaskPolling`: Background polling for task status updates (3s interval)

**Routing Structure**:
```
/ (Index)
├─ / → HomePage (video URL input and processing)
├─ /settings → SettingPage
│  ├─ /settings/model → Model management
│  │  ├─ /settings/model/new → Create provider
│  │  └─ /settings/model/:id → Edit provider
│  ├─ /settings/download → Downloader configuration
│  └─ /settings/about → About page
```

**API Integration Pattern**:
- Proxy via Vite dev server: `/api` → backend, `/static` → backend static files
- Request service (`utils/request.ts`) handles auth, errors, and response wrapping
- Backend initialization check via `useCheckBackend` hook before rendering app

## Important Technical Details

### FFmpeg Dependency
The project requires FFmpeg for audio processing. Ensure it's installed and in PATH:
```bash
# macOS
brew install ffmpeg

# Ubuntu/Debian
sudo apt install ffmpeg
```

### Transcriber System
- Singleton pattern: `get_transcriber()` caches instances in `_transcribers` dict
- Type-based selection via `TranscriberType` enum
- Configuration through `TRANSCRIBER_TYPE` environment variable
- Each transcriber returns standardized `TranscriptResult` with timestamped segments

### Provider System
- Database-driven: LLM configurations stored in SQLite
- Default providers seeded on startup via `seed_default_providers()`
- Frontend manages provider CRUD through `/api/provider` endpoints
- Factory pattern creates provider instances at runtime based on provider_id

### Task Lifecycle
- Tasks stored in SQLite with status enum: `pending`, `processing`, `completed`, `failed`
- Frontend polls `/api/status/{task_id}` every 3 seconds
- Background processing via FastAPI `BackgroundTasks`
- Results cached in `note_results/` directory as JSON files

### Video Understanding Feature
- Optional multimodal analysis: extract frames at intervals → send to vision-capable LLMs
- Controlled by `video_understanding` flag and `video_interval` parameter
- Frame grid assembly via `grid_size` parameter for batch vision processing

## Development Notes

- Frontend uses Tailwind CSS v4 with shadcn/ui component library
- Backend uses FastAPI async patterns for non-blocking I/O
- Video downloads handled by `yt-dlp` for YouTube, custom scrapers for other platforms
- Screenshot generation via `generate_screenshot()` with OpenCV/PIL
- Markdown export with optional timestamp links (`[HH:MM:SS](video_url?t=seconds)`)
