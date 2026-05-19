# TruthGuard

Multi-modal AI system for detecting fake news, manipulated media, and disinformation.
Accepts images, videos, PDFs, and URLs — runs a committee of specialized AI agents in parallel and returns a verdict.

---

## What it does

| Input | Agents activated |
|-------|-----------------|
| Image / screenshot | Image forensics · Linguistic · Context |
| Video clip | Video forensics (deepfake) · Audio transcript |
| News article / PDF | Claim extraction · Claim verification (RAG) · Source credibility |
| URL / social post | Source credibility · Network propagation · Linguistic |

**Output:** risk score (0–1), confidence, verdict (`allow` / `flag-review` / `block`), and a human-readable audit trail.

---

## Architecture

```
Browser (Laravel :8080)
        │  POST /analyse (multipart)
        ▼
orchestrator_api.py (:8001)        ← main entry point
        │
        ├── Preprocessing/ (:8000) ← OCR, ASR, EXIF, layout parsing
        │
        ├── Agents/
        │   ├── claim_extractor.py
        │   ├── claim_verifier.py  ← RAG + FAISS
        │   ├── agent_image_forensics.py
        │   ├── agent_video_forensics.py
        │   ├── linguistic_agent.py
        │   ├── context_agent.py
        │   ├── source_cred_agent.py
        │   └── network_agent.py
        │
        └── Evidence fusion → verdict
```

---

## Stack

| Layer | Technology |
|-------|-----------|
| Frontend | HTML/JS served via Laravel Blade |
| Web framework | Laravel 10 (PHP 8.2) |
| API / Agents | Python 3.11 · FastAPI · LangChain |
| LLM | OpenAI GPT-4o |
| Vector DB | FAISS + ChromaDB |
| Embeddings | sentence-transformers `all-MiniLM-L6-v2` |
| Re-ranker | cross-encoder `ms-marco-MiniLM-L-6-v2` |
| Image analysis | OpenCV · ManTraNet · ExifTool |
| Audio | Whisper (ASR) |

---

## Setup

### 1. Clone and install Python dependencies

```bash
git clone https://github.com/your-org/codecrafters.git
cd codecrafters
pip install -r requirements.txt
```

### 2. Configure environment

```bash
cp .env.example .env
# Add your OpenAI key:
# OPENAI_API_KEY=sk-...
```

### 3. Install Laravel dependencies

```bash
composer install
php artisan key:generate
```

### 4. Populate the trusted corpus (for RAG claim verification)

```bash
python scripts/ingest_corpus.py --source data/corpus.jsonl
```

Corpus JSONL format:
```json
{"text": "...", "source": "Reuters", "url": "https://...", "published_date": "2024-03-15"}
```

---

## Running

Open **3 terminals**:

```bash
# Terminal 1 — Laravel frontend
php artisan serve --port=8080

# Terminal 2 — Preprocessing service
uvicorn Preprocessing.main:app --port 8000

# Terminal 3 — Orchestrator (main API)
python orchestrator_api.py
```

Then open: **http://127.0.0.1:8080/truthguard**

---

## API

### `POST /analyse`

Multipart form fields:

| Field | Type | Description |
|-------|------|-------------|
| `file` | binary | Image, video, PDF, or text file |
| `input_type` | string | `image` `video` `document` `url` |
| `content_nature` | string | `social_post_screenshot` `news_article` `scientific_claim` ... |
| `analysis_goals` | string[] | `authenticity` `claim_verification` `source_credibility` ... |
| `source_url` | string | Optional — original URL of the content |
| `post_text` | string | Optional — typed text from the post |
| `platform` | string | Optional — `Twitter` `Facebook` `Telegram` ... |

**Response:**
```json
{
  "verdict": "Likely Manipulated or Misleading",
  "risk_score": 0.60,
  "confidence": 0.91,
  "action": "flag-review",
  "processing_time_ms": 2938,
  "agents": {
    "image_forensics": { "risk_score": 0.73, "anomalies": ["..."] },
    "claim_verify":    { "verified_count": 0, "total": 1 },
    "linguistic":      { "clickbait_score": 0.58, "ai_gen_prob": 0.57 }
  },
  "audit_trail": { ... }
}
```

### Other endpoints

| Endpoint | Description |
|----------|-------------|
| `POST /analyse/quick` | File only — auto-routing, no user dialogue |
| `GET /health` | Service health check |
| `GET /agents` | List all available agents |
| `GET /docs` | Swagger UI |

---

## Port map

| Service | Port | File |
|---------|------|------|
| Laravel (frontend) | 8080 | `routes/web.php` |
| Orchestrator API | 8001 | `orchestrator_api.py` |
| Preprocessing | 8000 | `Preprocessing/main.py` |

---

## Content types supported

```
social_post_screenshot  news_article        scientific_claim
government_document     video_clip          audio_clip
advertisement           meme                chat_screenshot
url_link                raw_image           unknown
```

## Analysis goals

```
authenticity            claim_verification      source_credibility
contextual_consistency  network_analysis        linguistic_analysis
deepfake_detection      metadata_forensics
```

---

## Known issues

- Claim extraction falls back to placeholder text when no `post_text` is provided and the real `ClaimExtractor` agent is not yet wired — fix in progress
- Video forensics and network agent are currently simulated — real models pending integration

---

## Project structure

```
codecrafters-main/
├── Agents/                  ← AI agent modules
├── Preprocessing/           ← Input normalization service
├── Router/                  ← Input type detection (3-layer)
├── chroma_db/               ← ChromaDB vector store
├── resources/views/
│   └── truthguard.blade.php ← Frontend UI
├── routes/web.php           ← Laravel routes
├── orchestrator_api.py      ← Main API entry point
├── requirements.txt
└── .env.example
```
