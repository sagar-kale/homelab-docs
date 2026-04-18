# NAS Knowledge Base — AI Document Chatbot

A RAG (Retrieval-Augmented Generation) chatbot that answers questions about any document stored on the NAS — PDFs, Word docs, text files, scanned images. Ask "what does my lease say about pets?" or "summarise my council tax bill" and it finds the answer.

---

## Architecture

### Ingestion Pipeline

```mermaid
graph LR
    NAS["NAS /Drive\nPDF · DOCX · TXT · MD"]
    N8N["n8n\nHourly schedule\nMAX 500 files/run"]
    PG["PostgreSQL\ningestion.files\nskip seen files"]
    GLMOCR["GLM-OCR\n0.9B vision model\n#1 on OmniDocBench"]
    TIKA["Apache Tika\nlatest-full\nTesseract fallback"]
    CHUNK["Chunker\n1500 chars · 300 overlap"]
    EMBED["nomic-embed-text\n768-dim vectors"]
    QDRANT["Qdrant\nknowledge_base\n~4500+ vectors"]

    NAS --> N8N
    N8N -->|track processed| PG
    N8N -->|scanned PDFs| GLMOCR
    N8N -->|standard PDFs + text| TIKA
    GLMOCR -->|fallback if fails| TIKA
    GLMOCR --> CHUNK
    TIKA --> CHUNK
    CHUNK --> EMBED
    EMBED --> QDRANT
```

### Chat Query Path

```mermaid
sequenceDiagram
    participant U as User
    participant N8N as n8n Chat Workflow
    participant Q as Qdrant
    participant LLM as Gemma4 26b (llama-swap)

    U->>N8N: "What does my lease say about pets?"
    N8N->>Q: Embed query → vector search\ntopK=5, score threshold ≥ 0.5
    Q-->>N8N: Relevant chunks + source filenames
    N8N->>LLM: System prompt + chunks + question
    LLM-->>N8N: Answer with source citations
    N8N-->>U: "Your lease (lease-2023.pdf) states..."
```

---

## Configuration

| Setting | Value | Why |
|---------|-------|-----|
| Chunk size | 1500 chars | Larger chunks = better semantic context vs default 1000 |
| Chunk overlap | 300 chars | Prevents mid-sentence splits losing context |
| Score threshold | 0.5 | Drops irrelevant matches — no forced answers from poor results |
| topK | 5 | Retrieves 5 best chunks per query |
| Vector dims | 768 | nomic-embed-text output size |
| Distance metric | Cosine | Standard for text embeddings |
| HNSW index threshold | 5000 | Index builds once collection exceeds 5000 points |

---

## OCR Pipeline

Standard PDF extraction fails on scanned documents (image-only PDFs). A separate OCR pipeline handles these:

1. **GLM-OCR** (primary) — 0.9B vision model, ranked #1 on OmniDocBench V1.5. Renders each PDF page to PNG (2× scale) and extracts text via vision inference
2. **Apache Tika** (fallback) — Tesseract OCR bundled in `tika:latest-full` image
3. Failed files tracked in Postgres with error type — retried by manual workflow run

---

## Stats

| Metric | Value |
|--------|-------|
| Total eligible files on NAS | ~640 |
| Successfully ingested | 533 |
| Qdrant vectors | ~4500+ (growing) |
| File types | PDF, DOCX, TXT, MD, HTML |
| Skipped (encrypted / tiny) | ~50 |

---

## n8n Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| NAS File Ingestion Pipeline | Hourly | Scan NAS, ingest new files into Qdrant |
| ingestion sub-workflow | Called by pipeline | Chunk → embed → upsert per file |
| NAS Knowledge Base Chat | On message | RAG query → LLM response |
| NAS OCR Ingestion | Manual | Re-process failed scanned PDFs via GLM-OCR |
