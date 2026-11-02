---
inclusion: always
---

# MotionForge AI — Project Context for Kiro

## What this project is
MotionForge AI converts uploaded MP4 videos into interactive 3D-animated React website landing pages.

## Stack
- **Frontend**: React 18 + Vite 5, React Three Fiber, Three.js r165, React Router v6, Axios
- **Backend**: Django 5 + Django REST Framework, Celery 5, Redis 7
- **Processing**: FFmpeg (CLI), OpenCV-Python, scikit-learn (K-means), open_clip (CLIP ViT-B/32)
- **Database**: PostgreSQL 16
- **Infra**: Docker Compose (local dev)

## Directory layout
- `backend/` — Django project (`motionforge/` settings package, `apps/jobs/`, `apps/templates_lib/`)
- `frontend/` — React + Vite app (`src/components/`, `src/hooks/`, `src/api/`)
- `backend/apps/templates_lib/templates/` — Three pre-built website templates (`neon-dark`, `minimal-light`, `cinematic`)

## Key constraints
- No paid APIs — all AI/ML is free and CPU-only.
- MVP only — no NeRF / Gaussian Splatting / full 3D reconstruction yet.
- Template selection is rule-based (not ML) for MVP.
- Per-job session tokens only — no user accounts.

## Processing pipeline order
`extract_frames` (FFmpeg + OpenCV) → `analyze_style` (K-means + optional CLIP) → `generate_website` (template copy + style.js injection)

## When implementing any task, match the patterns in the spec:
- Job model fields and the `StyleProfile` JSON schema are in `.kiro/specs/motionforge-ai/design.md`
- API endpoint signatures are in the same design doc
- All template components must read from `src/config/style.js` only — no hardcoded colors or asset paths
