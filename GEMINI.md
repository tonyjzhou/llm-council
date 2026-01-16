# LLM Council Project Context

## Project Overview
LLM Council is a local web application that orchestrates a "council" of multiple LLMs to deliberate on user queries. It implements a unique 3-stage process to produce high-quality responses:
1.  **Stage 1 (First Opinions):** The query is sent to multiple models (e.g., GPT-5, Claude, Gemini) in parallel.
2.  **Stage 2 (Peer Review):** Models anonymously review and rank each other's responses to prevent bias.
3.  **Stage 3 (Synthesis):** A designated "Chairman" model synthesizes a final answer based on the initial responses and the peer rankings.

The project is designed for "vibe coding" and experimentation with model collaboration.

## Tech Stack
*   **Backend:** Python 3.10+, FastAPI, `uv` for package management. Uses `httpx` for async OpenRouter API calls.
*   **Frontend:** React 19, Vite, `react-markdown`.
*   **API:** OpenRouter (requires `OPENROUTER_API_KEY`).
*   **Storage:** Local JSON files in `data/conversations/`.

## Setup & Running

### Prerequisites
*   **Python:** >= 3.10
*   **Node.js:** For frontend
*   **API Key:** An OpenRouter API key in `.env` (`OPENROUTER_API_KEY=...`)

### Installation
1.  **Backend:** `uv sync`
2.  **Frontend:** `cd frontend && npm install`

### Execution
*   **Recommended:** Run `./start.sh` to launch both backend (port 8001) and frontend (port 5173).
*   **Manual Backend:** `uv run python -m backend.main` (Must be run from project root).
*   **Manual Frontend:** `cd frontend && npm run dev`

## Architecture & Key Files

### Backend (`backend/`)
*   `main.py`: FastAPI entry point, CORS config, and SSE streaming endpoints.
*   `council.py`: Core logic for the 3-stage orchestration.
    *   `stage1_collect_responses()`
    *   `stage2_collect_rankings()`: Handles anonymization.
    *   `stage3_synthesize_final()`
*   `config.py`: Configuration for `COUNCIL_MODELS` and `CHAIRMAN_MODEL`.
*   `openrouter.py`: Async client for OpenRouter interactions.
*   `storage.py`: Manages JSON persistence of conversations.

### Frontend (`frontend/src/`)
*   `App.jsx`: Main state management and SSE stream handling.
*   `api.js`: Handles API requests and parses the SSE stream (`data: {JSON}`).
*   `components/`:
    *   `Stage1.jsx`: Tabbed view of individual model responses.
    *   `Stage2.jsx`: Displays peer rankings and "Street Cred" scores.
    *   `Stage3.jsx`: Shows the final synthesized answer.

## Key Implementation Details

### Data Flow
`User Query -> Stage 1 (Parallel) -> Stage 2 (Anonymized Peer Review) -> Stage 3 (Synthesis) -> SSE Stream -> Frontend`

### Anonymization (Stage 2)
To ensure fair reviews, the backend maps models to anonymous labels (e.g., "Response A") before sending them for review. The frontend receives the mapping to reveal identities in the UI after the fact.

### Streaming (SSE)
The backend uses Server-Sent Events to stream progress updates and results to the frontend.
**Event Types:** `stage1_start`, `stage1_complete`, `stage2_start`, `stage2_complete`, `stage3_start`, `stage3_complete`, `title_complete`, `complete`, `error`.

## Development Guidelines
*   **Backend Execution:** Always run the backend as a module from the root: `python -m backend.main`. **DO NOT** run from within the `backend/` directory to avoid relative import errors.
*   **CORS:** If changing ports, update CORS origins in `backend/main.py`.
*   **Configuration:** Model selection is hardcoded in `backend/config.py`.
*   **Error Handling:** The system is designed to degrade gracefully; if one model fails, others proceed. The request only fails if *all* models fail.
