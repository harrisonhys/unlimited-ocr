# PRD: PaddleOCR Service

> Product Requirements Document for self-hosted OCR microservice using [PaddlePaddle/PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR) — supports Indonesian & English.

**Author:** Hari (harrisonhys)  
**Date:** 2026-06-24  
**Status:** Draft  

---

## 1. Problem Statement

Need reliable, unlimited OCR — no API quotas, no per-request billing. Cloud OCR (Google Vision, Azure) cost money; Tesseract accuracy too low on complex Indonesian documents (mixed fonts, poor scan quality, tables).

PaddleOCR provides SOTA accuracy with models under 100MB, supports 100+ languages including Indonesian (Latin script), and runs on both CPU and GPU. Self-hosting removes vendor lock-in.

## 2. Goals

| # | Goal | MVP |
|---|------|-----|
| G1 | User sends image → returns extracted text (ID + EN) | MVP-1 |
| G2 | User sends PDF → returns extracted text (all pages) | MVP-1 |
| G3 | Image preprocessing: resize/compress large inputs | MVP-1 |
| G4 | REST API with async job queue | MVP-1 |
| G5 | Multi-worker inference | MVP-2 |
| G6 | Batch upload (multiple files) | MVP-2 |
| G7 | API key auth + rate limiting | MVP-2 |

## 3. Non-Goals (MVP-1)

- ❌ Web UI / frontend
- ❌ Document layout visualization
- ❌ Fine-tuning / training
- ❌ Billing / usage metering
- ❌ Formula / table structure extraction (use PaddleOCR-VL later)

## 4. Architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│   Client     │────▶│  API Gateway │────▶│  Job Queue       │
│  (HTTP/REST) │◀────│  (FastAPI)   │◀────│  (Redis/Celery)  │
└─────────────┘     └──────────────┘     └────────┬────────┘
                                                   │
                                          ┌────────▼────────┐
                                          │  Worker Pool     │
                                          │  (Celery)        │
                                          │                  │
                                          │  ┌────────────┐  │
                                          │  │ PaddleOCR  │  │
                                          │  │  (engine)   │  │
                                          │  └────────────┘  │
                                          └─────────────────┘
```

### Components

| Component | Tech | Purpose |
|-----------|------|---------|
| API Server | FastAPI + uvicorn | HTTP endpoints, validation, job dispatch |
| Job Queue | Celery + Redis | Async task management, retry, status tracking |
| Workers | Celery workers | Execute OCR jobs |
| OCR Engine | PaddleOCR (Python API) | Detection + recognition inference |
| Storage | Local filesystem | Temporary files, auto-cleanup |
| Models | PP-OCRv4 / PP-OCRv5 | Pre-trained multilingual models |

### Why PaddleOCR

- **Model size:** 10.5MB (mobile) to 81MB (server) — fits anywhere
- **Languages:** 100+ langs including Indonesian (Latin script family)
- **Speed:** CPU ~17ms recognition (mobile), GPU ~1ms (server high-perf)
- **Backends:** CPU (OpenVINO), GPU (TensorRT/CUDA), Apple Silicon
- **No external API:** 100% offline after model download

## 5. Language Support

### Indonesian + English Strategy

| Aspect | Detail |
|--------|--------|
| Script | Indonesian uses Latin alphabet — covered by PaddleOCR Latin/multilingual models |
| Recommended model | `PP-OCRv4_server_rec_doc` (182MB) or `PP-OCRv5_server_rec` (81MB) for best accuracy |
| Mobile alternative | `PP-OCRv4_mobile_rec` (10.5MB) for CPU-only deployments |
| Language code | Use `lang='en'` or `lang='latin'` in PaddleOCR init — both cover English + Indonesian characters |
| Special chars | All standard Latin chars (a-z, A-Z, é, è, ê, ñ, etc.) supported |

### Model Selection Matrix

| Deployment | Model | Size | GPU VRAM | CPU RAM | Latency/img |
|------------|-------|------|----------|---------|-------------|
| Edge / low-spec | PP-OCRv4_mobile | 10.5 MB | 0 GB | 2 GB | ~50-100 ms |
| Balanced | PP-OCRv5_mobile | 16 MB | 0 GB | 4 GB | ~30-50 ms |
| Server (recommended) | PP-OCRv5_server | 81 MB | 2-4 GB | 8 GB | ~10-20 ms |
| High-accuracy docs | PP-OCRv4_server_doc | 182 MB | 4-6 GB | 16 GB | ~15-25 ms |

**MVP-1 default:** PP-OCRv5_server (best accuracy/size ratio, supports EN + ID).

## 6. API Design

### Endpoints

```
POST   /api/v1/ocr                    # Submit OCR job (image or PDF)
GET    /api/v1/ocr/{job_id}           # Get job status + result
DELETE /api/v1/ocr/{job_id}           # Cancel job
GET    /health                        # Health check
```

### POST /api/v1/ocr

**Request:** `multipart/form-data`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | File | ✅ | Image (jpg/png/webp) or PDF |
| `lang` | string | ❌ | `id_en` (default) or `en` |
| `dpi` | int | ❌ | PDF render DPI (default: 300) |
| `priority` | string | ❌ | `normal` (default) or `high` |

**Image Constraints (MVP-1):**

| Constraint | Value | Rationale |
|------------|-------|-----------|
| Max file size | 20MB | Pre-upload check |
| Max resolution | 4096px longest side | Auto-resize if exceeded |
| Max PDF pages | 50 | Truncate + warn if exceeded |
| Supported formats | jpg, png, webp, pdf | Auto-detect |

**Response (202 Accepted):**

```json
{
  "job_id": "ocr_a1b2c3d4",
  "status": "queued",
  "created_at": "2026-06-24T10:00:00Z",
  "estimated_seconds": 5
}
```

### GET /api/v1/ocr/{job_id}

```json
{
  "job_id": "ocr_a1b2c3d4",
  "status": "completed",
  "result": {
    "text": "Extracted text content...",
    "pages": 3,
    "processing_time_ms": 4200,
    "language": "id_en",
    "confidence_avg": 0.92
  },
  "meta": {
    "file_name": "invoice.pdf",
    "file_type": "pdf",
    "model": "PP-OCRv5_server"
  }
}
```

**Status values:** `queued` → `processing` → `completed` | `failed`

**Error response:**

```json
{
  "job_id": "ocr_a1b2c3d4",
  "status": "failed",
  "error": {
    "code": "PROCESSING_ERROR",
    "message": "OCR inference timeout after 60s"
  }
}
```

## 7. Image Preprocessing Pipeline

```
Input Image
    │
    ▼
┌──────────────┐
│ Validate     │ ← check format, size, corruption
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Resize       │ ← if longest side > 4096px, resize proportionally
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Compress     │ ← if > 5MB, JPEG quality reduce to 85
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ PaddleOCR    │ ← detect text boxes, recognize text
│ inference    │
└──────┬───────┘
       │
       ▼
   Text + confidence scores
```

**Detection input sizing:**

| Model Tier | Detection input size | Rationale |
|------------|---------------------|-----------|
| Mobile | 736 px longest side | Fast, low memory |
| Server | 1280 px longest side | Better small text detection |

## 8. PDF Processing Pipeline

```
Input PDF
    │
    ▼
┌──────────────┐
│ Page count   │ ← > 50 pages: warn + truncate
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ PyMuPDF      │ ← render each page to PNG @ 300 DPI
│ to images    │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Preprocess   │ ← same image pipeline per page
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ PaddleOCR    │ ← run OCR per page
│ per page     │
└──────┬───────┘
       │
       ▼
   Concatenated text result
```

## 9. Project Structure

```
~/projects/unlimited-ocr/
├── PRD.md                          # This document
├── docker-compose.yml              # Full stack: api + worker + redis
├── Dockerfile.api                  # API server image
├── Dockerfile.worker               # Worker + PaddleOCR image
├── requirements.txt                # Python deps
├── .env.example                    # Template for secrets
├── src/
│   ├── __init__.py
│   ├── main.py                     # FastAPI app entry
│   ├── config.py                   # Settings (env-based)
│   ├── api/
│   │   ├── __init__.py
│   │   ├── routes_ocr.py           # /api/v1/ocr endpoints
│   │   └── middleware.py           # Auth, rate limit, logging
│   ├── core/
│   │   ├── __init__.py
│   │   ├── preprocessor.py         # Image resize/compress/validate
│   │   ├── pdf_converter.py        # PDF → images via PyMuPDF
│   │   └── ocr_engine.py           # PaddleOCR wrapper
│   ├── workers/
│   │   ├── __init__.py
│   │   ├── celery_app.py           # Celery config
│   │   └── tasks.py                # OCR task definitions
│   └── models/
│       ├── __init__.py
│       └── schemas.py              # Pydantic models
├── tests/
│   ├── __init__.py
│   ├── test_preprocessor.py
│   ├── test_pdf_converter.py
│   ├── test_ocr_engine.py
│   ├── test_api.py
│   └── fixtures/
│       ├── sample_invoice_id.jpg
│       ├── sample_document_en.pdf
│       └── large_image.png
├── scripts/
│   ├── setup.sh                    # First-time setup
│   ├── download_models.py          # Download PaddleOCR models
│   └── benchmark.py                # Load test / perf benchmark
└── docs/
    └── api.md                      # API documentation
```

## 10. Docker Compose (MVP-1)

```yaml
# docker-compose.yml
version: "3.8"

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    ports:
      - "8000:8000"
    environment:
      - REDIS_URL=redis://redis:6379/0
      - MAX_FILE_SIZE_MB=20
      - MAX_PDF_PAGES=50
      - TEMP_DIR=/tmp/ocr
      - OCR_MODEL=PP-OCRv5_server
      - OCR_LANG=id_en
    depends_on:
      - redis
    volumes:
      - ./models:/app/models:ro

  worker:
    build:
      context: .
      dockerfile: Dockerfile.worker
    environment:
      - REDIS_URL=redis://redis:6379/0
      - OCR_MODEL=PP-OCRv5_server
      - OCR_LANG=id_en
      - CELERY_CONCURRENCY=2
      - USE_GPU=false
    depends_on:
      - redis
    volumes:
      - ./models:/app/models:ro
    command: celery -A src.workers.celery_app worker --loglevel=info --concurrency=2

volumes:
  redis_data:
```

## 11. MVP-2: Multi-Worker + GPU

```yaml
# docker-compose.scale.yml (override)
services:
  worker:
    deploy:
      replicas: 4
    environment:
      - USE_GPU=true
      - CELERY_CONCURRENCY=1
    runtime: nvidia
```

**Scaling strategy:**

| Setup | Workers | GPU per worker | Concurrency |
|-------|---------|----------------|-------------|
| CPU small | 2 | 0 | 2 threads each |
| CPU medium | 4 | 0 | 2 threads each |
| GPU single | 1 | 1 | 1 process |
| GPU multi | 4 | 1 each | 1 process each |

**GPU note:** PaddleOCR with GPU uses ~2-4GB VRAM per worker. Single 16GB GPU can run 3-4 workers.

## 12. Hardware Requirements

### Minimum (MVP-1, CPU-only)

| Resource | Spec | Notes |
|----------|------|-------|
| CPU | 4 vCPU | Any modern x86_64 |
| RAM | 8 GB | Model ~81MB + PDF rendering |
| Disk | 50 GB | Model weights + temp files |
| GPU | None | CPU inference acceptable for low traffic |

### Recommended (MVP-1, moderate load)

| Resource | Spec | Notes |
|----------|------|-------|
| CPU | 8 vCPU | FastAPI + workers + Redis |
| RAM | 16 GB | Comfortable for concurrent PDFs |
| Disk | 100 GB SSD | Temp file I/O critical |
| GPU | Optional | 4-6 GB VRAM if using GPU |

### High-Throughput (MVP-2, GPU)

| Resource | Spec | Notes |
|----------|------|-------|
| GPU | 1-2x NVIDIA, 16GB+ VRAM | T4, A10, RTX 4090 |
| RAM | 32 GB | Multiple workers |
| Disk | 200 GB SSD | Model cache + outputs |

### Cloud Options

| Provider | Instance | GPU | Cost/hr |
|----------|----------|-----|---------|
| Lambda Labs | gpu_1x_a10 | A10 24GB | $0.60 |
| RunPod | A100 40GB | A100 | $1.20 |
| Vast.ai | RTX 4090 | 4090 24GB | $0.35 |
| Current VPS | 14GB RAM, no GPU | CPU only | $existing |

**Current VPS (14GB RAM, no GPU):** Can run CPU mode with PP-OCRv5_server. Sufficient for dev + low traffic.

## 13. Tech Stack Summary

| Layer | Technology | Version |
|-------|-----------|---------|
| Language | Python | 3.10+ |
| API Framework | FastAPI | 0.115+ |
| Task Queue | Celery | 5.4+ |
| Message Broker | Redis | 7+ |
| OCR Engine | PaddleOCR | 3.7.0+ |
| Model | PP-OCRv5_server | latest |
| PDF Processing | PyMuPDF (fitz) | 1.27+ |
| Image Processing | Pillow | 10+ |
| Containerization | Docker + Compose | 24+ |
| HTTP Client | httpx | 0.28+ |

## 14. Milestones & Timeline

### MVP-1: Single Worker OCR Service (Week 1-2)

| Day | Task | Deliverable |
|-----|------|-------------|
| D1 | Project scaffold, Dockerfile, deps | Empty project running |
| D2 | Download models (PP-OCRv5_server, EN+ID) | Models cached locally |
| D3 | Image preprocessor (validate, resize, compress) | `preprocessor.py` + tests |
| D4 | PDF converter (PyMuPDF render) | `pdf_converter.py` + tests |
| D5 | PaddleOCR engine wrapper (ID+EN) | `ocr_engine.py` + tests |
| D6 | Celery task definitions | `tasks.py` + job queue working |
| D7 | FastAPI endpoints (POST, GET, DELETE) | API + tests |
| D8 | Docker Compose full stack | `docker-compose.yml` up |
| D9 | Integration tests, error handling | End-to-end working |
| D10 | Documentation, README, deploy notes | `docs/api.md` |

### MVP-2: Multi-Worker + Scaling (Week 3-4)

| Day | Task |
|-----|------|
| D11-12 | GPU support + TensorRT acceleration |
| D13-14 | Celery worker pool scaling |
| D15-16 | Batch upload endpoint |
| D17-18 | API key auth + rate limiting |

## 15. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Indonesian accuracy on poor scans | Bad OCR | Use server model + 300 DPI, add confidence threshold |
| CPU inference too slow for batch | User timeout | Async jobs + progress tracking; add GPU later |
| PyMuPDF memory on large PDFs | OOM | Stream pages, process in batches of 10 |
| Model download blocked (China CDN) | Can't start | Mirror models to HuggingFace, pre-download in Dockerfile |
| Mixed-language text (ID + EN) | Wrong model selection | Use multilingual model by default, auto-detect not needed |

## 16. Success Metrics

| Metric | Target (MVP-1) | Target (MVP-2) |
|--------|----------------|-----------------|
| Single image latency | < 3s (CPU) | < 500ms (GPU) |
| 10-page PDF latency | < 30s (CPU) | < 5s (GPU) |
| Concurrent requests | 2 | 10+ |
| Uptime | 99% | 99.5% |
| Max file size | 20MB | 50MB |
| Max PDF pages | 50 | 200 |

## 17. Open Questions

1. **GPU procurement:** Rent cloud GPU or stay CPU-only on current VPS?
2. **Deployment target:** Docker on current VPS, or split API + workers?
3. **Auth scope:** API key per user, or single shared key for internal use?
4. **Monitoring:** Prometheus + Grafana, or simple logs + health endpoint?
5. **Backup strategy:** Job results persisted to DB or just Redis ephemeral?

---

*Next step: Confirm hardware strategy, then scaffold project.*
