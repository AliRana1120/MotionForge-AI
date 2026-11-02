# MotionForge AI — Requirements

## Overview

MotionForge AI is a web application that takes an uploaded MP4 video and converts it into an interactive, 3D-animated website landing page. The MVP targets hackathon demonstration and is designed to run locally or on a minimal free-tier cloud instance.

---

## Functional Requirements

### FR-01 · Video Upload
- Users can upload an MP4 video file from their local machine via a drag-and-drop or file-picker interface.
- Accepted format: MP4 (H.264 encoded). Maximum file size: 100 MB for MVP.
- The system provides real-time upload progress feedback.
- The system validates the file type and size before uploading and shows a clear error if rejected.

### FR-02 · Frame Extraction
- Upon successful upload, the backend extracts a set of representative frames from the video using FFmpeg.
- Extraction strategy: uniform sampling (one frame per N seconds, configurable) plus scene-change detection via OpenCV to capture visually distinct moments.
- Extracted frames are stored temporarily on the server (cleaned up after session ends or after 24 hours).
- The system exposes a frame preview endpoint so the frontend can show thumbnails to the user.

### FR-03 · Visual Style Analysis
- The backend analyzes the extracted frames to derive the video's visual identity:
  - **Color palette**: dominant colors extracted via K-means clustering (scikit-learn, free).
  - **Brightness / contrast profile**: determines dark vs. light aesthetic.
  - **Motion intensity**: estimated from inter-frame pixel difference — determines how animated the generated site should feel.
  - **Scene classification**: optional, lightweight tag inference using a small pre-trained model (e.g., CLIP ViT-B/32 via `open_clip`, free and runs on CPU).
- Results are structured as a `StyleProfile` JSON object and stored with the job.

### FR-04 · Website Generation
- Using the `StyleProfile`, the system selects and configures a **website template** from a library of pre-built React + Three.js templates.
- Template selection is rule-based for MVP (e.g., dark palette → "neon-dark" template; bright/natural → "minimal-light" template).
- The system injects extracted frames as textures and background imagery into the selected template.
- Generated output is a self-contained React project directory (Vite scaffold).
- A "preview token" is returned to the frontend so the user can preview the result immediately.

### FR-05 · Interactive Preview
- The frontend renders the generated website in an isolated `<iframe>` using a served static preview build.
- The Three.js scene in the preview is interactive: users can orbit the 3D elements, and animations play automatically.
- Preview is available for at least 1 hour after generation (configurable TTL).

### FR-06 · Project Export
- Users can click "Export Project" to download a ZIP archive of the generated React project.
- The ZIP contains: `src/`, `public/` (with extracted frame assets), `package.json`, `vite.config.js`, and a `README.md` explaining how to run it.
- The export is generated on-demand by the backend and streamed directly to the browser.

### FR-07 · Job Status & Progress
- Every processing pipeline (upload → extract → analyze → generate) is tracked as an asynchronous **Job**.
- The frontend polls (or uses WebSocket) for job status updates: `queued → processing → complete → error`.
- Users see a step-by-step progress indicator matching the pipeline stages.
- On error, a human-readable message and suggested resolution are shown.

---

## Non-Functional Requirements

### NFR-01 · Performance
- Frame extraction for a 30-second video should complete within 10 seconds on a standard developer laptop.
- Style analysis should complete within 5 seconds on CPU (no GPU required for MVP).
- Website generation (template rendering + asset injection) should complete within 15 seconds.
- Total pipeline time target: under 30 seconds for a 30-second, 1080p video.

### NFR-02 · Scalability (future-readiness)
- The processing pipeline is implemented as discrete, loosely coupled stages so individual stages can be swapped or scaled independently.
- Background job processing uses Celery + Redis so workers can be scaled horizontally without code changes.

### NFR-03 · Reliability
- Failed jobs must not leave orphaned temp files; a cleanup task runs on job failure and on TTL expiry.
- All API endpoints return structured JSON error responses with a consistent `{ error, detail, code }` shape.

### NFR-04 · Security
- Uploaded files are stored under a UUID-named path, never under a user-controlled filename.
- File type is validated server-side via MIME sniffing (python-magic), not just extension.
- Temp files and generated assets are isolated per-job and inaccessible across jobs.
- For MVP there is no authentication; a simple per-session UUID token is used to gate access to job results.

### NFR-05 · Resource Efficiency
- No paid external API calls are required for the MVP.
- All AI/ML inference runs on CPU using lightweight models (CLIP ViT-B/32 is ~350 MB; K-means is negligible).
- FFmpeg and OpenCV are used in-process; no separate media server is needed.

### NFR-06 · Developer Experience
- The project is split into clearly separated `frontend/` and `backend/` directories with independent dependency management.
- A single `docker-compose.yml` can bring up the full stack (Django + Celery + Redis + Postgres + Vite dev server) with one command.
- Hot-reload is supported for both the React frontend (Vite HMR) and Django backend (auto-reloader).

### NFR-07 · Portability
- The backend runs on Python 3.11+.
- The frontend runs on Node.js 20+ / npm 10+.
- No OS-specific dependencies beyond FFmpeg (installable via package manager on Linux, macOS, and Windows).

---

## Out of Scope for MVP
- User accounts and authentication (beyond session token).
- Full photogrammetry or neural radiance field (NeRF) 3D reconstruction.
- Real-time collaborative editing.
- Mobile app.
- Paid API integrations (OpenAI, Replicate, etc.).
- Video editing or trimming inside the application.
- Support for video formats other than MP4.
