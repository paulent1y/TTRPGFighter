# Audio Transcription Web App — Specifications

## 1. Overview

A Python-based web application that transcribes audio files using locally running Ollama models. The user selects a model and an audio file through the browser UI, then monitors the transcription pipeline in real time. The resulting transcript is displayed in an editable output field with options to copy or download it.

---

## 2. Goals

- Run entirely locally with no cloud dependency (Ollama as the inference backend).
- Provide clear, step-by-step visual feedback so the user always knows what the app is doing.
- Fail gracefully with actionable error messages.
- Keep the UI minimal and self-explanatory.

---

## 3. Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.11+, FastAPI |
| ASGI server | Uvicorn |
| Frontend | Vanilla HTML / CSS / JavaScript (no framework) |
| LLM interface | Ollama REST API (`http://localhost:11434` by default) |
| Audio pre-processing | `pydub` + `ffmpeg` (format normalisation) |
| HTTP client | `httpx` (async, for Ollama calls) |
| Real-time updates | Server-Sent Events (SSE) over a `/api/stream/{job_id}` endpoint |

---

## 4. Architecture

```
Browser
  │  HTTP / SSE
  ▼
FastAPI backend   ──── REST ────►  Ollama  (localhost:11434)
  │
  └── serves static HTML/JS/CSS
```

- The backend owns all communication with Ollama; the frontend never contacts Ollama directly.
- Long-running transcription jobs run in a background `asyncio` task; the frontend polls progress via SSE.
- Uploaded audio files are held in a temporary directory and deleted after the job finishes (success or failure).

---

## 5. Configuration

All tunables live in a single `config.py` (or `.env` file read by `python-dotenv`):

| Key | Default | Description |
|---|---|---|
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama API root |
| `OLLAMA_TIMEOUT_SECONDS` | `300` | Max wait for a single Ollama response |
| `MAX_UPLOAD_MB` | `200` | Hard cap on uploaded audio file size |
| `ALLOWED_AUDIO_EXTENSIONS` | `.mp3 .wav .m4a .ogg .flac .webm .mp4` | Accepted extensions |
| `HOST` | `127.0.0.1` | Interface the web server binds to |
| `PORT` | `8000` | TCP port |
| `LOG_LEVEL` | `info` | Uvicorn log level |

---

## 6. Backend API Endpoints

### 6.1 `GET /api/health`

Checks whether Ollama is reachable.

**Response 200**
```json
{ "status": "ok", "ollama_reachable": true }
```

**Response 200** (Ollama down)
```json
{ "status": "degraded", "ollama_reachable": false, "detail": "Connection refused" }
```

---

### 6.2 `GET /api/models`

Fetches the model list from `ollama list` (internally calls `GET /api/tags`).

**Response 200**
```json
{
  "models": [
    { "name": "llama3.2-vision:11b", "size_gb": 6.8 },
    { "name": "gemma3:12b",          "size_gb": 7.4 }
  ]
}
```

**Response 502** — Ollama unreachable.

---

### 6.3 `POST /api/transcribe`

Starts a transcription job.

**Request** — `multipart/form-data`

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | binary | yes | Audio file |
| `model` | string | yes | Model name from `/api/models` |

**Response 202**
```json
{ "job_id": "a1b2c3d4" }
```

**Response 400** — Unsupported format, file too large, or missing fields.

**Response 502** — Ollama unreachable at job start.

---

### 6.4 `GET /api/stream/{job_id}`

Server-Sent Events stream. Each event carries a JSON payload.

**Event: `status`** — pipeline stage update
```json
{ "event": "status", "stage": "sending_request", "message": "Sending request to Ollama..." }
```

Possible stages (in order):

| Stage key | User-facing message |
|---|---|
| `checking_connection` | Checking Ollama connection… |
| `uploading` | Receiving audio file… |
| `preprocessing` | Normalising audio format… |
| `preparing_request` | Preparing transcription request… |
| `sending_request` | Sending request to Ollama… |
| `computing` | Ollama is computing the transcript… |
| `done` | Transcription complete. |

**Event: `token`** — incremental transcript text (streaming from Ollama)
```json
{ "event": "token", "text": "Hello, " }
```

**Event: `result`** — full final transcript
```json
{ "event": "result", "transcript": "Hello, welcome to the meeting..." }
```

**Event: `error`** — job failed
```json
{ "event": "error", "code": "OLLAMA_TIMEOUT", "message": "Ollama did not respond within 300 s." }
```

Error codes:

| Code | Cause |
|---|---|
| `OLLAMA_UNREACHABLE` | Cannot connect to Ollama |
| `OLLAMA_TIMEOUT` | Request timed out |
| `MODEL_NOT_FOUND` | Selected model not loaded in Ollama |
| `UNSUPPORTED_FORMAT` | Audio format rejected after inspection |
| `FILE_TOO_LARGE` | Upload exceeds `MAX_UPLOAD_MB` |
| `PREPROCESSING_FAILED` | `ffmpeg` conversion error |
| `INTERNAL_ERROR` | Unexpected server error |

---

## 7. Ollama Integration Detail

### 7.1 Encoding Audio for the Model

Ollama multimodal models accept base64-encoded binary content in the `images` array of the `/api/generate` payload. Audio files must be converted to a compatible representation:

1. Use `pydub` + `ffmpeg` to re-encode the audio to 16 kHz mono WAV (lowest common denominator).
2. Base64-encode the WAV bytes.
3. Pass them in the `images` field.

> **Note for implementers:** Not all Ollama models accept audio. The UI should display a non-blocking warning badge next to models that are text- or vision-only, determined by checking the model's `details.families` field from the Ollama API. If the chosen model does not support audio natively, the backend should still attempt the request and surface any Ollama-level error clearly.

### 7.2 API Call Shape

```python
payload = {
    "model": selected_model,
    "prompt": "<audio content provided as base64 above>",
    "system": SYSTEM_PROMPT,       # see Section 8
    "images": [base64_audio_str],
    "stream": True,
    "options": {
        "temperature": 0.0,        # deterministic output
        "num_predict": 8192
    }
}
# POST http://localhost:11434/api/generate
```

Stream the response line-by-line; each line is a JSON object with a `response` key. Concatenate `response` values and emit them as `token` SSE events.

---

## 8. System Prompt

```
You are an expert audio transcriptionist. Your sole task is to produce a
verbatim, accurate written transcript of the audio content provided to you.

Rules you must follow:
1. Transcribe every spoken word exactly as said — do not paraphrase, summarise,
   or correct the speaker's grammar.
2. If multiple speakers are present, label each turn as [Speaker 1], [Speaker 2],
   etc. Maintain consistent labels throughout the transcript.
3. Mark unintelligible or inaudible segments with [inaudible].
4. Mark non-speech audio events that are relevant to meaning (e.g. laughter,
   applause, long pause) with [event description].
5. Add standard punctuation (commas, periods, question marks) to improve
   readability without altering meaning.
6. Do NOT include any commentary, preamble, or conclusion outside the
   transcript itself — output the transcript and nothing else.
7. Format timestamps as [HH:MM:SS] at the start of each speaker turn if the
   audio duration is longer than 60 seconds.
```

---

## 9. Frontend UI

### 9.1 Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  Ollama Audio Transcriber              ● Ollama: Connected       │
├─────────────────────────────────────────────────────────────────┤
│  Model                                                          │
│  ┌─────────────────────────────────────────────┐  [↺ Refresh]  │
│  │  llama3.2-vision:11b (6.8 GB)           ▼  │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
│  Audio File                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Drop file here or click to browse                      │   │
│  │  Accepted: .mp3 .wav .m4a .ogg .flac .webm .mp4        │   │
│  └─────────────────────────────────────────────────────────┘   │
│  meeting_recording.mp3  (12.4 MB)                               │
│                                                                 │
│                                [  Transcribe  ]  ← disabled    │
├─────────────────────────────────────────────────────────────────┤
│  Progress                                                       │
│  ✔ Ollama connection verified                                   │
│  ✔ Audio file received                                          │
│  ✔ Audio normalised (WAV 16 kHz mono)                           │
│  ⟳ Sending request to Ollama…                                  │
│  ○ Computing transcript                                         │
├─────────────────────────────────────────────────────────────────┤
│  Transcript                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ [00:00:00] [Speaker 1] Hello everyone, thanks for       │   │
│  │ joining today's meeting…                                │   │
│  └─────────────────────────────────────────────────────────┘   │
│  1,243 words  |  [Copy]  [Download .txt]                        │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 Ollama Status Badge (top-right)

- Checked automatically on page load.
- Re-checked every 15 seconds via `GET /api/health`.
- **Green dot + "Ollama: Connected"** — healthy.
- **Red dot + "Ollama: Unreachable"** — error state; Transcribe button stays disabled and shows tooltip.

### 9.3 Model Selector

- Populated from `GET /api/models` on load.
- Shows model name and file size.
- If list is empty, shows: *"No models found — run `ollama pull <model>` to add one."*
- Refresh button re-calls `/api/models` and rebuilds the dropdown.
- Selecting a model with a non-audio architecture shows an orange warning icon with tooltip: *"This model may not support audio input. Results may be empty or incorrect."*

### 9.4 File Picker

- Drag-and-drop zone + click-to-browse `<input type="file">`.
- Accepted MIME types and extensions enforced client-side first, then validated server-side.
- Displays file name and size after selection.
- Shows red validation message if file exceeds `MAX_UPLOAD_MB`.

### 9.5 Transcribe Button

| Condition | State |
|---|---|
| Model not selected OR file not selected OR Ollama unreachable | Disabled (grey) |
| All inputs ready | Enabled (primary colour) |
| Job in progress | Disabled + spinner + label "Transcribing…" |
| Job complete | Re-enabled, label resets to "Transcribe" |

### 9.6 Progress Panel

Appears below the Transcribe button once the job starts. Each step shows one of:
- `○` — pending (grey)
- `⟳` — active (animated, blue)
- `✔` — complete (green)
- `✗` — failed (red)

Steps are populated from incoming `status` SSE events.

### 9.7 Transcript Output

- Hidden until the first `token` event arrives.
- Tokens are appended in real time as they stream from Ollama, so the user sees the transcript being written character-by-character.
- Text area is read-only by default; **Edit** toggle makes it editable.
- Word count and character count update live.
- **Copy** button: copies full transcript to clipboard, button label briefly changes to "Copied!".
- **Download .txt** button: triggers browser download of `transcript_<filename>.txt`.

### 9.8 Error Display

- A dismissible red alert banner appears at the top of the progress panel.
- Shows the human-readable `message` from the `error` SSE event.
- Includes a **Retry** link that resets the UI to ready state without clearing the file/model selection.

---

## 10. File & Directory Structure

```
audio_transcriber/
├── main.py                  # FastAPI app, route definitions
├── config.py                # Settings (env vars + defaults)
├── transcribe.py            # Job logic: preprocessing + Ollama calls
├── models.py                # Pydantic schemas
├── static/
│   ├── index.html           # Single-page UI
│   ├── app.js               # Frontend logic (fetch, SSE, DOM)
│   └── style.css            # Styles
├── requirements.txt
└── README.md
```

---

## 11. Non-Functional Requirements

| Requirement | Target |
|---|---|
| Startup time | < 3 s from `uvicorn main:app` |
| UI responsiveness | All interactions feel instant (< 100 ms feedback) |
| Memory footprint (backend) | < 256 MB excluding model weights |
| Concurrent jobs | 1 (serial queue; second request returns 429) |
| Temp file cleanup | Guaranteed via `finally` block, even on error |
| Browser support | Latest Chrome, Firefox, Safari, Edge |

---

## 12. Error Handling Matrix

| Scenario | Backend action | UI feedback |
|---|---|---|
| Ollama not running | Return `ollama_reachable: false` in `/api/health` | Red badge; Transcribe disabled |
| Model disappears mid-job | Catch 404 from Ollama, emit `error` event | Error banner with "Model not found" |
| Network blip during streaming | Close SSE; emit `error` with `OLLAMA_TIMEOUT` | Error banner with Retry option |
| ffmpeg not installed | Log warning; skip preprocessing step, use raw file | Non-blocking console warning only |
| Upload exceeds size limit | Reject with HTTP 400 before processing starts | Red validation text under file picker |
| Unsupported file extension | Reject with HTTP 400 | Red validation text under file picker |

---

## 13. Out of Scope (v1)

- User accounts or job history persistence.
- Batch processing of multiple files.
- Speaker diarisation beyond simple labelling by the LLM.
- Deployment outside localhost (no auth, TLS, or reverse proxy config).
- Whisper-specific integration (all transcription delegated to the selected Ollama model).
