# MotionForge AI — System Design

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Browser                                     │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  React + Vite Frontend                                        │   │
│  │                                                              │   │
│  │  ┌──────────────┐  ┌─────────────────┐  ┌───────────────┐   │   │
│  │  │ Upload UI    │  │ Progress Tracker│  │ Preview iframe│   │   │
│  │  └──────┬───────┘  └────────┬────────┘  └───────┬───────┘   │   │
│  │         │                   │ polling            │           │   │
│  └─────────┼───────────────────┼────────────────────┼───────────┘   │
│            │ REST              │                    │               │
└────────────┼───────────────────┼────────────────────┼───────────────┘
             │                   │                    │
             ▼                   ▼                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Django REST Framework API                       │
│                                                                     │
│  POST /api/jobs/           GET /api/jobs/{id}/       GET /preview/  │
│  (upload video)            (poll status)             {token}/       │
│          │                                                          │
│          ▼                                                          │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    Celery Task Queue                           │  │
│  │                                                               │  │
│  │  Task 1: extract_frames  →  Task 2: analyze_style             │  │
│  │           (FFmpeg/OpenCV)          (K-means + CLIP)           │  │
│  │                                         │                     │  │
│  │  Task 3: generate_website  ◄────────────┘                     │  │
│  │           (template engine + asset injection)                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
│          │                                                          │
│  ┌───────┴──────────┐    ┌──────────────────┐                      │
│  │  PostgreSQL DB   │    │  Redis (Broker)  │                      │
│  │  (Job metadata)  │    │  + Celery results│                      │
│  └──────────────────┘    └──────────────────┘                      │
│                                                                     │
│  Local Filesystem (or S3-compatible in future)                      │
│  /media/jobs/{job_id}/frames/  →  generated/  →  export.zip        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Directory Structure

```
MotionForge-AI/
├── backend/                        # Django project root
│   ├── manage.py
│   ├── requirements.txt
│   ├── motionforge/                # Django project settings package
│   │   ├── settings/
│   │   │   ├── base.py
│   │   │   ├── development.py
│   │   │   └── production.py
│   │   ├── urls.py
│   │   └── celery.py
│   ├── apps/
│   │   ├── jobs/                   # Core processing app
│   │   │   ├── models.py           # Job model
│   │   │   ├── serializers.py
│   │   │   ├── views.py
│   │   │   ├── urls.py
│   │   │   └── tasks/
│   │   │       ├── __init__.py
│   │   │       ├── extract.py      # FR-02: FFmpeg + OpenCV frame extraction
│   │   │       ├── analyze.py      # FR-03: color palette + CLIP analysis
│   │   │       └── generate.py     # FR-04: template selection + site generation
│   │   └── templates_lib/          # Template library app
│   │       ├── models.py           # Template metadata
│   │       ├── registry.py         # Template selector (rule-based)
│   │       └── templates/          # Pre-built React template source trees
│   │           ├── neon-dark/
│   │           ├── minimal-light/
│   │           └── cinematic/
│   └── media/                      # Uploaded files and generated output (gitignored)
│
├── frontend/                       # React + Vite app
│   ├── package.json
│   ├── vite.config.js
│   ├── index.html
│   └── src/
│       ├── main.jsx
│       ├── App.jsx
│       ├── components/
│       │   ├── UploadZone.jsx       # Drag-and-drop upload
│       │   ├── JobProgress.jsx      # Step-by-step progress tracker
│       │   ├── FrameGallery.jsx     # Thumbnail preview of extracted frames
│       │   ├── PreviewPanel.jsx     # iframe + export button
│       │   └── StyleBadges.jsx      # Displays detected color palette + tags
│       ├── hooks/
│       │   ├── useJobPoller.js      # Polls /api/jobs/{id}/ every 2s
│       │   └── useUpload.js         # Handles chunked upload + progress
│       ├── api/
│       │   └── client.js            # Axios instance + typed API wrappers
│       └── styles/
│           └── global.css
│
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## 3. Data Models

### Job
```
Job
├── id              UUID (PK)
├── status          Enum: QUEUED | PROCESSING | COMPLETE | ERROR
├── current_stage   Enum: UPLOAD | EXTRACT | ANALYZE | GENERATE
├── video_path      CharField  (relative path under /media/)
├── session_token   CharField  (UUID, given to browser for auth-lite)
├── style_profile   JSONField  (null until ANALYZE completes)
├── template_id     CharField  (null until GENERATE completes)
├── preview_token   CharField  (UUID, null until GENERATE completes)
├── error_message   TextField  (null on success)
├── created_at      DateTimeField
└── updated_at      DateTimeField
```

### StyleProfile (JSON schema, stored in Job.style_profile)
```json
{
  "palette": ["#1a1a2e", "#16213e", "#0f3460", "#e94560"],
  "brightness": 0.22,
  "contrast": 0.71,
  "motion_intensity": 0.65,
  "scene_tags": ["urban", "night", "motion"],
  "aspect_ratio": "16:9",
  "frame_count": 12,
  "frame_paths": ["frames/00001.jpg", "frames/00005.jpg", "..."]
}
```

---

## 4. API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/jobs/` | Upload video, create job, return `{ job_id, session_token }` |
| `GET` | `/api/jobs/{id}/` | Poll job status; requires `X-Session-Token` header |
| `GET` | `/api/jobs/{id}/frames/` | Return thumbnail URLs for extracted frames |
| `GET` | `/preview/{preview_token}/` | Serve the generated static preview site |
| `GET` | `/api/jobs/{id}/export/` | Stream export ZIP; requires `X-Session-Token` header |

---

## 5. Processing Pipeline Detail

### Stage 1 — Frame Extraction (Celery Task: `extract_frames`)
1. Receive `job_id`.
2. Open video with FFmpeg subprocess: extract 1 frame/2s → JPEG → `media/jobs/{id}/frames/`.
3. Run OpenCV scene-change detection: compute per-frame histogram difference; flag frames where diff > threshold.
4. Keep union of uniform-sampled + scene-change frames, deduplicate, cap at 20 frames.
5. Update `Job.current_stage = ANALYZE`, save frame metadata to `Job.style_profile` (partial).
6. Chain to `analyze_style` task.

### Stage 2 — Style Analysis (Celery Task: `analyze_style`)
1. Load all extracted JPEG frames as numpy arrays.
2. Stack pixel samples, run `sklearn.cluster.MiniBatchKMeans(n_clusters=5)` → dominant palette (hex codes).
3. Compute mean brightness (average luminance across frames) and contrast (std of luminance).
4. Compute motion intensity: mean absolute difference of consecutive frames, normalized to [0, 1].
5. (Optional) Run CLIP `encode_image` on 3 keyframes, compute cosine similarity against a fixed set of scene-tag embeddings (pre-computed offline, shipped with the app). Tags with similarity > 0.25 are included.
6. Assemble `StyleProfile` dict, write to `Job.style_profile`.
7. Chain to `generate_website` task.

### Stage 3 — Website Generation (Celery Task: `generate_website`)
1. Call `TemplateRegistry.select(style_profile)` → returns template name (rule-based lookup table).
2. Copy template source tree to `media/jobs/{id}/generated/`.
3. Run a **simple AST-free string substitution** pass over `src/config/style.js` inside the template — inject palette, motion intensity, frame asset paths.
4. Copy extracted frames to `generated/public/assets/frames/`.
5. Write `src/config/style.js` with generated values.
6. Mark `Job.status = COMPLETE`, set `Job.preview_token`.
7. Serve `generated/` as static files under `/preview/{preview_token}/`.

---

## 6. Template Library Design

Each template is a standalone Vite + React project pre-built around a configurable `style.js`:

```
templates/neon-dark/
├── package.json         (dependencies pinned)
├── vite.config.js
├── index.html
├── public/
│   └── assets/
│       └── frames/      (← frame images injected here)
└── src/
    ├── main.jsx
    ├── App.jsx
    ├── config/
    │   └── style.js     (← generated values injected here)
    ├── components/
    │   ├── HeroScene.jsx       (Three.js canvas, full-screen)
    │   ├── ParticleField.jsx   (particle system driven by motion_intensity)
    │   ├── FrameCarousel.jsx   (extracted frames as 3D cards)
    │   └── ColorOverlay.jsx    (palette-driven gradient overlays)
    └── hooks/
        └── useScrollAnimation.js
```

The `style.js` contract:
```js
export default {
  palette: ['#1a1a2e', '#16213e', '#0f3460', '#e94560'],
  motionIntensity: 0.65,           // drives particle speed, animation easing
  brightness: 0.22,                // drives ambient light intensity in Three.js
  frames: ['/assets/frames/00001.jpg', '/assets/frames/00005.jpg'],
  aspectRatio: '16:9',
  tags: ['urban', 'night'],
};
```

---

## 7. Technology Decisions & Rationale

| Decision | Choice | Reason |
|----------|--------|--------|
| Task queue | Celery + Redis | Free, battle-tested, horizontally scalable, good Django integration |
| Color analysis | scikit-learn K-means | Zero cost, runs on CPU in <1s, accurate enough for palette extraction |
| Scene tagging | open_clip (CLIP ViT-B/32) | Free, runs on CPU, pre-trained, no fine-tuning needed |
| Frame extraction | FFmpeg CLI + OpenCV | FFmpeg is the industry standard; OpenCV adds scene-change detection |
| DB | PostgreSQL | Chosen for JSON field support (StyleProfile), free, robust |
| 3D rendering | Three.js via React Three Fiber | Most popular, huge ecosystem, works in-browser, no backend GPU |
| Preview serving | Django `FileResponse` / whitenoise | No extra infra needed for MVP |
| Export | Python `zipfile` streaming | Zero dependency, sufficient for MVP |

---

## 8. Extension Points (Future 3D Reconstruction)

The architecture is deliberately designed to allow future stages to be inserted into the pipeline without breaking existing stages:

- **New Celery task `reconstruct_3d`** can be inserted between `analyze_style` and `generate_website`.
- `StyleProfile` has a reserved `mesh_path` field (currently null) that `generate_website` will use if present.
- The template library accepts an optional `Three.js GLTFLoader` scene when a mesh is available.
- Potential future integrations: Gaussian Splatting (3DGS), instant-ngp, or Shap-E (all open-source, GPU-optional).
- The `TemplateRegistry` can be upgraded from rule-based selection to a small classifier or LLM-based generation without touching the pipeline orchestration.
