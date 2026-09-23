---
title: AI Interviewer
emoji: microphone
colorFrom: red
colorTo: yellow
sdk: docker
app_port: 7860
pinned: false
---

# AI Interviewer

AI Interviewer is a voice-first interview practice application for software engineering candidates. It runs structured interviews, asks follow-up questions based on the conversation, accepts an optional resume, reviews submitted code, and returns evidence-based feedback after the interview.

The application is called **NoFoxAI** in the user interface.

## What it does

- Supports DSA/coding, system design, behavioral/HR, and company-specific interview modes.
- Uses staged interview prompts so the interviewer moves from introduction to technical or behavioral probing and then closing.
- Accepts PDF, DOCX, and TXT resumes up to 5 MB and limits extracted context to 8,000 characters.
- Uses voice input through browser MediaRecorder and server-side speech-to-text providers.
- Speaks interviewer responses through provider TTS, with browser speech synthesis as a fallback.
- Provides a browser code editor for Python, JavaScript, Java, and C++ submissions.
- Grounds Amazon and Microsoft company interviews in the bundled DOCX interview corpus using keyword-ranked retrieval.
- Generates structured feedback with an overall score, strengths, pain points, improvement areas, hiring verdict, skill scores, and next steps.
- Supports Clerk authentication and MongoDB-backed saved interviews for signed-in users.

## Architecture

```text
React + Vite client
        |
        | REST calls and Clerk bearer token
        v
FastAPI application (src/main.py)
        |
        +--> LLM: Groq -> Google Gemini -> OpenRouter
        +--> STT: Groq Whisper -> Deepgram
        +--> TTS: Groq Orpheus -> ElevenLabs -> browser fallback
        +--> RAG: keyword retrieval from data/*.docx
        +--> MongoDB: users and saved interview transcripts
```

The Docker image builds the client first, copies `client/dist` into `public/`, and then serves the compiled single-page application and API from the Python process on port `7860`.

## Repository layout

```text
src/main.py                 FastAPI routes, sessions, provider fallbacks, file parsing
src/interview_templates.py  Interview plans, stage guidance, and prompt construction
src/rag.py                  DOCX loading and company-context retrieval
src/analytics.py            Structured post-interview feedback generation
src/storage.py              MongoDB persistence and saved-session queries
src/utils.py                Older MongoDB helper module; not used by the current app
client/src/App.tsx          React interview setup, voice flow, editor, and feedback UI
client/src/main.tsx         React and Clerk bootstrap
client/src/index.css        Tailwind CSS entry point
client/package.json         Client scripts and dependencies
data/Amazon.docx           Amazon interview source corpus
data/Microsoft.docx        Microsoft interview source corpus
Dockerfile                  Multi-stage client and Python production image
Procfile                    Platform process command
```

## Local development

### Prerequisites

- Python 3.11
- Node.js 22 and npm
- A Groq API key for the primary LLM and optional speech services
- A Clerk application if using the authenticated UI
- MongoDB if saved interviews are required

### 1. Configure the backend

Create a root `.env` from `.env.example` and set the provider values you plan to use:

```dotenv
GROQ_API_KEY=your_groq_key
GROQ_MODEL=openai/gpt-oss-20b
GOOGLE_API_KEY=optional_gemini_key
OPENROUTER_API_KEY=optional_openrouter_key
DEEPGRAM_API_KEY=optional_deepgram_key
ELEVENLABS_API_KEY=optional_elevenlabs_key
CLERK_JWT_ISSUER=https://your-clerk-issuer
CLERK_JWKS_URL=https://your-clerk-issuer/.well-known/jwks.json
MONGO_URI=mongodb+srv://...
MONGO_DB_NAME=ai_interviewer
```

The backend can start without MongoDB, but authenticated saved-session endpoints will not work. The primary Groq key is also optional at startup, but an LLM provider must be available to conduct an interview.

### 2. Install and run the backend

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
uvicorn src.main:app --reload --port 8000
```

### 3. Install and run the client

```powershell
cd client
npm ci
Copy-Item .env.example .env
npm run dev
```

Set `VITE_CLERK_PUBLISHABLE_KEY` in `client/.env`. Leave `VITE_BACKEND_URL` empty when the Vite client is proxied or served from the same origin; set it to `http://localhost:8000` when the development client calls the backend directly.

The client provides the interactive UI. The root `public/index.html` is a legacy static artifact and is not used by the Docker build, which replaces `public/` with the current Vite output.

## Production build

Build and run the bundled image from the repository root:

```powershell
docker build --build-arg VITE_CLERK_PUBLISHABLE_KEY=your_public_key -t ai-interviewer .
docker run --env-file .env -p 7860:7860 ai-interviewer
```

Open `http://localhost:7860`. The production process is also defined in `Procfile` for platforms that provide a `$PORT` variable.

## API surface

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/api/auth/me` | Return the current Clerk-authenticated user, if any |
| GET | `/api/sessions` | List the current user's saved interviews |
| GET | `/api/sessions/{interview_id}` | Load one saved interview owned by the current user |
| POST | `/api/start-interview` | Create an interview and return its opening question |
| POST | `/api/respond` | Submit a spoken answer and optional code for the next question |
| POST | `/api/transcribe` | Convert recorded WebM audio to text |
| POST | `/api/synthesize` | Generate interviewer audio or signal browser TTS fallback |
| POST | `/api/end-interview` | End the session and generate structured feedback |

All request routes have an in-memory limit of 30 requests per client IP per 60 seconds.

## Notes and limitations

- Active interview sessions are held in process memory, so a restart loses unfinished sessions.
- The server supports multiple LLM providers, but feedback analysis currently uses the configured Groq client only.
- Company retrieval is intentionally lightweight keyword overlap rather than embedding or vector-database search.
- Resume text is sent to the configured LLM as prompt context; do not upload sensitive information you do not want processed by those providers.
- Configure Clerk issuer/JWKS values and MongoDB in deployment secrets. Never commit `.env` files or credentials.

## Verification commands

```powershell
cd client
npm run build
npm run lint
```

## License

No license file is currently included in the repository. Add one before distributing the project under a specific license.
