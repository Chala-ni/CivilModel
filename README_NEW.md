# Civil Case Structure & Metadata Extractor

A production-grade FastAPI backend for extracting, structuring, and analyzing Sri Lankan Supreme Court civil case PDFs with support for English and Sinhala languages.

## 🎯 Project Overview

This system converts civil case PDFs (judgments, petitions, orders) into structured, machine-readable hierarchical JSON format.

### Features
- ✅ **Hybrid OCR**: Fast pdfplumber extraction + Tesseract fallback for Sinhala text
- ✅ **LLM-Powered Extraction**: Uses DeepSeek (via OpenRouter) for intelligent metadata extraction
- ✅ **Fine-Tuned Model Support**: Can run custom-trained Qwen2.5-3B model locally via Ollama
- ✅ **RESTful API**: Clean endpoints for frontend integration
- ✅ **Bilingual Support**: English + Sinhala (සිංහල) text extraction

---

## 🏗️ Architecture

```
CivilModel/
├── app/
│   ├── api/v1/endpoints/     # API endpoints
│   │   ├── upload.py         # POST /upload
│   │   ├── process.py        # POST /process/{doc_id}
│   │   └── retrieve.py       # GET /documents/*
│   ├── services/
│   │   ├── ocr_service.py    # Hybrid OCR (pdfplumber + Tesseract)
│   │   ├── llm_service.py    # OpenRouter/Ollama integration
│   │   └── parser_service.py # JSON parsing & validation
│   ├── core/config.py        # Settings & environment
│   ├── schemas/models.py     # Pydantic schemas
│   └── main.py               # FastAPI app
├── scripts/
│   └── merge_adapter.py      # Merge LoRA adapter for local deployment
├── Modelfile                 # Ollama model template
├── requirements.txt
└── .env.example
```

---

## 🚀 Quick Start

### Prerequisites

1. **Python 3.10+**
2. **Tesseract OCR** with Sinhala language pack

#### Install Tesseract

**Windows:**
```bash
# Download from: https://github.com/UB-Mannheim/tesseract/wiki
# During install, select "Additional language data" → check "sin" (Sinhala)
# Default path: C:\Program Files\Tesseract-OCR\tesseract.exe
```

**macOS:**
```bash
brew install tesseract tesseract-lang
```

**Linux:**
```bash
sudo apt-get install tesseract-ocr tesseract-ocr-sin tesseract-ocr-eng
```

### Installation

```bash
# 1. Clone repository
git clone <repository-url>
cd CivilModel

# 2. Create virtual environment
python -m venv venv

# Windows:
venv\Scripts\activate

# macOS/Linux:
source venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Configure environment
copy .env.example .env
# Edit .env with your API key
```

---

## 🔧 LLM Configuration

You have **TWO options** for the LLM backend:

### Option A: OpenRouter (Recommended - Easy Setup)

Best for: Quick setup, no GPU required, cloud-based inference.

1. **Get API Key** from https://openrouter.ai/keys
2. **Edit `.env`:**
   ```env
   LLM_PROVIDER=openrouter
   OPENROUTER_API_KEY=sk-or-v1-your-key-here
   OPENROUTER_MODEL=deepseek/deepseek-chat
   ```

**Cost:** ~$0.14 per million tokens (very cheap!)

---

### Option B: Ollama with Trained Model (Advanced)

Best for: Running the fine-tuned model locally, offline inference.

**Requirements:**
- GPU with 8GB+ VRAM (RTX 3060 or better)
- 20GB disk space for merged model

**Step 1: Merge the LoRA Adapter**
```bash
pip install torch transformers peft
python scripts/merge_adapter.py
```

**Step 2: Import into Ollama**
```bash
# Install Ollama from: https://ollama.ai/download
ollama create civilmodel-qwen3b -f Modelfile
ollama run civilmodel-qwen3b "Hello"
```

**Step 3: Edit `.env`:**
```env
LLM_PROVIDER=ollama
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=civilmodel-qwen3b
```

**Step 4: Start Ollama**
```bash
ollama serve
```

---

## 🎮 Running the Server

```bash
# Activate virtual environment
venv\Scripts\activate  # Windows
source venv/bin/activate  # macOS/Linux

# Start the server
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

**Server URLs:**
- API: http://localhost:8000
- Interactive Docs: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc

---

## 📡 API Endpoints

### Core Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/upload` | Upload a PDF document |
| `POST` | `/api/v1/process/{doc_id}` | Process uploaded document |
| `GET` | `/api/v1/documents/{doc_id}` | Get full document data |
| `GET` | `/api/v1/documents` | List all documents |

### Frontend-Friendly Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/documents/{doc_id}/metadata` | Get only metadata (lightweight) |
| `GET` | `/api/v1/documents/{doc_id}/sections` | Get only sections |
| `GET` | `/api/v1/documents-summary` | List documents with brief summaries |
| `GET` | `/api/v1/health` | Health check for monitoring |

---

## 📋 API Usage Examples

### 1. Upload a PDF

```bash
curl -X POST "http://localhost:8000/api/v1/upload" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@path/to/document.pdf"
```

**Response:**
```json
{
  "document_id": "doc_abc123",
  "filename": "document.pdf",
  "status": "uploaded",
  "message": "Document uploaded successfully"
}
```

### 2. Process the Document

```bash
curl -X POST "http://localhost:8000/api/v1/process/doc_abc123"
```

**Response:**
```json
{
  "document_id": "doc_abc123",
  "status": "processing",
  "message": "Document processing started"
}
```

### 3. Get Extracted Data

```bash
curl "http://localhost:8000/api/v1/documents/doc_abc123"
```

**Response:**
```json
{
  "document_id": "doc_abc123",
  "filename": "document.pdf",
  "status": "completed",
  "metadata": {
    "case_number": "SC Appeal No. 105/2012",
    "court": "Supreme Court of Sri Lanka",
    "date": "2025-11-07",
    "parties": ["Party 1", "Party 2"],
    "judges": ["Judge 1", "Judge 2", "Judge 3"],
    "case_type": "Appeal"
  },
  "sections": [
    {
      "title": "Header and Case Details",
      "content": "..."
    },
    {
      "title": "Judgment and Legal Analysis",
      "content": "..."
    },
    {
      "title": "Conclusion and Order",
      "content": "..."
    }
  ]
}
```

### 4. Get Only Metadata (Lightweight)

```bash
curl "http://localhost:8000/api/v1/documents/doc_abc123/metadata"
```

### 5. List All Documents

```bash
curl "http://localhost:8000/api/v1/documents-summary"
```

---

## 🧪 Testing

### Using PowerShell Script

```powershell
.\test_api.ps1
```

### Manual Testing

```powershell
# Upload
$response = Invoke-RestMethod -Uri "http://localhost:8000/api/v1/upload" `
  -Method Post -InFile ".\data\sample_cases\sc_appeal_105_2012.pdf" `
  -ContentType "multipart/form-data"

# Process
Invoke-RestMethod -Uri "http://localhost:8000/api/v1/process/$($response.document_id)" -Method Post

# Wait 30 seconds for processing...

# Retrieve
Invoke-RestMethod -Uri "http://localhost:8000/api/v1/documents/$($response.document_id)"
```

---

## 🐛 Troubleshooting

### Common Issues

#### 1. "Tesseract not found"
```
Solution: Install Tesseract and update TESSERACT_PATH in .env
Windows: C:\Program Files\Tesseract-OCR\tesseract.exe
```

#### 2. "OPENROUTER_API_KEY not set"
```
Solution: Get key from https://openrouter.ai/keys
Add to .env: OPENROUTER_API_KEY=sk-or-v1-xxx
```

#### 3. "Cannot connect to Ollama"
```
Solution: Start Ollama server
Run: ollama serve
```

#### 4. "Model not found" (Ollama)
```
Solution: Create the model first
Run: ollama create civilmodel-qwen3b -f Modelfile
```

#### 5. "Rate limit exceeded" (OpenRouter)
```
Solution: Wait a few seconds and retry
The system automatically retries 3 times
```

#### 6. Sinhala text appears as garbage
```
Solution: Ensure Tesseract has Sinhala language pack
Windows: Reinstall with language pack selected
Linux: sudo apt-get install tesseract-ocr-sin
```

### Debug Mode

Enable detailed logging:
```env
DEBUG=True
LOG_LEVEL=DEBUG
```

Check logs:
```bash
# Logs are printed to console and saved to logs/app.log
```

---

## 📊 Processing Pipeline

```
1. UPLOAD
   └── PDF saved to data/uploads/

2. OCR EXTRACTION
   ├── pdfplumber (fast, for clean text)
   └── Tesseract (fallback, for Sinhala/scanned pages)

3. LLM EXTRACTION
   ├── OpenRouter API (cloud)
   └── OR Ollama (local)

4. JSON PARSING
   └── Validate and structure response

5. STORAGE
   └── Save to in-memory database
```

---

## 📁 Output Format

```json
{
  "metadata": {
    "case_number": "SC Appeal No. X/YYYY",
    "court": "Supreme Court of Sri Lanka",
    "date": "YYYY-MM-DD",
    "parties": ["Party 1", "Party 2"],
    "judges": ["Judge 1", "Judge 2"],
    "case_type": "Appeal"
  },
  "sections": [
    {
      "title": "Header and Case Details",
      "content": "Full text of this section..."
    },
    {
      "title": "Judgment and Legal Analysis", 
      "content": "Full text of this section..."
    },
    {
      "title": "Conclusion and Order",
      "content": "Full text of this section..."
    }
  ]
}
```

---

## 🔒 CORS Configuration

For frontend integration, CORS is enabled. Edit `.env`:

```env
CORS_ORIGINS=["http://localhost:3000", "http://localhost:5173"]
```

---

## 📈 Performance

| Operation | Time |
|-----------|------|
| OCR (10 pages, hybrid) | ~30 seconds |
| LLM Extraction (OpenRouter) | ~20-30 seconds |
| Total Processing | ~1 minute |

---

## 🤝 About the Trained Model

The project includes a fine-tuned **Qwen2.5-3B** model trained on Sri Lankan legal documents:

- **Training Data**: 60 annotated Supreme Court judgments
- **Fine-tuning Method**: LoRA (Low-Rank Adaptation)
- **Adapter Size**: ~130MB
- **Final Loss**: 1.8477

To use the trained model:
1. Run `scripts/merge_adapter.py` to create full model
2. Import into Ollama using the provided `Modelfile`
3. Set `LLM_PROVIDER=ollama` in `.env`

---

## 📝 License

[MIT License]

---

## 👥 Contributors

- Your Name - Initial Development

---

## 📧 Support

For issues or questions, please open a GitHub issue.
