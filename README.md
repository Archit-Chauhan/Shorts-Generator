# 🎬 AI YouTube Shorts Generator

> Turn any long-form YouTube video into 3–5 polished, story-complete YouTube Shorts — guided by **real audience behavior**, not arbitrary clipping.

[![Python](https://img.shields.io/badge/python-3.12%2B-blue)]()
[![FastAPI](https://img.shields.io/badge/backend-FastAPI-009688)]()
[![License](https://img.shields.io/badge/license-MIT-green)]()
[![Status](https://img.shields.io/badge/status-MVP-yellow)]()

---

## Table of Contents

- [Overview](#overview)
- [Why This Is Different](#why-this-is-different)
- [MVP Scope](#mvp-scope)
- [Pipeline](#pipeline)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Processing Details](#processing-details)
  - [Transcript Processing](#transcript-processing)
  - [Comment Processing](#comment-processing)
  - [Timestamp Extraction](#timestamp-extraction)
  - [Timestamp Clustering](#timestamp-clustering)
  - [Candidate Ranking](#candidate-ranking)
  - [LLM Context Analysis](#llm-context-analysis)
  - [Human Emotional Judgment Layer](#human-emotional-judgment-layer)
  - [Clip Boundary Detection](#clip-boundary-detection)
  - [Video Editing & Vertical Conversion](#video-editing--vertical-conversion)
  - [Subtitles, Hooks, Captions & Quotes](#subtitles-hooks-captions--quotes)
  - [Export Format](#export-format)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [Environment Variables](#environment-variables)
- [Usage](#usage)
- [Coding Standards](#coding-standards)
- [Error Handling](#error-handling)
- [Performance](#performance)
- [Future Roadmap](#future-roadmap)
- [Success Criteria](#success-criteria)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

The AI YouTube Shorts Generator automatically produces high-quality vertical Shorts from long-form YouTube videos.

Unlike existing AI clipping tools, this project does **not** simply summarize a video or cut random sections. Instead, it uses **human audience behavior** — comments, timestamp mentions, engagement patterns — as the primary signal for what actually resonated with viewers. An LLM then reconstructs the full context around those moments and edits them into complete, shareable stories.

The system is designed to imitate the decision-making process of a human video editor: find where people reacted, understand *why*, and cut a clip that feels complete — not truncated.

## Why This Is Different

| Typical AI Clippers | This Project |
|---|---|
| Summarize transcript | Analyzes real viewer reactions |
| Clip fixed-length windows | Detects natural story boundaries |
| No reasoning about content | LLM judges setup, payoff, emotion, shareability |
| Black-box output | Confidence score + explicit reason per clip |

## MVP Scope

**Input:** A single YouTube URL

**Output:** 3–5 polished Shorts, each including:

- MP4 video (9:16, vertical)
- Burned-in subtitles
- AI-generated title
- Hook
- Caption
- Hashtags
- Quote overlay
- Confidence score
- Reason the clip was selected

**Explicitly out of scope for MVP:** automatic publishing, channel monitoring, and scheduling (see [Future Roadmap](#future-roadmap)).

## Pipeline

```text
YouTube URL
      │
      ▼
Download Video
      │
      ▼
Extract Transcript
      │
      ▼
Download Comments
      │
      ▼
Extract Timestamp Comments
      │
      ▼
Cluster Nearby Timestamps
      │
      ▼
Rank Candidate Moments
      │
      ▼
LLM Context Analysis
      │
      ▼
Determine Story Boundaries
      │
      ▼
Cut Video
      │
      ▼
Convert to Vertical
      │
      ▼
Generate Subtitles
      │
      ▼
Generate Hook
      │
      ▼
Generate Caption
      │
      ▼
Generate Quote
      │
      ▼
Export Short
```

## Architecture

The project is fully modular. Every stage is an independent service, and no module depends on another module's internal implementation — communication happens through interfaces / service abstractions. This makes it possible to swap any component (e.g. the LLM provider, the transcription engine, or the vertical-conversion strategy) without touching the rest of the system.

**Core modules:**

- `Downloader` — fetches source video via `yt-dlp`
- `TranscriptService` — gets transcript (native or transcribed)
- `CommentService` — retrieves comments via YouTube Data API
- `TimestampExtractor` — parses timestamp mentions from comments
- `TimestampClustering` — groups nearby timestamps into candidate moments
- `CandidateRanking` — scores clusters by engagement signals
- `LLMService` — provider-agnostic LLM abstraction
- `EmotionAnalyzer` — scores sentiment/emotion of comments and clips
- `StoryBoundaryDetector` — finds narrative start/end of a clip
- `ClipGenerator` — frame-accurate video cutting
- `SubtitleGenerator` — generates and burns subtitles
- `HookGenerator` — generates and ranks multiple hooks
- `CaptionGenerator` — generates captions, hashtags, descriptions
- `QuoteGenerator` — generates a shareable quote
- `ExportService` — packages final MP4 + subtitles + metadata

## Tech Stack

| Concern | Choice |
|---|---|
| Language | Python 3.12+ |
| Backend framework | FastAPI |
| Database | SQLite (MVP) → PostgreSQL (later, no code changes to business logic) |
| Video processing | FFmpeg |
| Speech recognition | Faster-Whisper |
| Video download | yt-dlp |
| Comment retrieval | YouTube Data API |
| LLM | Provider-agnostic abstraction — Ollama, OpenAI, Gemini, or local models |

> **LLM provider rule:** No API-specific logic is ever hardcoded outside the provider abstraction layer. Swapping providers should require zero changes to any downstream module.

## Processing Details

### Transcript Processing

Two methods, in priority order:

1. **Primary:** existing YouTube transcript
2. **Fallback:** Faster-Whisper transcription

Output format:

```json
[
  {
    "start": 0.0,
    "end": 4.2,
    "text": "..."
  }
]
```

### Comment Processing

Collected comment sets:

- Top comments
- Latest comments

Stored fields per comment:

- `text`
- `likes`
- `reply_count`
- `author`
- `published_at`

### Timestamp Extraction

Detects timestamp mentions such as:

```
0:45
12:31
1:03:18
```

Requirements:

- Support multiple timestamps per comment
- Ignore invalid/out-of-range timestamps
- Configurable regex pattern

### Timestamp Clustering

Nearby timestamps are merged into a single cluster.

- Cluster window is configurable
- Default: **±10 seconds**

Output format:

```json
[
  {
    "start": 750,
    "end": 770,
    "mentions": 42
  }
]
```

### Candidate Ranking

A configurable scoring system ranks clusters using:

- Timestamp mention frequency
- Likes
- Reply count
- Sentiment
- Duplicate mentions
- Emoji density

Score range: **0–100**

### LLM Context Analysis

For each cluster, the transcript window around the peak is extracted, e.g.:

```
Peak: 12:35
Transcript window: 12:00 → 13:20
```

The LLM is prompted to determine:

- Where the story starts
- Where it ends
- Emotional payoff
- Summary
- Confidence
- Emotion category

Response is returned as structured JSON.

### Human Emotional Judgment Layer

This is the core differentiator of the project. Rather than summarizing, the LLM behaves like a professional editor, evaluating:

- Setup strength
- Payoff strength
- Emotional intensity
- Curiosity
- Relatability
- Replay value
- Quote quality
- Shareability

Example output:

```json
{
  "setup": 9,
  "payoff": 10,
  "emotion": "Humor",
  "shareability": 9,
  "quote_worthy": true,
  "reason": "Strong payoff after believable setup."
}
```

### Clip Boundary Detection

Clips are never cut at just the mentioned timestamp. The LLM determines the full narrative arc:

```
Story begins → Conversation develops → Peak → Natural ending
```

The resulting clip should always feel like a complete story.

### Video Editing & Vertical Conversion

**Video editing requirements:**

- Frame-accurate trimming
- Quality preservation
- Avoid unnecessary re-encoding where possible

**Vertical conversion (9:16):**

- Initial strategy: center crop
- Future: speaker tracking, face tracking, dynamic crop

### Subtitles, Hooks, Captions & Quotes

**Subtitles**
- Accurate timestamps
- Multiline, properly wrapped
- *Future:* karaoke-style, animated word-by-word

**Hooks**
- At least 5 generated per clip
- Styles: curiosity, funny, motivational, dramatic, storytelling
- Ranked by predicted CTR

**Captions**
- Caption text, hashtags, and description
- Must avoid misleading claims

**Quote**
- One short, emotional, shareable quote per clip

### Export Format

Each exported Short includes:

- `.mp4` video file
- Subtitle file
- Metadata JSON

Example metadata:

```json
{
  "title": "...",
  "hook": "...",
  "caption": "...",
  "hashtags": [],
  "confidence": 95,
  "emotion": "Funny",
  "reason": "High audience engagement."
}
```

## Project Structure

```text
ai-yt-shorts-generator/
├── app/
│   ├── api/                  # FastAPI routes
│   ├── core/                 # Config, logging, DI container
│   ├── services/
│   │   ├── downloader/
│   │   ├── transcript/
│   │   ├── comments/
│   │   ├── timestamps/
│   │   ├── ranking/
│   │   ├── llm/
│   │   │   ├── providers/    # openai.py, gemini.py, ollama.py, local.py
│   │   │   └── base.py       # provider interface
│   │   ├── emotion/
│   │   ├── boundaries/
│   │   ├── video/
│   │   ├── subtitles/
│   │   ├── hooks/
│   │   ├── captions/
│   │   ├── quotes/
│   │   └── export/
│   ├── models/                # DB models / schemas
│   ├── repositories/          # DB access layer
│   └── utils/
├── prompts/                   # Versioned LLM prompt templates
├── tests/
├── samples/                   # Sample outputs
├── .env.example
├── requirements.txt
├── README.md
└── docs/
    ├── API.md
    └── INSTALL.md
```

## Installation

```bash
# 1. Clone the repository
git clone https://github.com/<your-org>/ai-yt-shorts-generator.git
cd ai-yt-shorts-generator

# 2. Create a virtual environment
python3.12 -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Install FFmpeg (system dependency)
# macOS
brew install ffmpeg
# Ubuntu/Debian
sudo apt install ffmpeg

# 5. Copy environment template and fill in your keys
cp .env.example .env

# 6. Run database migrations (SQLite by default)
alembic upgrade head

# 7. Start the API server
uvicorn app.main:app --reload
```

## Environment Variables

Create a `.env` file based on `.env.example`:

```env
# --- General ---
ENVIRONMENT=development
LOG_LEVEL=INFO

# --- Database ---
DATABASE_URL=sqlite:///./app.db

# --- YouTube Data API ---
YOUTUBE_API_KEY=your_youtube_api_key

# --- LLM Provider (choose one) ---
LLM_PROVIDER=openai        # options: openai | gemini | ollama | local
OPENAI_API_KEY=
GEMINI_API_KEY=
OLLAMA_BASE_URL=http://localhost:11434
LOCAL_MODEL_PATH=

# --- Whisper / Transcription ---
WHISPER_MODEL_SIZE=medium
WHISPER_DEVICE=cpu         # cpu | cuda

# --- Clustering / Ranking Config ---
TIMESTAMP_CLUSTER_WINDOW_SECONDS=10
CANDIDATE_MIN_SCORE=50

# --- Export ---
OUTPUT_DIR=./output
```

## Usage

```bash
# Example: kick off a generation job via the API
curl -X POST http://localhost:8000/api/v1/shorts/generate \
  -H "Content-Type: application/json" \
  -d '{"youtube_url": "https://www.youtube.com/watch?v=VIDEO_ID"}'
```

The endpoint returns a job ID. Poll the status endpoint (or use the progress-reporting mechanism described in [Performance](#performance)) to track pipeline progress and retrieve the final Shorts once complete.

Full endpoint reference lives in [`docs/API.md`](docs/API.md).

## Coding Standards

- Type hints on all functions and public interfaces
- Docstrings for all modules, classes, and public functions
- Structured logging throughout the pipeline
- Configuration via environment/config files — no hardcoded values
- Dependency injection where practical
- Small, single-responsibility, modular services
- Reusable shared utilities
- Clean, consistent folder structure
- No monolithic scripts — every pipeline stage is its own service

## Error Handling

The system gracefully handles:

- Invalid or unsupported YouTube URLs
- Unavailable/disabled transcripts
- YouTube/LLM API rate limits
- Video download failures
- FFmpeg processing failures
- Videos with no comments
- Videos with no timestamp-referencing comments

All failures surface meaningful, actionable error messages rather than raw stack traces.

## Performance

- Caching of expensive intermediate results (transcripts, comments, LLM responses)
- Resumable processing — a failed run can continue from the last completed stage
- Asynchronous task execution where appropriate (FastAPI background tasks / task queue)
- Progress reporting throughout the pipeline

## Future Roadmap

The architecture is designed so the following can be added without major refactoring:

- Automatic YouTube channel monitoring
- New upload detection
- Job queue system
- Scheduled processing
- AI-generated thumbnails
- Sound effects
- Background music recommendation
- B-roll suggestions
- Speaker identification
- Analytics dashboard
- Cloud deployment
- Multi-user authentication

> These are **not implemented in the MVP** — they are architectural design targets only.

## Success Criteria

The MVP is considered successful when a user can:

1. Paste a YouTube URL
2. Click **Generate**
3. Receive 3–5 coherent Shorts
4. Each Short includes:
   - A complete story
   - Subtitles
   - A generated hook
   - A caption
   - Hashtags
   - A confidence score
   - An explanation of why the clip was selected

The guiding principle throughout is **human-guided AI editing**: audience reactions identify candidate moments, and AI transforms those moments into complete, engaging short-form videos — never simply clipping isolated timestamps.

## Contributing

Contributions are welcome. Please:

1. Open an issue describing the change before large PRs
2. Follow the [Coding Standards](#coding-standards)
3. Add/update unit tests for any core logic changes
4. Keep modules decoupled — depend on interfaces, not implementations

## License

[MIT](LICENSE)
