# Document RAG Pipeline

Data pipeline for ingesting, structuring, and querying unstructured documents - hybrid retrieval (vector + BM25) with cross-encoder reranking and Gemini for answer generation.

Upload a document and ask plain-English questions. The pipeline extracts and cleans text, classifies content by type, indexes it using hybrid vector and keyword search, and returns concise grounded answers.

## Overview

### Ingestion & Indexing

1. **Ingestion** - PDF uses a two-pass extract: fitz classifies each page (`native_text` / `image_dominant` / `table_heavy`), then routes to batched `pymupdf4llm`, parallel `unstructured`, or OCR. Other files use single-pass `unstructured` extract.
2. **Segmentation** - heuristic boundaries group PDF pages into logical segments
3. **Indexing** - segments are chunked, embedded (`all-MiniLM-L6-v2`), and stored in a session vector index; same segment + similar text indexed once (`dedupe_nodes`)

### Retrieval, Reranking & Generation

1. **Query rewrite** - Gemini normalizes typos before search (`QUERY_REWRITE=true`)
2. **Hybrid retrieval** - queries hit both semantic (vector top-K) and keyword (BM25 top-K) search; results are merged and deduplicated by page/segment key
3. **Reranking** - a cross-encoder (`ms-marco-MiniLM-L-6-v2`) scores and re-orders the top chunks
4. **Generation** - Gemini synthesizes a concise answer from the retrieved context only; citation lines deduplicated in output
5. **Session caching** - embeddings and BM25 index stay in memory until you exit the CLI

## Tech Stack

| Component | Tool |
|-----------|------|
| LLM | Google Gemini (via `google.genai`) |
| Embeddings | HuggingFace `sentence-transformers/all-MiniLM-L6-v2` |
| RAG Framework | LlamaIndex |
| PDF Extraction | PyMuPDF (`fitz`), `pymupdf4llm`, `unstructured`, Tesseract OCR |
| Other Formats | `unstructured.partition.auto` (docx, xlsx, pptx, html, txt, images) |
| Vector Search | LlamaIndex `VectorStoreIndex` |
| Keyword Search | BM25 Retriever |
| Reranking | `cross-encoder/ms-marco-MiniLM-L-6-v2` (SentenceTransformer) |
| Retrieval Strategy | Hybrid (vector + BM25) with reranking |

## Quick Start

1. Install dependencies:
   ```bash
   python lib.py
   ```

2. Create a `.env` file:
   ```
   GEMINI_API_KEY=your_key_here
   TESSERACT_CMD=C:\Program Files\Tesseract-OCR\tesseract.exe
   MAX_EXTRACT_WORKERS=4
   QUERY_REWRITE=true
   DATA_DIR=./data/rag
   ```

3. Install Tesseract OCR (required for scanned PDF pages and images):
   - Windows: `winget install UB-Mannheim.TesseractOCR`
   - Or download from: https://github.com/UB-Mannheim/tesseract/wiki
   - If not on PATH, set `TESSERACT_CMD` in `.env`

4. Run the CLI:
   ```bash
   python rag_pipeline.py
   ```
   Enter a file path when prompted, then ask questions.

   Or run the Streamlit UI:
   ```bash
   streamlit run app.py
   ```

### Inspect ingest output

To see what the pipeline extracts for PDFs (1 text page, 3 table pages, 3 scanned/OCR pages):
```bash
python "tests/test ingestion/test_ingest_sample.py" "your.pdf" --full -o ingest_sample_report.txt
```
Requires `TESSERACT_CMD` in `.env` for `image_dominant` pages.

## Notes

Requires a Gemini API key - get one at [Google AI Studio](https://aistudio.google.com). Tesseract must be installed separately for scanned PDF pages. 

Supported uploads: PDF (two-pass page pipeline), plus Word, Excel, PowerPoint, HTML, plain text, CSV, and images via `unstructured` - most common office formats work out of the box.
