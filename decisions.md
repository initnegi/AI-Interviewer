# Project Decisions

This document records the implementation choices visible in the repository and the changes made while documenting it. It is intended to make the current system understandable without reconstructing the commit history.

## Current implementation decisions

### 1. FastAPI owns the application boundary

The backend is a FastAPI application in `src/main.py`. It owns interview lifecycle routes, provider integrations, resume parsing, rate limiting, authentication lookup, and static-file serving. Keeping these concerns behind REST endpoints lets the React client remain focused on interview interaction and presentation.

### 2. The client is built separately and bundled into the backend image

The client uses React, TypeScript, Vite, Tailwind CSS, and Clerk. The Dockerfile uses a Node 22 build stage, runs `npm ci`, builds the client, removes the checked-in `public/` contents, and copies the Vite output into `public/`. This creates one deployable service with a same-origin production experience.

### 3. Interview behavior is prompt-driven and stage-aware

`src/interview_templates.py` centralizes mode plans, stage objectives, resume rules, and DSA-specific guidance. `stage_for_exchange` changes the prompt objective as the exchange count advances. The backend retains the system prompt and recent conversation messages, then uses `compact_messages` to limit prompt size while preserving the latest context.

### 4. Multiple provider fallbacks reduce single-provider failure risk

Text generation attempts Groq, then Google Gemini, then OpenRouter. Speech-to-text attempts Groq Whisper, then Deepgram. Text-to-speech attempts Groq Orpheus, then ElevenLabs, and finally asks the browser to use its native speech synthesis. Provider credentials are read from environment variables rather than source code.

### 5. Company-specific retrieval stays intentionally small

`src/rag.py` loads `data/Amazon.docx` and `data/Microsoft.docx` at import time, groups paragraphs into roughly 600-character chunks, and ranks chunks by shared query words. This avoids a vector database and keeps deployment simple for the two bundled corpora. It is lexical retrieval, not semantic embedding search.

### 6. Resumes are parsed server-side with explicit limits

The backend accepts PDF, DOCX, and TXT files, rejects files larger than 5 MB, normalizes extracted text, rejects empty content, and truncates the prompt context to 8,000 characters. The client performs matching file-type and size checks for immediate feedback, while the server remains the authoritative validator.

### 7. Feedback is structured JSON

`src/analytics.py` asks the LLM for a fixed schema containing a score, summary, strengths, weaknesses, improvement actions, skill scores, hiring verdict, and next steps. The response is stripped of optional markdown fences and parsed as JSON. Missing keys or invalid JSON produce an explicit error payload instead of silently displaying fabricated feedback.

### 8. Clerk and MongoDB are optional at process startup

Clerk JWT verification is enabled only when both issuer and JWKS settings exist. MongoDB initialization is also conditional on `MONGO_URI`; index creation failures are deferred rather than preventing the process from starting. Saved-session routes require authentication and a configured database, while the core interview flow remains available to the process.

### 9. User data is scoped to the authenticated account

Clerk `sub` claims are upserted into MongoDB users. Interview reads filter by both the authenticated MongoDB user ID and interview ID. A short per-user cache reduces repeated list queries, and writes invalidate that cache.

### 10. Browser APIs handle the interactive voice loop

The React client records microphone input with `MediaRecorder`, sends WebM audio to `/api/transcribe`, renders responses progressively, and plays returned audio through an HTML audio element. If provider TTS is unavailable, it uses `window.speechSynthesis`. The UI keeps an active interview separate from a read-only saved-session view.

## Repository history and documentation changes

### Existing project history

- The initial implementation commit added the FastAPI backend, React/Vite client, Docker deployment files, interview corpora, provider integrations, prompt templates, persistence layer, and UI.
- The following commit removed the root `README.md`, leaving the repository without a current setup or architecture guide.

### Changes made in this documentation pass

- Added a new root `README.md` with the product overview, architecture diagram, repository map, local setup, environment variables, production Docker flow, API table, limitations, and verification commands.
- Added this `decisions.md` to explain why the current architecture and provider choices were made and to record the historical documentation gap.
- Updated the root `.gitignore` to explicitly exclude client dependencies/build output, Python caches and test caches, local environment files, logs, and generated database files.
- Kept `README.md` and `decisions.md` tracked intentionally. They are project documentation, not generated artifacts, so ignoring them would make the requested record disappear from version control.

## Known follow-up opportunities

- Replace the lexical company retriever with embeddings and a persistent vector index if the corpus grows.
- Move active sessions to a shared store if multiple application replicas or restart resilience are required.
- Add automated backend tests for authentication, file parsing, rate limiting, provider fallback, and ownership checks.
- Add a backend dependency lock file and a client deployment proxy configuration for reproducible local development.
- Remove or clearly archive `src/utils.py`, `public/index.html`, and the stale `run.sh` instructions once compatibility with older deployments is no longer needed.
