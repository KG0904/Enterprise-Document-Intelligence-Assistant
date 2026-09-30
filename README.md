# 🧠 Enterprise Document Intelligence Assistant

A production-ready, multi-tenant document ingestion and intelligence backend built with **FastAPI**, **PostgreSQL + pgvector**, **Celery**, and **Redis**. It extracts, chunks, embeds, and stores content from 12+ file formats, with pluggable OCR and embedding providers.

---

## ✨ Features

- 📄 **Universal Document Parsing** — PDF (native + OCR fallback), DOCX, PPTX, XLSX/XLS, HTML, EML, TXT, MD, CSV, images (PNG/JPG/TIFF/etc.), and ZIP archives
- 🔍 **Smart Chunking** — Fixed-size (token-aware), sentence-boundary, and Markdown-header strategies with configurable size and overlap
- 🧮 **Vector Embeddings** — Pluggable provider: OpenAI `text-embedding-3-small`, HuggingFace `all-MiniLM-L6-v2`, or deterministic mock for development
- 🔒 **Multi-Tenant Auth** — JWT access + refresh tokens; org-scoped RBAC (Admin / member roles); SHA-256 duplicate detection; automatic document versioning
- ⚡ **Async Ingestion Pipeline** — Celery workers process documents asynchronously (extract → chunk → embed) without blocking the API
- 🗄️ **pgvector Storage** — Embeddings stored in PostgreSQL via the `pgvector` extension for fast cosine-similarity search
- 🐳 **Docker-first** — One `docker-compose up` spins up Postgres (pgvector) + Redis
- 📊 **Audit Logging**, **API Key management**, **Chat sessions** with feedback

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         FastAPI Backend                          │
│                                                                  │
│  /api/v1/auth        JWT signup · login · refresh               │
│  /api/v1/users       User profile management                    │
│  /api/v1/projects    Project CRUD (org-scoped)                  │
│  /api/v1/documents   Upload · list · preview                    │
│  /api/v1/tasks       Task status polling                        │
└────────────────────────┬─────────────────────────────────────────┘
                         │ triggers
                         ▼
              ┌──────────────────────┐
              │   Celery Worker      │
              │  (ingestion.py)      │
              │                      │
              │  1. Validate file    │
              │  2. Extract text     │
              │  3. Chunk text       │
              │  4. Generate embeds  │
              │  5. Store in DB      │
              └──────────┬───────────┘
                         │
          ┌──────────────┴──────────────┐
          │                             │
          ▼                             ▼
  PostgreSQL + pgvector              Redis
  (documents, chunks,            (Celery broker
   embeddings, users)              & result store)
```

---

## 🚀 Quick Start

### 1. Prerequisites

- **Python 3.11+**
- **Docker & Docker Compose**
- *(Optional)* Tesseract OCR for scanned PDFs

### 2. Clone & Configure

```bash
git clone https://github.com/KG0904/Enterprise-Document-Intelligence-Assistant.git
cd Enterprise-Document-Intelligence-Assistant/backend
cp .env.example .env
```

Edit `.env` with your settings (see [Environment Variables](#-environment-variables)).

### 3. Start Infrastructure

```bash
docker-compose up -d
```

This starts:
- **PostgreSQL 16 + pgvector** on port `5433`
- **Redis 7** on port `6379`

### 4. Install Dependencies

```bash
pip install -r requirements.txt
```

### 5. Run Database Migrations

```bash
alembic upgrade head
```

### 6. Start the API Server

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

### 7. Start the Celery Worker

```bash
celery -A app.core.celery_app worker --loglevel=info
```

API is now live at **http://localhost:8000**  
Interactive docs: **http://localhost:8000/docs** · **http://localhost:8000/redoc**

---

## 📁 Project Structure

```
backend/
├── app/
│   ├── main.py                   # FastAPI app factory, CORS, router registration
│   ├── api/
│   │   ├── deps.py               # Shared dependencies (DB session, auth guards)
│   │   └── v1/
│   │       ├── auth.py           # Signup, login, token refresh
│   │       ├── documents.py      # Upload, list, preview
│   │       ├── projects.py       # Project CRUD
│   │       ├── tasks.py          # Task status
│   │       └── users.py          # User profile
│   ├── core/
│   │   ├── config.py             # Pydantic settings (env-driven)
│   │   ├── security.py           # Password hashing, JWT creation
│   │   └── celery_app.py         # Celery broker configuration
│   ├── db/
│   │   └── session.py            # Async SQLAlchemy session factory
│   ├── models/                   # SQLAlchemy ORM models
│   │   ├── organization.py
│   │   ├── user.py
│   │   ├── project.py
│   │   ├── document.py
│   │   ├── chunk.py              # Chunk + ChunkEmbedding (pgvector)
│   │   ├── chat.py               # ChatSession, ChatMessage, Feedback
│   │   ├── api_key.py
│   │   └── audit.py
│   ├── schemas/                  # Pydantic request/response schemas
│   ├── services/
│   │   ├── extractor.py          # Multi-format text extraction + OCR fallback
│   │   ├── chunker.py            # Fixed-size / sentence / Markdown chunking
│   │   ├── embedding.py          # OpenAI / HuggingFace / mock embeddings
│   │   └── ocr.py                # Tesseract / EasyOCR / PaddleOCR adapter
│   └── tasks/
│       └── ingestion.py          # Celery async ingestion pipeline
├── .env.example
├── alembic.ini
├── docker-compose.yml
└── requirements.txt
```

---

## 🔧 Environment Variables

Copy `.env.example` to `.env` and configure:

| Variable | Default | Description |
|---|---|---|
| `PROJECT_NAME` | `Enterprise Document Intelligence Assistant` | API title |
| `ENV` | `development` | `development` \| `production` \| `testing` |
| `DEBUG` | `True` | Enable debug mode |
| `HOST` | `0.0.0.0` | Bind host |
| `PORT` | `8000` | Bind port |
| `POSTGRES_HOST` | `localhost` | PostgreSQL host |
| `POSTGRES_PORT` | `5433` | PostgreSQL port |
| `POSTGRES_USER` | `postgres` | PostgreSQL user |
| `POSTGRES_PASSWORD` | `postgrespassword` | PostgreSQL password |
| `POSTGRES_DB` | `document_intelligence` | Database name |
| `REDIS_HOST` | `localhost` | Redis host |
| `REDIS_PORT` | `6379` | Redis port |
| `SECRET_KEY` | *(generated)* | JWT signing key — **change in production!** |
| `ALGORITHM` | `HS256` | JWT algorithm |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | `30` | Access token lifetime |
| `REFRESH_TOKEN_EXPIRE_DAYS` | `7` | Refresh token lifetime |
| `OCR_PROVIDER` | `tesseract` | `tesseract` \| `easyocr` \| `paddleocr` \| `mock` |
| `TESSERACT_CMD` | `tesseract` | Path to Tesseract binary |
| `EMBEDDING_PROVIDER` | `mock` | `openai` \| `huggingface` \| `mock` |
| `OPENAI_API_KEY` | *(empty)* | Required when `EMBEDDING_PROVIDER=openai` |
| `OPENAI_EMBEDDING_MODEL` | `text-embedding-3-small` | OpenAI embedding model |
| `HUGGINGFACE_EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | HuggingFace model name |

> **⚠️ Security:** Always generate a new `SECRET_KEY` for production:
> ```bash
> openssl rand -hex 32
> ```

---

## 📡 API Reference

### Authentication

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/v1/auth/signup` | Register user; creates org if `org_name` provided |
| `POST` | `/api/v1/auth/login` | OAuth2 login; returns access + refresh tokens |
| `POST` | `/api/v1/auth/refresh` | Exchange refresh token for new token pair |

### Documents

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/v1/documents/upload` | Upload one or more files; triggers async ingestion |
| `GET` | `/api/v1/documents/` | List all documents in a project |
| `GET` | `/api/v1/documents/{id}/preview` | Stream/download original uploaded file |

### Projects & Tasks

| Method | Endpoint | Description |
|---|---|---|
| `GET/POST` | `/api/v1/projects/` | List / create projects |
| `GET` | `/api/v1/tasks/{task_id}` | Poll Celery task status |
| `GET` | `/api/v1/health` | Health check |

Full interactive docs are available at `/docs` (Swagger UI) or `/redoc`.

---

## 📄 Supported File Types

| Category | Extensions |
|---|---|
| Documents | `.pdf`, `.docx`, `.pptx` |
| Spreadsheets | `.xlsx`, `.xls`, `.csv` |
| Web & Email | `.html`, `.htm`, `.eml` |
| Plain Text | `.txt`, `.md`, `.markdown` |
| Images (OCR) | `.png`, `.jpg`, `.jpeg`, `.tiff`, `.bmp`, `.gif` |
| Archives | `.zip` (recursive extraction of supported types) |

---

## ✂️ Chunking Strategies

| Strategy | Description |
|---|---|
| `fixed_size` | Token-based sliding window using `cl100k_base` tokenizer (default: 500 tokens, 50 overlap) |
| `sentence` | Sentence-boundary aware chunking with token-counted sliding overlap |
| `markdown` | Splits on `#` headers — ideal for structured `.md` documents |

Configure per-upload via form fields: `chunk_strategy`, `chunk_size`, `chunk_overlap`.

---

## 🔌 Embedding Providers

| Provider | Model | Notes |
|---|---|---|
| `openai` | `text-embedding-3-small` (default) | Requires `OPENAI_API_KEY` |
| `huggingface` | `all-MiniLM-L6-v2` (default) | Requires `sentence-transformers` package |
| `mock` | Deterministic 1536-dim L2-normalized | For development/testing — no API key needed |

All embeddings are stored as 1536-dimensional vectors in PostgreSQL via `pgvector`.

---

## 🛡️ Multi-Tenancy & Security

- **Organization isolation** — every resource (project, document, chunk) is scoped to an `org_id`
- **RBAC** — `Admin` and member roles embedded in JWT claims for stateless authorization
- **Duplicate detection** — SHA-256 content hashing prevents re-processing identical files
- **Document versioning** — automatic version increment when a file with the same name is re-uploaded to the same project
- **Audit logging** — all significant actions recorded in `AuditLog`

---

## 🧪 Development

### Mock Mode (no external services needed)

Set in `.env`:

```env
EMBEDDING_PROVIDER=mock
OCR_PROVIDER=mock
```

This lets you develop and test the full ingestion pipeline without an OpenAI key or Tesseract installation.

### Running with Auto-reload

```bash
# Terminal 1 — API
uvicorn app.main:app --reload

# Terminal 2 — Worker
celery -A app.core.celery_app worker --loglevel=debug --concurrency=2
```

### Database Migrations

```bash
# Create a new migration after model changes
alembic revision --autogenerate -m "describe your change"

# Apply migrations
alembic upgrade head

# Rollback one step
alembic downgrade -1
```

---

## 🗃️ Tech Stack

| Layer | Technology |
|---|---|
| API Framework | FastAPI 0.111 + Uvicorn |
| ORM | SQLAlchemy 2.0 (async) + asyncpg |
| Database | PostgreSQL 16 + pgvector extension |
| Task Queue | Celery 5.4 + Redis 7 |
| Auth | python-jose (JWT) + passlib/bcrypt |
| PDF Parsing | PyMuPDF (fitz) |
| Office Docs | python-docx, python-pptx, openpyxl |
| OCR | pytesseract / EasyOCR / PaddleOCR |
| Embeddings | OpenAI SDK / sentence-transformers |
| Tokenizer | tiktoken (cl100k_base) |
| Migrations | Alembic |
| Config | pydantic-settings |

---

## 📜 License

This project is licensed under the **MIT License**.

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit your changes: `git commit -m "feat: add your feature"`
4. Push to the branch: `git push origin feature/your-feature`
5. Open a Pull Request
