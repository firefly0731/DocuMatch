# DocuMatch

**Audit Evidence Automation Tool** - Extract structured data from audit evidence documents (PDF & Word) and produce Excel outputs with single-cell summary sentences.

## Architecture

```
DocuMatch/
├── frontend/          # Next.js 14+ (App Router, TypeScript, Tailwind, shadcn/ui)
├── backend/           # Python FastAPI + LangGraph + LangChain
├── PLAN.md            # Detailed implementation plan
└── README.md
```

## Core Workflow

### 1. Plan Mode (Interactive Calibration)
- Upload ONE sample document
- AI Agent (LangGraph) analyzes and proposes extraction schema
- Refine schema through chat-based feedback
- Approve and save schema for batch processing

### 2. Batch Mode (High-Volume Processing)
- Upload 100+ documents (PDF & DOCX mixed)
- Apply approved schema with Anthropic Prompt Caching (~90% cost savings)
- Review results in split-view grid
- Export to Excel

## Tech Stack

### Backend
- **Framework:** FastAPI + Pydantic V2
- **AI Orchestration:** LangGraph (Plan Mode), LangChain LCEL (Batch Mode)
- **LLM:** Claude 3.5 Sonnet (claude-3-5-sonnet-20241022)
- **Document Processing:** pypdf, python-docx, pdf2image
- **Database:** SQLite (aiosqlite)
- **Excel Export:** openpyxl

### Frontend
- **Framework:** Next.js 14+ (App Router)
- **Language:** TypeScript
- **Styling:** Tailwind CSS + shadcn/ui
- **Data Grid:** TanStack Table
- **Split View:** react-resizable-panels

## Quick Start

### Prerequisites
- Python 3.11+
- Node.js 18+
- Poetry (Python package manager)
- Poppler (for PDF to image conversion)

### Backend Setup

```bash
cd backend
poetry install
cp ../.env.example .env
# Edit .env and add your ANTHROPIC_API_KEY
poetry run uvicorn app.main:app --reload
```

Backend will be available at:
- API: http://localhost:8000
- Docs: http://localhost:8000/docs
- Health: http://localhost:8000/health

### Frontend Setup

```bash
cd frontend
npm install
npm run dev
```

Frontend will be available at http://localhost:3000

## API Endpoints

### Health & Status
- `GET /health` - Health check
- `GET /api/v1/status` - Detailed status

### Plan Mode
- `POST /api/v1/plan/start` - Start new planning session
- `POST /api/v1/plan/feedback` - Submit feedback/approval
- `POST /api/v1/plan/test-extraction` - Test current schema
- `GET /api/v1/plan/specs` - List saved specs
- `GET /api/v1/plan/specs/{spec_id}` - Get specific spec

### Batch Mode
- `POST /api/v1/batch/start` - Start batch processing
- `GET /api/v1/batch/jobs/{job_id}/status` - Get job status
- `GET /api/v1/batch/jobs/{job_id}/results` - Get extraction results
- `POST /api/v1/batch/jobs/{job_id}/retry` - Retry failed documents
- `GET /api/v1/batch/jobs/{job_id}/export` - Export to Excel

## Environment Variables

See `.env.example` for required configuration:

```
ANTHROPIC_API_KEY=your-api-key-here
DEBUG=false
```

## Development

### Running Tests

```bash
# Backend
cd backend
poetry run pytest

# Frontend
cd frontend
npm test
```

### Code Formatting

```bash
# Backend (ruff)
cd backend
poetry run ruff check --fix .
poetry run ruff format .

# Frontend (prettier)
cd frontend
npm run format
```

## License

MIT
