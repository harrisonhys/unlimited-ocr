# PRD: Unlimited-OCR Service

> Product Requirements Document for self-hosted OCR service based on [baidu/Unlimited-OCR](https://github.com/baidu/Unlimited-OCR)

**Author:** Hari (harrisonhys)  
**Date:** 2026-06-24  
**Status:** Draft  

---

## 1. Problem Statement

Need reliable, unlimited OCR capability — no API quotas, no per-image billing. Existing cloud OCR (Google Vision, Azure, Tesseract) either cost money per request or produce low-quality results on complex documents (tables, handwriting, multi-column layouts).

Baidu's Unlimited-OCR model achieves SOTA on long-horizon document parsing via single-shot inference, built on DeepSeek-OCR architecture. Self-hosting removes vendor lock-in and usage caps.

## 2. Goals

| # | Goal | MVP |
|---|------|-----|
| G1 | User sends image → returns extracted text | MVP-1 |
| G2 | User sends PDF → returns extracted text (all pages) | MVP-1 |
| G3 | Image preprocessing: resize/compress large inputs | MVP-1 |
| G4 | REST API with async job queue | MVP-1 |
| G5 | Multi-worker GPU inference | MVP-2 |
| G6 | Streaming response for large documents | MVP-2 |
| G7 | Batch upload (multiple files) | MVP-2 |
| G8 | Auth / API key management | MVP-2 |

## 3. Non-Goals (MVP-1)

- ❌ Web UI / frontend
- ❌ Document layout visualization
- ❌ Fine-tuning / training
- ❌ Multi-language selection (model auto-detects)
- ❌ Billing / usage metering

## 4. Architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│   Client     │────▶│  API Gateway │────▶│  Job Queue       │
│  (HTTP/REST) │◀────│  (FastAPI)   │◀────│  (Redis/Celery)  │
└─────────────┘     └──────────────┘     └────────┬────────┘
                                                   │
                                          ┌────────▼────────┐
                                          │  Worker Pool     │
                                          │  (Celery + GPU)  │
                                          │                  │
                                          │  ┌────────────┐  │
                                          │  │ SGLang Srv │  │
                                          │  │ Unlimited  │  │
                                          │  │   OCR      │  │
                                          │  └────────────┘  │
                                          └─────────────────┘
```

### Components

| Component | Tech | Purpose |
|-----------|------|---------|
| API Server | FastAPI + uvicorn | HTTP endpoints, validation, job dispatch |
| Job Queue | Celery + Redis | Async task management, retry, status tracking |
| Workers | Celery workers | Execute OCR jobs against model |
| Inference | SGLang server | High-throughput model serving (OpenAI-compatible API) |
| Storage | Local filesystem | Temporary file storage (input/output), auto-cleanup |
| Model | baidu/Unlimited-OCR | Document parsing VLM (~16GB VRAM) |

### Why SGLang over raw Transformers

- SGLang provides OpenAI-compatible API endpoint → clean worker separation
- Built-in batching, streaming, KV-cache management
- Custom logit processor support (no_repeat_ngram) already integrated
- Can scale to multi-GPU later via tensor parallelism

## 5. API Design

### Endpoints

```
POST   /api/v1/ocr                    # Submit OCR job (image or PDF)
GET    /api/v1/ocr/{job_id}           # Get job status + result
GET    /api/v1/ocr/{job_id}/stream    # SSE stream for large docs
DELETE /api/v1/ocr/{job_id}           # Cancel job
GET    /health                        # Health check
```

### POST /api/v1/ocr

**Request:** `multipart/form-data`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | File | ✅ | Image (jpg/png/webp) or PDF |
| `mode` | string | ❌ | `gundam` (crop, default) or `base` |
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
  "estimated_seconds": 15
}
```

### GET /api/v1/ocr/{job_id}

```json
{
  "job_id": "ocr_a1b2c3d4",
  "status": "completed",
  "result": {
    "text": "Extracted markdown/text content...",
    "pages": 3,
    "processing_time_ms": 12400
  },
  "meta": {
    "file_name": "invoice.pdf",
    "file_type": "pdf",
    "mode": "gundam",
    "model": "Unlimited-OCR"
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
    "message": "Model inference timeout after 120s"
  }
}
```

## 6. Image Preprocessing Pipeline

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
│ Mode Select  │ ← gundam (crop small regions) or base (full image)
└──────┬───────┘
       │
       ▼
   SGLang inference
```

**Why resize:** Model trained on `image_size=640` (gundam) or `1024` (base). Sending 4K+ images wastes memory without improving accuracy. Resize before sending to model.

## 7. PDF Processing Pipeline

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
│ infer_multi  │ ← batch all pages in single model call
└──────┬───────┘
       │
       ▼
   Concatenated text result
```

## 8. Project Structure

```
~/projects/unlimited-ocr/
├── PRD.md                          # This document
├── docker-compose.yml              # Full stack: api + worker + redis + sglang
├── Dockerfile.api                  # API server image
├── Dockerfile.worker               # Worker + model image
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
│   │   └── ocr_client.py           # SGLang API client
│   ├── workers/
│   │   ├── __init__.py
│   │   ├── celery_app.py           # Celery config
│   │   └── tasks.py                # OCR task definitions
│   └── models/
│       ├── __init__.py
│       └── schemas.py              # Pydantic models (Job, Result, etc.)
├── tests/
│   ├── __init__.py
│   ├── test_preprocessor.py
│   ├── test_pdf_converter.py
│   ├── test_api.py
│   └── fixtures/                   # Sample images/PDFs for testing
│       ├── sample_invoice.jpg
│       ├── sample_document.pdf
│       └── large_image.png
├── scripts/
│   ├── setup.sh                    # First-time setup
│   ├── start_sglang.sh             # Launch SGLang model server
│   └── benchmark.py                # Load test / perf benchmark
└── docs/
    └── api.md                      # API documentation
```

## 9. Docker Compose (MVP-1: single worker)

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

  sglang:
    image: sglang:latest  # Custom build with model
    build:
      context: .
      dockerfile: Dockerfile.worker
    ports:
      - "10000:10000"
    environment:
      - CUDA_VISIBLE_DEVICES=0
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    command: >
      python -m sglang.launch_server
        --model baidu/Unlimited-OCR
        --served-model-name Unlimited-OCR
        --attention-backend fa3
        --page-size 1
        --mem-fraction-static 0.8
        --context-length 32768
        --enable-custom-logit-processor
        --disable-overlap-schedule
        --skip-server-warmup
        --host 0.0.0.0
        --port 10000

  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    ports:
      - "8000:8000"
    environment:
      - REDIS_URL=redis://redis:6379/0
      - SGLANG_URL=http://sglang:10000
      - MAX_FILE_SIZE_MB=20
      - MAX_PDF_PAGES=50
      - TEMP_DIR=/tmp/ocr
    depends_on:
      - redis
      - sglang

  worker:
    build:
      context: .
      dockerfile: Dockerfile.api
    environment:
      - REDIS_URL=redis://redis:6379/0
      - SGLANG_URL=http://sglang:10000
      - CELERY_CONCURRENCY=2
    depends_on:
      - redis
      - sglang
    command: celery -A src.workers.celery_app worker --loglevel=info --concurrency=2

volumes:
  redis_data:
```

## 10. MVP-2: Multi-Worker Scaling

```yaml
# docker-compose.scale.yml (override for MVP-2)
services:
  sglang-0:
    extends: sglang
    environment:
      - CUDA_VISIBLE_DEVICES=0

  sglang-1:
    extends: sglang
    environment:
      - CUDA_VISIBLE_DEVICES=1

  worker:
    deploy:
      replicas: 4
    environment:
      - SGLANG_URLS=http://sglang-0:10000,http://sglang-1:10000
      - LOAD_BALANCE=round-robin
```

**Multi-GPU strategy:**

| Approach | Pros | Cons |
|----------|------|------|
| Multiple SGLang instances (1 per GPU) | Simple, independent | No cross-GPU batching |
| SGLang tensor parallelism | Better large-batch perf | More complex setup |
| Celery round-robin across instances | Easy horizontal scaling | Need health checks |

MVP-2 start: multiple SGLang instances + Celery round-robin. Simpler, proven.

## 11. Infrastructure Requirements

### Minimum (MVP-1, single worker)

| Resource | Spec | Notes |
|----------|------|-------|
| GPU | 1x NVIDIA GPU, 16GB+ VRAM | RTX 4090, A100, L40S |
| RAM | 32GB | Model loading + PyMuPDF rendering |
| Disk | 100GB | Model weights (~15GB) + temp files |
| CPU | 8 cores | API + worker + Redis |
| OS | Ubuntu 22.04+ | CUDA 12.9 compatible |

### Recommended (MVP-2, multi-worker)

| Resource | Spec | Notes |
|----------|------|-------|
| GPU | 2-4x NVIDIA GPU, 24GB+ VRAM | A100 40GB ideal |
| RAM | 64GB | Multiple workers |
| Disk | 500GB SSD | Temp file I/O critical |

### Cloud Options

| Provider | Instance | GPU | Cost/hr (approx) |
|----------|----------|-----|-------------------|
| Lambda Labs | gpu_1x_a10 | A10 24GB | $0.60 |
| RunPod | A100 40GB | A100 | $1.20 |
| Vast.ai | RTX 4090 | 4090 24GB | $0.35 |
| Modal | T4/A10G | pay-per-use | $0.00046/sec |

**Dev VPS (current):** No GPU. Use remote GPU instance for model serving, keep API + Redis on VPS.

## 12. Tech Stack Summary

| Layer | Technology | Version |
|-------|-----------|---------|
| Language | Python | 3.12 |
| API Framework | FastAPI | 0.115+ |
| Task Queue | Celery | 5.4+ |
| Message Broker | Redis | 7+ |
| Model Serving | SGLang | 0.0.0.dev+ |
| Model | baidu/Unlimited-OCR | latest |
| PDF Processing | PyMuPDF (fitz) | 1.27+ |
| Image Processing | Pillow | 12.1+ |
| Containerization | Docker + Compose | 24+ |
| HTTP Client | httpx | 0.28+ |

## 13. Milestones & Timeline

### MVP-1: Single Worker OCR Service (Week 1-2)

| Day | Task | Deliverable |
|-----|------|-------------|
| D1 | Project scaffold, Dockerfile, deps | Empty project running |
| D2 | SGLang server setup + health check | Model loads and responds |
| D3 | Image preprocessor (validate, resize, compress) | `preprocessor.py` + tests |
| D4 | PDF converter (PyMuPDF render) | `pdf_converter.py` + tests |
| D5 | SGLang client (single/multi image) | `ocr_client.py` + tests |
| D6 | Celery task definitions | `tasks.py` + job queue working |
| D7 | FastAPI endpoints (POST, GET, DELETE) | API + tests |
| D8 | Docker Compose full stack | `docker-compose.yml` up |
| D9 | Integration tests, error handling | End-to-end working |
| D10 | Documentation, README, deploy notes | `docs/api.md` |

### MVP-2: Multi-Worker + Scaling (Week 3-4)

| Day | Task |
|-----|------|
| D11-12 | Multi-GPU SGLang setup + load balancing |
| D13-14 | Celery worker pool scaling |
| D15-16 | Streaming response (SSE) |
| D17-18 | Batch upload endpoint |
| D19-20 | API key auth + rate limiting |

## 14. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Model requires 16GB+ VRAM | Can't run on consumer GPUs < 16GB | Use quantized version if available, or rent A100 |
| SGLang wheel is custom build | Install fragility | Pin exact wheel version in Dockerfile, cache in private registry |
| Long inference time (>30s per page) | User timeout | Async jobs + SSE streaming + progress updates |
| No GPU on current dev VPS | Can't test locally | Docker Compose with remote SGLang endpoint |
| PyMuPDF memory on large PDFs | OOM on 100+ page docs | Stream pages, process in batches of 10 |
| Model hallucination on poor quality images | Bad OCR output | Input validation + quality warnings + confidence scores (future) |

## 15. Success Metrics

| Metric | Target (MVP-1) | Target (MVP-2) |
|--------|----------------|-----------------|
| Single image latency | < 10s | < 5s |
| 10-page PDF latency | < 60s | < 30s |
| Concurrent requests | 1 | 10+ |
| Uptime | 99% | 99.5% |
| Max file size | 20MB | 50MB |
| Max PDF pages | 50 | 200 |

## 16. Open Questions

1. **GPU procurement:** Rent cloud GPU (RunPod/Vast.ai) or buy dedicated? Affects cost model.
2. **Deployment target:** Docker on new GPU VPS, or split (API on current VPS + GPU elsewhere)?
3. **Auth scope:** API key per user, or single shared key for internal use?
4. **Monitoring:** Prometheus + Grafana, or simpler (just logs + health endpoint)?
5. **Backup strategy:** Job results persisted to DB (Postgres) or just Redis ephemeral?

---

*Next step: Confirm GPU strategy, then scaffold project.*
