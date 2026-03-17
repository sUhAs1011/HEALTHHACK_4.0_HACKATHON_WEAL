# HEALTHHACK_4.0_HACKATHON_WEAL

This file is the implementation-aware guide for AI coding assistants working on the **Kalpana** repository. Read fully before changing architecture or feature behavior.

---

## 1. Project Overview & Philosophy

**Kalpana** is an AI-driven mental health peer-support platform with two parallel goals:
1. Deliver immediate, empathetic support through a local Listener agent.
2. Analyze user distress in the background through a local Mapper agent and route safely.

### Core Principles
- **Safety First:** Routing and crisis behavior are deterministic Python logic, not LLM-only decisions.
- **Decoupled Agents:** Empathy generation and clinical mapping are separate concurrent pipelines.
- **Privacy Aware by Path:** Core Listener/Mapper reasoning runs locally via Ollama.  
  Voice mode currently uses Sarvam cloud APIs (STT/translation/TTS), so audio/text in that path is not fully local.

---

## 2. System Architecture

The backend processes each turn with concurrent components and deterministic session state.

### A. Listener Agent (`backend/agents/listener.py`)
- Model: `ministral-3:3b` (local via Ollama)
- Role: empathetic user-facing response generation
- Output: streamed text chunks over SSE (`type: "chunk"`)
- Prompt steering via phases: `greeting`, `explore`, `probe`, `process`, `crisis`

### B. Clinical Mapper Agent (`backend/agents/mapper.py`)
- Model: `gemma3:4b` (local via Ollama)
- Role: structured distress profiling in strict JSON format
- Fields:
  - `clinical_summary`
  - `primary_emotion`
  - `detected_risk`
  - `risk_score` (1-10)
  - `self_harm_indicators` (bool)
  - `root_cause_of_the_distress`
- Defensive parsing fallback is required due small-model JSON fragility.

### C. Controller / Routing (`backend/api.py`)
Main endpoint: `POST /api/chat` (SSE)

State rules:
1. Root-cause lock once non-`-` value appears.
2. Session risk memory via `session_risk_score`.
3. Crisis intercept when `self_harm_indicators == True` OR `risk_score >= 8`.
4. Crisis intercept bypasses Pinecone matching.
5. Match gate when all are true:
   - `session_root_cause != "-"`,
   - current `risk_score >= 5`,
   - `history_len >= 4`,
   - not in crisis intercept.
6. Metadata SSE packet always emitted as:
   - `{"type":"metadata","peer_group_match":...,"crisis_intercept":...}`

### D. Matchmaker (`backend/utils/matchmaker.py`)
- Embedding model: `bert-base-nli-mean-tokens` (local)
- Vector DB: Pinecone
- Similarity threshold: `0.70`
- On success, attaches `peer_id` and score metadata.

### E. Voice Pipeline (Sarvam) (`backend/api.py` + `backend/utils/sarvam_api.py`)
- `POST /api/transcribe`
  - Audio upload -> Sarvam STT (`saaras:v3`, `mode=translate`, auto language detect)
  - Returns English transcript (`transcript_en`) and detected language.
  - Updates session `preferred_voice_language` when confidence threshold is met.
- `POST /api/tts`
  - Takes final assistant text.
  - Translates from English to target language via Sarvam Translate.
  - Synthesizes speech via Sarvam TTS.
  - Returns `audio_base64` for frontend playback.
- Frontend applies reply mode policy:
  - `auto`: TTS only for voice user turns
  - `text_only`: no TTS
  - `voice_preferred`: TTS for all turns

---

## 3. Technology Stack

- **Frontend:** React + Vite + Tailwind v4 (`kalpana-frontend/`)
- **Backend:** FastAPI (`backend/api.py`)
- **Streaming:** SSE via FastAPI `StreamingResponse`
- **Concurrency:** `concurrent.futures.ThreadPoolExecutor`
- **Local LLMs:** Ollama (`ministral-3:3b`, `gemma3:4b`)
- **Vector Search:** Pinecone + sentence-transformers
- **Voice APIs:** Sarvam (`/speech-to-text`, `/translate`, `/text-to-speech`)
- **Persistence:**
  - `session_logs/session_logN.json`
  - `data/peers.json`
  - `data/appointments.json`
- **Upload support:** `python-multipart`

---

## 4. Repository Structure Highlights

- `backend/api.py`
  - primary controller
  - `/api/chat`, `/api/transcribe`, `/api/tts`, `/api/schedule`
- `backend/agents/listener.py`
  - phase-conditioned empathetic generation
- `backend/agents/mapper.py`
  - strict clinical JSON mapping with safe fallback
- `backend/utils/matchmaker.py`
  - Pinecone peer search
- `backend/utils/sarvam_api.py`
  - STT/Translate/TTS wrappers + retries/timeouts
- `kalpana-frontend/src/App.jsx`
  - chat orchestration, reply mode, voice processing states, SSE handling
- `kalpana-frontend/src/components/ChatInput.jsx`
  - text + voice capture (`audioBlob` passed upward)
- `kalpana-frontend/src/components/PeerMatchModal.jsx`
  - scheduling UI with custom dropdown
- `kalpana-frontend/src/components/CrisisModal.jsx`
  - crisis overlay (currently dummy action buttons)
- `app0.py`, `app1.py`
  - legacy Streamlit variants kept for reference only

---

## 5. Coding Standards & Guardrails

1. **No session mutation in agents:** `listener.py` and `mapper.py` are I/O only.
2. **Mapper defensiveness mandatory:** always tolerate malformed JSON output.
3. **Mapper context bound:** user-only recent messages (no full conversation dump).
4. **Core chat model policy:** no OpenAI/Anthropic for Listener/Mapper.
5. **SSE metadata contract is strict:** keep `type`, `peer_group_match`, `crisis_intercept`.
6. **Crisis has UI priority:** crisis modal must override peer match modal.
7. **Custom dropdown requirement:** avoid native `<select>` in peer scheduling UI.
8. **Voice fallback requirement:** STT/translation/TTS failures must degrade to text, never break chat.
9. **Pre-seed availability:** `peer_match["availability"] = []` before lookup.

---

## 6. API Endpoints (`backend/api.py`)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/chat` | Main SSE chat endpoint. Streams Listener chunks then emits metadata packet. |
| `POST` | `/api/transcribe` | Voice STT endpoint. Accepts multipart audio and returns `transcript_en` + language signals. |
| `POST` | `/api/tts` | Assistant TTS endpoint. Translates then synthesizes speech, returns `audio_base64`. |
| `POST` | `/api/schedule` | Saves appointment into `data/appointments.json`. |

---

## 7. Current Development Status (as of 2026-03-15)

| Feature | Status |
|---------|--------|
| FastAPI SSE backend | Implemented |
| React frontend | Implemented |
| Dual-agent concurrency | Implemented |
| Pinecone matchmaking (0.70 threshold) | Implemented |
| Peer scheduling modal | Implemented |
| Crisis intercept (`crisis_intercept`) + CrisisModal | Implemented (UI content currently placeholder/demo) |
| Voice STT -> chat -> translate -> TTS loop | Implemented (active) |
| Reply mode (`Auto`, `Text Only`, `Voice Preferred`) | Implemented |
| PII scrubbing in active runtime path | Not implemented |
| Realtime WebSocket chatrooms | Planned |

---

## 8. Roadmap Priorities

- Stabilize production-grade crisis content/resources (current modal actions are placeholder).
- Harden voice reliability and observability (better error surfacing, speaker/language diagnostics).
- Add realtime peer chatrooms with secure transitions after match/scheduling.
- Add passive moderation layer for live peer rooms.
- Add optional PII scrubbing pipeline once latency/perf budget allows.
