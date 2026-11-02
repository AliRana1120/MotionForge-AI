# MotionForge AI — Implementation Tasks

## Milestone 0 · Project Scaffold (Day 1, ~3h)

- [ ] **T-01** Initialize Django project under `backend/` with `motionforge` settings package (base/development/production).
- [ ] **T-02** Configure PostgreSQL connection in `settings/development.py` and create initial migrations.
- [ ] **T-03** Install and configure Celery with Redis broker in `motionforge/celery.py`; add health-check task.
- [ ] **T-04** Initialize Vite + React project under `frontend/` with React Router and Axios.
- [ ] **T-05** Write `docker-compose.yml` covering: `db` (postgres:16), `redis` (redis:7-alpine), `backend` (Django + Gunicorn), `worker` (Celery), `frontend` (Vite dev server).
- [ ] **T-06** Create `.env.example` with all required environment variables documented.
- [ ] **T-07** Configure CORS in Django to allow requests from `localhost:5173` in development.

---

## Milestone 1 · Video Upload & Job Creation (Day 1–2, ~4h)

- [ ] **T-08** Define `Job` model in `apps/jobs/models.py` with all fields from the data model spec.
- [ ] **T-09** Write `JobSerializer` (create + detail variants) in `apps/jobs/serializers.py`.
- [ ] **T-10** Implement `POST /api/jobs/` view: validate file (MIME + size), save to UUID path, create Job record, enqueue Celery chain, return `{ job_id, session_token }`.
- [ ] **T-11** Implement `GET /api/jobs/{id}/` view with `X-Session-Token` guard; return status + current_stage + partial style_profile.
- [ ] **T-12** Build `UploadZone.jsx` frontend component: drag-and-drop, file type/size validation, progress bar via `XMLHttpRequest.upload.onprogress`.
- [ ] **T-13** Build `useUpload.js` hook that calls `POST /api/jobs/` and surfaces `{ jobId, sessionToken, progress, error }`.
- [ ] **T-14** Build `useJobPoller.js` hook that polls `GET /api/jobs/{id}/` every 2 seconds and returns live job state.

---

## Milestone 2 · Frame Extraction (Day 2, ~3h)

- [ ] **T-15** Implement Celery task `extract_frames` in `apps/jobs/tasks/extract.py`:
  - FFmpeg uniform sampling (1 frame / 2s).
  - OpenCV histogram scene-change detection.
  - Deduplicate + cap at 20 frames.
  - Save frame metadata to `Job.style_profile` (partial update).
  - Update `Job.current_stage = ANALYZE`.
- [ ] **T-16** Implement `GET /api/jobs/{id}/frames/` endpoint returning thumbnail URLs.
- [ ] **T-17** Configure Django to serve media files in development (`MEDIA_URL` / `MEDIA_ROOT`).
- [ ] **T-18** Build `FrameGallery.jsx` component: display thumbnail grid once frames are available.

---

## Milestone 3 · Style Analysis (Day 2–3, ~4h)

- [ ] **T-19** Implement Celery task `analyze_style` in `apps/jobs/tasks/analyze.py`:
  - K-means color palette extraction (scikit-learn `MiniBatchKMeans`).
  - Brightness and contrast computation (numpy).
  - Motion intensity from inter-frame pixel differences.
  - Write complete `StyleProfile` to `Job.style_profile`.
- [ ] **T-20** (Optional / stretch) Add CLIP scene tagging: pre-compute tag embeddings, load CLIP ViT-B/32, compute similarity at inference time.
- [ ] **T-21** Build `StyleBadges.jsx` component: display color swatches, motion intensity bar, and scene tags from `style_profile`.
- [ ] **T-22** Write unit tests for palette extraction and motion intensity calculation.

---

## Milestone 4 · Template Library (Day 3, ~5h)

- [ ] **T-23** Create `TemplateRegistry` in `apps/templates_lib/registry.py` with rule-based `select(style_profile)` method.
- [ ] **T-24** Build **Template 1: `neon-dark`** — Three.js full-screen hero with particle field, frame carousel as 3D floating cards, dark background, neon color overlays.
- [ ] **T-25** Build **Template 2: `minimal-light`** — Clean layout, subtle Three.js floating geometry, frames as flat image cards, light background.
- [ ] **T-26** Build **Template 3: `cinematic`** — Full-bleed video-poster frame, Three.js camera drift animation, film-grain overlay, widescreen crop.
- [ ] **T-27** Ensure every template reads exclusively from `src/config/style.js` for all colors, motion settings, and asset paths — no hardcoded values.

---

## Milestone 5 · Website Generation & Preview (Day 3–4, ~4h)

- [ ] **T-28** Implement Celery task `generate_website` in `apps/jobs/tasks/generate.py`:
  - Call `TemplateRegistry.select(style_profile)`.
  - Copy template source tree to `media/jobs/{id}/generated/`.
  - Inject `StyleProfile` values into `style.js` via string templating.
  - Copy extracted frames to `generated/public/assets/frames/`.
  - Set `Job.preview_token`, `Job.status = COMPLETE`.
- [ ] **T-29** Configure Django to serve `media/jobs/{id}/generated/` as static site under `/preview/{preview_token}/`.
- [ ] **T-30** Build `PreviewPanel.jsx` component: full-width iframe pointing to `/preview/{token}/`, with a toolbar showing "Open in new tab" and "Export" buttons.
- [ ] **T-31** Build `JobProgress.jsx` component: step indicator (Upload → Extract → Analyze → Generate → Done) driven by `job.current_stage`.

---

## Milestone 6 · Export (Day 4, ~2h)

- [ ] **T-32** Implement `GET /api/jobs/{id}/export/` view: stream a ZIP of `media/jobs/{id}/generated/` as `application/zip`.
- [ ] **T-33** Inject `README.md` into the ZIP explaining: `npm install && npm run dev`.
- [ ] **T-34** Wire "Export Project" button in `PreviewPanel.jsx` to trigger download.

---

## Milestone 7 · Polish & Hackathon Demo Prep (Day 4–5, ~4h)

- [ ] **T-35** Add error handling: surface `Job.error_message` in `JobProgress.jsx` with a retry option.
- [ ] **T-36** Add temp file cleanup: Django management command + Celery periodic task to delete jobs older than 24h.
- [ ] **T-37** Style the main app UI (upload → progress → preview flow) with a cohesive dark-themed design.
- [ ] **T-38** Record a 2-minute demo video using a visually rich MP4 (e.g., a drone footage clip) to showcase all three templates.
- [ ] **T-39** Update `README.md` with full setup instructions, architecture diagram link, and demo GIF.
- [ ] **T-40** Verify the full Docker Compose stack starts cleanly with `docker compose up --build`.

---

## Risk Mitigations Baked into Tasks

| Risk | Mitigation Task |
|------|-----------------|
| CLIP model too slow on CPU | T-20 marked optional; scene tags gracefully absent from StyleProfile |
| FFmpeg not installed in env | T-05 Docker image includes `apt install ffmpeg` |
| Large video exceeds RAM during analysis | T-15 caps frames at 20; numpy operations work per-frame, not all-at-once |
| Preview iframe CORS issues | T-07 + T-29 configure CORS and same-origin preview serving |
| Template build takes too long at demo time | Templates are pre-built static assets; no `npm build` at generation time |
