
# Image Processing & Understanding  
**R&D by Debojyoti Chakraborty**

---

## Goals

### Requirement 1
User uploads an image → system processes → understands → answers the user.

### Requirement 2
Existing images → vectorized → stored → searchable (RAG / dedup).

---

## 1. High-Level Architecture

```mermaid
---
config:
  layout: fixed
---
flowchart TB
    U["User / App"] -- Upload Image + Question --> API["FastAPI API Gateway"]
    API --> VAL["Validation &amp; Security<br>(file type, size, EXIF, virus scan optional)"]
    VAL --> OBJ[("Object Storage<br>S3/MinIO/Azure Blob")] & Q["Job Queue<br>Redis/Kafka/SQS"]
    Q --> W["Processing Worker"]
    W --> PRE["Preprocessing<br>rotate, resize, denoise, deskew"]
    PRE --> OCR["OCR Extractor"] & CAP["Caption/Tags Extractor"] & EMB["Image Embedding Extractor"]
    OCR --> META[("Postgres Metadata")] & LLM["Answer Generator<br>LLM or Vision LLM"]
    CAP --> META & LLM
    EMB --> VDB[("Vector DB<br>pgvector/Milvus/Pinecone/Weaviate")]
    API -- Optional --> RETR["Retriever"]
    RETR --> VDB & META & LLM
    LLM --> API
    API -- Answer + Evidence --> U
```

**Why this works**
- Raw image stored once
- Everything else is derived
- Components replaceable independently
- Enterprise‑safe and auditable

---

## 2. Use Case A — User Image → Answer

### 2.1 Processing Pipeline

```mermaid
sequenceDiagram
    participant User
    participant API
    participant Storage
    participant Worker
    participant LLM

    User->>API: Upload image
    API->>Storage: Save raw image
    API->>Worker: Trigger processing
    Worker->>Worker: Preprocess + OCR + Caption
    Worker->>Worker: Generate embeddings
    Worker->>LLM: Send context + query
    LLM->>User: Answer with evidence
```

### 2.2 Processing Stages

| Stage | Action | Output |
|-----|------|------|
| Validate | Type, size, EXIF | Safe image |
| Preprocess | Denoise, deskew | Clean image |
| OCR | Text extraction | Text + layout |
| Caption | Semantic summary | Tags + caption |
| Embed | Vectorization | Image vector |
| Retrieve | Similarity search | Context |
| Answer | LLM reasoning | Final answer |

---

## 3. OCR vs Vision LLM

```mermaid
graph TD
    A[Input Image] --> B{Contains Text?}
    B -- Yes --> C[OCR]
    B -- No --> D[Vision LLM]
    C --> E[LLM Reasoning]
    D --> E
    E --> F[Answer]
```

---

## 4. Use Case B — Batch Image Indexing

```mermaid
flowchart LR
    I[Image Repository] --> P[Preprocess]
    P --> C[Caption]
    C --> E[Embedding]
    E --> DB[(Vector DB)]
```

### Stored Metadata

| Field | Purpose |
|----|----|
| image_id | Unique ID |
| storage_uri | Object path |
| image_embedding | Similarity |
| caption | Semantics |
| caption_embedding | Text search |
| ocr_text | Documents |
| tags | JSON metadata |
| tenant_id | Multi‑tenant |

---

## 5. Tooling Landscape

### OCR
- Tesseract
- EasyOCR
- Google Document AI
- AWS Textract

### Vision & Captioning
- Gemini Vision
- OpenAI Vision
- Open‑source caption models

### Embeddings
- CLIP / OpenCLIP
- Azure Vision Embeddings

### Vector Databases
- pgvector (Postgres)
- Pinecone
- Weaviate
- Milvus

---

## 6. Cost Model

```mermaid
flowchart TB
    OCR --> Cost
    VLM --> Cost
    Embeddings --> Cost
    Storage --> Cost
    Compute --> Cost
```

**Cost Control Levers**
- Cache OCR results
- Embed once, reuse forever
- Call Vision LLM only when needed

---

## 7. V1 Architecture Blueprints

### Low‑Cost / Self‑Hosted

```mermaid
graph LR
    FastAPI --> OpenCV
    OpenCV --> Tesseract
    Tesseract --> OpenCLIP
    OpenCLIP --> Postgres
```

### Enterprise Balanced

```mermaid
graph LR
    FastAPI --> CloudOCR
    CloudOCR --> VisionLLM
    VisionLLM --> LocalEmbedding
    LocalEmbedding --> Postgres
```

---

## 8. Deliverables

- `/upload-image`
- Async processing workers
- `/ask-image`
- `/search-image`
- Logs, retries, metrics

---

**Status:** Production‑ready documentation
