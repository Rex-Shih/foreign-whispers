# Foreign Whispers

Foreign Whispers is a YouTube video dubbing pipeline. It downloads a source
video, transcribes speech with Whisper, translates the transcript, synthesizes
target-language speech with Chatterbox TTS, and stitches the dubbed audio back
onto the video.

The easiest way to run the project is Docker Compose. The TA machine has a GPU,
so use the NVIDIA profile below.

## Requirements

- Docker and Docker Compose
- NVIDIA GPU
- NVIDIA Container Toolkit installed on the host
- Git

Optional:

- A Hugging Face token if you want to run diarization with pyannote.
- A `cookies.txt` file in the repo root if YouTube blocks downloads or requires
  authenticated cookies.

## Quick Start

From the repo root:

```bash
cp .env.example .env
```

Edit `.env` if needed:

```bash
# Use your host UID/GID so files created by Docker are not owned by root.
id -u
id -g

# Optional, only needed for diarization.
FW_HF_TOKEN=your_huggingface_token
```

Build and start the GPU stack:

```bash
docker compose --profile nvidia up --build
```

Then open the web app:

```text
http://localhost:8501
```

The backend health check is:

```text
http://localhost:8080/healthz
```

## What Starts

Docker Compose starts four services:

| Service | Port | Purpose |
| --- | --- | --- |
| `foreign-whispers-frontend` | `8501` | Next.js web UI |
| `foreign-whispers-api` | `8080` | FastAPI orchestration API |
| `foreign-whispers-stt` | `8000` | GPU Whisper speech-to-text server |
| `foreign-whispers-tts` | `8020` | GPU Chatterbox text-to-speech server |

The API container coordinates the pipeline and calls the GPU STT/TTS services
over HTTP.

## Running a Dubbing Job

1. Open `http://localhost:8501`.
2. Select or enter a YouTube video.
3. Run the pipeline stages from the UI:
   - Download
   - Transcribe
   - Translate
   - TTS
   - Stitch
4. The final dubbed video and captions are available in the UI after stitching.

Pipeline outputs are written under:

```text
pipeline_data/
```

Important output folders:

```text
pipeline_data/videos/              # Downloaded source videos
pipeline_data/youtube_captions/    # Captions from YouTube
pipeline_data/transcriptions/      # Whisper transcripts
pipeline_data/translations/        # Translated transcripts
pipeline_data/tts_audio/           # Generated TTS audio
pipeline_data/dubbed_captions/     # Generated WebVTT captions
pipeline_data/dubbed_videos/       # Final dubbed videos
```

## Useful Commands

Start the GPU stack:

```bash
docker compose --profile nvidia up --build
```

Start in the background:

```bash
docker compose --profile nvidia up --build -d
```

View logs:

```bash
docker compose --profile nvidia logs -f
```

View one service:

```bash
docker compose --profile nvidia logs -f api
docker compose --profile nvidia logs -f whisper-gpu
docker compose --profile nvidia logs -f chatterbox-gpu
docker compose --profile nvidia logs -f frontend
```

Stop everything:

```bash
docker compose --profile nvidia down
```

Rebuild after dependency or Dockerfile changes:

```bash
docker compose --profile nvidia build
docker compose --profile nvidia up
```

Run tests locally, outside Docker:

```bash
uv sync
uv run pytest
```

## API Endpoints

The UI is the intended entry point, but the main API endpoints are:

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/healthz` | Backend health check |
| `GET` | `/api/videos` | Video catalog |
| `POST` | `/api/download` | Download source video and captions |
| `POST` | `/api/transcribe/{video_id}` | Run transcription |
| `POST` | `/api/translate/{video_id}` | Run translation |
| `POST` | `/api/tts/{video_id}` | Generate dubbed speech |
| `POST` | `/api/stitch/{video_id}` | Create final dubbed video |
| `GET` | `/api/video/{video_id}` | Stream dubbed video |
| `GET` | `/api/video/{video_id}/original` | Stream original video |
| `GET` | `/api/captions/{video_id}` | Stream translated captions |
| `GET` | `/api/captions/{video_id}/original` | Stream original captions |

## Troubleshooting

If Docker cannot see the GPU, verify the NVIDIA runtime:

```bash
docker run --rm --gpus all nvidia/cuda:12.6.3-base-ubuntu22.04 nvidia-smi
```

If YouTube download fails, export browser cookies to a Netscape-format
`cookies.txt` file and place it in the repo root. The Docker Compose file mounts
that file into the API container automatically.

If files in `pipeline_data/` are owned by root, update `UID` and `GID` in `.env`
to match your host user, then fix existing ownership:

```bash
sudo chown -R $(id -u):$(id -g) pipeline_data/
```

If a model download is slow on the first run, wait for the STT and TTS containers
to finish downloading their model files. Later runs reuse Docker volumes for the
Whisper and Chatterbox model caches.
