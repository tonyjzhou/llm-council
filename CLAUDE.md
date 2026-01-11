# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Development Commands

```bash
# Install all dependencies
uv sync                          # Backend (Python)
cd frontend && npm install       # Frontend (React)

# Run development servers
./start.sh                       # Both servers (recommended)
# OR manually:
uv run python -m backend.main    # Backend on port 8001
cd frontend && npm run dev       # Frontend on port 5173

# Frontend only
cd frontend
npm run dev      # Dev server with HMR
npm run build    # Production build
npm run lint     # ESLint
npm run preview  # Preview production build
```

**Critical**: Backend MUST be run as `python -m backend.main` from project root (not from backend/ directory) due to relative imports.

## Project Overview

LLM Council is a 3-stage deliberation system where multiple LLMs collaboratively answer user questions via OpenRouter:

1. **Stage 1**: Parallel queries to all council models → individual responses
2. **Stage 2**: Anonymized peer review (models see "Response A, B, C" without knowing sources) → rankings
3. **Stage 3**: Chairman model synthesizes final answer from all responses + rankings

Key innovation: Anonymous labels in Stage 2 prevent models from playing favorites.

## Architecture

```
backend/
├── main.py        # FastAPI app, routes, SSE streaming
├── config.py      # COUNCIL_MODELS, CHAIRMAN_MODEL, API key
├── council.py     # Core 3-stage orchestration logic
├── openrouter.py  # Async OpenRouter API client
└── storage.py     # JSON conversation persistence

frontend/src/
├── App.jsx        # State management, streaming handlers
├── api.js         # HTTP/SSE client for backend
└── components/
    ├── Stage1.jsx # Tab view of individual model responses
    ├── Stage2.jsx # Peer rankings + aggregate "Street Cred" scores
    └── Stage3.jsx # Final synthesized answer
```

### Data Flow

```
User Query → Stage 1 (parallel) → Stage 2 (anonymize + parallel rank)
    → Aggregate Rankings → Stage 3 (chairman synthesis) → SSE stream to frontend
```

### Key Files

- **`council.py`**: Core logic - `stage1_collect_responses()`, `stage2_collect_rankings()`, `stage3_synthesize_final()`, `parse_ranking_from_text()`
- **`config.py`**: Model configuration - edit `COUNCIL_MODELS` list and `CHAIRMAN_MODEL`
- **`api.js`**: SSE stream handling with `sendMessageStream()` parsing `data: {JSON}` lines

## Configuration

**Environment** (`.env` in project root):
```
OPENROUTER_API_KEY=sk-or-v1-...
```

**Models** (`backend/config.py`):
```python
COUNCIL_MODELS = ["openai/gpt-5.1", "google/gemini-3-pro-preview", ...]
CHAIRMAN_MODEL = "google/gemini-3-pro-preview"
```

**Ports**: Backend 8001, Frontend 5173 (update `main.py` CORS + `api.js` if changing)

## Important Implementation Details

### Stage 2 Anonymization
- Backend creates `label_to_model` mapping: `{"Response A": "openai/gpt-5.1", ...}`
- Models receive only anonymous labels during evaluation
- Frontend de-anonymizes for display (models in **bold**)
- Metadata (label_to_model, aggregate_rankings) NOT persisted to JSON, only returned via API

### Ranking Parsing
Stage 2 prompt enforces strict format:
```
FINAL RANKING:
1. Response C
2. Response A
...
```
Fallback regex extracts any "Response X" patterns if models deviate.

### Error Handling
- Graceful degradation: if one model fails, others continue
- Only fails entire request if ALL models fail
- `query_models_parallel()` returns None for failed models, filters them out

### Frontend Patterns
- All ReactMarkdown content wrapped in `<div className="markdown-content">` (12px padding)
- SSE events: `stage1_start/complete`, `stage2_start/complete`, `stage3_start/complete`, `title_complete`, `complete`, `error`
- Optimistic UI: user message added immediately, assistant message updated progressively

## Common Gotchas

1. **Import errors**: Run `python -m backend.main` from project root, never `cd backend && python main.py`
2. **CORS errors**: Frontend origin must match allowed origins in `main.py` (localhost:5173, localhost:3000)
3. **Missing metadata on reload**: `label_to_model` and `aggregate_rankings` are ephemeral (API response only)
4. **Parse failures**: Check `parsed_ranking` field - fallback regex may produce incomplete results

## Storage

JSON files in `data/conversations/`. Each conversation contains:
```json
{
  "id": "uuid", "created_at": "...", "title": "...",
  "messages": [
    {"role": "user", "content": "..."},
    {"role": "assistant", "stage1": [...], "stage2": [...], "stage3": {...}}
  ]
}
```
