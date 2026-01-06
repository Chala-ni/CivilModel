# Civil Case Structure & Metadata Extractor

A production-grade FastAPI backend for extracting, structuring, and analyzing Sri Lankan Supreme Court civil case PDFs with support for English and Sinhala languages.

## 🎯 Purpose

This system converts civil case PDFs (judgments, petitions, orders) into structured, machine-readable hierarchical JSON format. It handles:
- Mixed English/Sinhala text extraction using Tesseract OCR
- Intelligent section and clause detection
- Hierarchical document structure (Metadata → Sections → Clauses → Footnotes)
- LLM-powered entity extraction and metadata analysis

## 🏗️ Architecture Overview

```
CivilModel/
├── app/
│   ├── api/
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── endpoints/
│   │       │   ├── __init__.py
│   │       │   ├── upload.py          # PDF upload endpoint
│   │       │   ├── process.py         # Document processing endpoint
│   │       │   └── retrieve.py        # Retrieve structured data
│   │       └── router.py              # API router configuration
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py                  # Environment variables, paths, settings
│   │   ├── logging.py                 # Logging configuration
│   │   └── dependencies.py            # FastAPI dependencies
│   ├── services/
│   │   ├── __init__.py
│   │   ├── ocr_service.py             # Tesseract integration (eng/sin)
│   │   ├── llm_service.py             # Ollama/Qwen2.5 interface
│   │   ├── parser_service.py          # JSON structure enforcement
│   │   ├── document_service.py        # Document processing orchestration
│   │   └── storage_service.py         # SQLite/JSON storage
│   ├── models/
│   │   ├── __init__.py
│   │   └── database.py                # SQLite models
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── document.py                # Pydantic schemas for documents
│   │   ├── section.py                 # Section schemas
│   │   ├── clause.py                  # Clause schemas
│   │   └── metadata.py                # Metadata schemas
│   ├── db/
│   │   ├── __init__.py
│   │   └── session.py                 # Database session management
│   └── main.py                        # FastAPI application entry point
├── scripts/
│   ├── prepare_training_data.py       # Convert PDFs to JSONL for fine-tuning
│   ├── test_ocr.py                    # Test Tesseract with sample PDFs
│   └── test_ollama.py                 # Test Ollama connection
├── tests/
│   ├── __init__.py
│   ├── test_ocr_service.py
│   ├── test_llm_service.py
│   └── test_api.py
├── data/
│   ├── sample_cases/                  # Sample Supreme Court PDFs
│   ├── uploads/                       # User-uploaded PDFs
│   ├── processed/                     # Processed JSON outputs
│   └── training/                      # Training data for fine-tuning
├── .env.example                       # Environment variables template
├── .gitignore
├── requirements.txt
└── README.md
```

## 🔧 Technology Stack

### Backend
- **Python 3.10+**: Core language
- **FastAPI**: High-performance async API framework
- **Pydantic v2**: Data validation and serialization
- **SQLite**: Lightweight database (production can migrate to PostgreSQL)

### OCR & Document Processing
- **Tesseract 5**: OCR engine with `eng` and `sin` language support
- **pdf2image**: PDF to image conversion
- **pytesseract**: Python wrapper for Tesseract
- **Pillow**: Image processing

### LLM Integration (Two Options)

**Option A - Cloud API (Easy)**:
- **OpenRouter**: Multi-provider API gateway
- **DeepSeek Chat**: Powerful, cheap model ($0.14/1M tokens)

**Option B - Local GPU (Advanced)**:
- **Qwen2.5-3B-Instruct**: Fine-tuned base model
- **LoRA Adapter**: Custom-trained for Sri Lankan legal documents
- **Ollama**: Local LLM inference server
- **PEFT**: For merging LoRA adapters

### Development Tools
- **uvicorn**: ASGI server
- **python-multipart**: File upload handling
- **pytest**: Testing framework

## 📋 Prerequisites

### 1. Install Tesseract OCR
**Windows:**
```bash
# Download installer from: https://github.com/UB-Mannheim/tesseract/wiki
# Install to default location: C:\Program Files\Tesseract-OCR
# Add to PATH: C:\Program Files\Tesseract-OCR
```

**macOS:**
```bash
brew install tesseract tesseract-lang
```

**Linux:**
```bash
sudo apt-get install tesseract-ocr tesseract-ocr-sin tesseract-ocr-eng
```

### 2. LLM Setup (Choose ONE Option)

You have **two options** for the LLM backend:

---

#### **OPTION A: OpenRouter API (Recommended - Easy Setup)**

Best for: Quick setup, no GPU required, cloud-based inference.

1. **Get API Key** (free tier available):
   - Go to: https://openrouter.ai/keys
   - Sign up and create an API key

2. **Configure `.env`**:
   ```env
   LLM_PROVIDER=openrouter
   OPENROUTER_API_KEY=sk-or-v1-your-key-here
   OPENROUTER_MODEL=deepseek/deepseek-chat
   ```

3. **That's it!** No GPU or local installation needed.

**Cost**: DeepSeek is very cheap (~$0.14 per million tokens)

---

#### **OPTION B: Ollama with Trained Model (Advanced - Local GPU)**

Best for: Running the fine-tuned model locally, offline inference, full control.

**Requirements**:
- GPU with 8GB+ VRAM (RTX 3060 or better)
- 20GB disk space for merged model
- Ollama installed from https://ollama.ai/download

**Step 1: Merge the LoRA Adapter**

The trained model is a LoRA adapter that needs to be merged with the base Qwen2.5-3B model:

```bash
# Install merge dependencies
pip install torch transformers peft

# Run the merge script (downloads base model + merges adapter)
python scripts/merge_adapter.py
```

This will:
- Download the base Qwen2.5-3B-Instruct model (~6GB)
- Merge it with the trained LoRA adapter
- Save the full model to `./merged_model/` (~6GB)

**Step 2: Import Merged Model into Ollama**

```bash
# Create the model in Ollama
ollama create civilmodel-qwen3b -f Modelfile

# Test it works
ollama run civilmodel-qwen3b "Extract metadata from a legal document"
```

**Step 3: Start Ollama Server**

Open a **new terminal** and keep it running:

```bash
# Start Ollama server (keep this running in background)
ollama serve
```

**Step 4: Configure `.env`**

```env
LLM_PROVIDER=ollama
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=civilmodel-qwen3b
```

**Step 5: Run Your FastAPI Server**

In **another terminal**:

```bash
cd D:\CivilModel
venv\Scripts\activate  # Windows
# source venv/bin/activate  # macOS/Linux

uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Now your API will use the **locally-trained model** instead of cloud APIs!

---

### 3. Python Environment
```bash
# Create virtual environment
python -m venv venv

# Activate
# Windows:
venv\Scripts\activate
# macOS/Linux:
source venv/bin/activate
```

## 🚀 Installation

1. **Clone the repository**
```bash
git clone <repository-url>
cd CivilModel
```

2. **Install Python dependencies**
```bash
pip install -r requirements.txt
```

3. **Configure environment variables**
```bash
cp .env.example .env
# Edit .env with your specific paths
```

4. **Verify Tesseract installation**
```bash
python scripts/test_ocr.py
```

5. **Verify Ollama connection**
```bash
python scripts/test_ollama.py
```

## 🎮 Usage

### Start the API Server
```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

API will be available at: `http://localhost:8000`
Interactive docs: `http://localhost:8000/docs`

### API Endpoints

#### 1. Upload PDF
```bash
POST /api/v1/upload
Content-Type: multipart/form-data

{
  "file": <PDF file>
}

Response: 
{
  "document_id": "uuid-string",
  "filename": "case_123.pdf",
  "status": "uploaded"
}
```

#### 2. Process Document
```bash
POST /api/v1/process/{document_id}

Response:
{
  "document_id": "uuid-string",
  "status": "processing",
  "message": "Document processing started"
}
```

#### 3. Retrieve Structured Data
```bash
GET /api/v1/documents/{document_id}

Response:
{
  "metadata": {
    "case_number": "SC/123/2023",
    "court": "Supreme Court of Sri Lanka",
    "date": "2023-12-15",
    "parties": ["Petitioner A", "Respondent B"]
  },
  "sections": [
    {
      "id": "intro",
      "title": "Introduction",
      "clauses": [
        {
          "id": "clause_1",
          "text": "...",
          "footnotes": []
        }
      ]
    }
  ]
}
```

## 🔬 Workflow

### Document Processing Pipeline

1. **Upload**: User uploads PDF via API
2. **OCR Extraction**: 
   - Convert PDF pages to images
   - Apply Tesseract with `eng+sin` languages
   - Clean text (remove headers/footers like "Page X of Y")
3. **Section Detection**: 
   - LLM identifies main sections (Introduction, Facts, Analysis, Decision)
   - Extracts section titles and boundaries
4. **Clause Parsing**:
   - Detect numbered clauses and sub-clauses
   - Link footnotes to parent clauses
5. **Hierarchical Structure**:
   - Build JSON tree: Metadata → Sections → Clauses → Footnotes
6. **Entity Extraction**:
   - Extract parties, court names, laws, dates
   - Store metadata separately
7. **Validation & Storage**:
   - Validate against Pydantic schemas
   - Store in SQLite + export JSON

## 🧪 Testing

Run the test suite:
```bash
pytest tests/ -v
```

Test individual components:
```bash
# Test OCR
pytest tests/test_ocr_service.py -v

# Test LLM
pytest tests/test_llm_service.py -v

# Test API
pytest tests/test_api.py -v
```

## 📊 Training Data Preparation

For fine-tuning Qwen2.5 with Unsloth on Google Colab:

```bash
python scripts/prepare_training_data.py \
  --input_dir data/sample_cases \
  --output_file data/training/civil_cases.jsonl
```

Output format (JSONL):
```json
{
  "instruction": "Extract sections and clauses from this Sri Lankan Supreme Court judgment",
  "input": "<OCR text>",
  "output": "<Structured JSON>"
}
```

Upload `civil_cases.jsonl` to Colab for fine-tuning.

## 🔒 Data Privacy

- All processing happens **locally** (no external API calls except Ollama)
- Uploaded PDFs stored in `data/uploads/` (gitignored)
- Add sensitive cases to `.gitignore` patterns

## 🛠️ Configuration

Key settings in `.env`:
```env
# Application
APP_NAME=CivilCaseExtractor
DEBUG=True
LOG_LEVEL=INFO

# Tesseract
TESSERACT_PATH=/usr/bin/tesseract
TESSERACT_LANGS=eng+sin

# Ollama
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=qwen2.5:7b-instruct
OLLAMA_TIMEOUT=120

# Database
DATABASE_URL=sqlite:///./data/civil_cases.db

# Directories
UPLOAD_DIR=./data/uploads
PROCESSED_DIR=./data/processed
```

## 📈 Performance Considerations

| Operation | OpenRouter | Ollama (Local) |
|-----------|------------|----------------|
| **OCR Extraction** | ~30 seconds (10 pages) | ~30 seconds (10 pages) |
| **LLM Processing** | ~20-30 seconds | ~10-20 seconds (with GPU) |
| **Total Time** | ~1 minute | ~40-50 seconds |

### Optimization Tips
- **OpenRouter**: Use DeepSeek for cost-effectiveness ($0.14/1M tokens)
- **Ollama**: Requires GPU (RTX 3060+) for optimal speed
- Batch process multiple pages in parallel
- Cache OCR results to avoid reprocessing

---

## 🎓 About the Trained Model

This project includes a **custom fine-tuned Qwen2.5-3B model** trained specifically for Sri Lankan legal documents.

### Training Details
- **Base Model**: Qwen/Qwen2.5-3B-Instruct
- **Fine-tuning Method**: LoRA (Low-Rank Adaptation)
- **Training Data**: 60 annotated Supreme Court judgments
- **Adapter Size**: ~130MB (full merged model: ~6GB)
- **Training Loss**: 1.8477
- **Training Framework**: Unsloth (Kaggle notebook)

### Why Use the Trained Model?

The trained model has been specifically optimized for:
- Extracting metadata from Sri Lankan Supreme Court cases
- Understanding legal terminology in English and Sinhala
- Structuring judgments into logical sections
- Identifying parties, judges, and case numbers with high accuracy

### How to Use It

**Option 1: Use OpenRouter (Easier)**
- No local GPU needed
- Uses DeepSeek (general-purpose LLM)
- Still performs well with our master system prompt

**Option 2: Run Trained Model Locally (Better accuracy)**
- Requires GPU (8GB+ VRAM)
- Follow the steps in [Option B: Ollama with Trained Model](#option-b-ollama-with-trained-model-advanced---local-gpu)
- Best performance for Sri Lankan legal documents

---

### Optimization Tips
- Use GPU-accelerated Ollama for faster inference
- Batch process multiple pages in parallel
- Cache OCR results to avoid reprocessing

## 🔄 Future Enhancements

- [ ] Support for additional languages (Tamil)
- [ ] Integration with Knowledge Graph component
- [ ] PostgreSQL migration for production
- [ ] Real-time processing status via WebSockets
- [ ] PDF annotation export with highlighted clauses
- [ ] Batch processing API for multiple documents

## 📝 License

[MIT License]

## 🤝 Contributing

[Add contribution guidelines]

## 📧 Contact

[Add contact information]
