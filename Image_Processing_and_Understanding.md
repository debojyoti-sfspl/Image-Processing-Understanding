
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
flowchart TB
  U[User / Client] -->|"Upload Image + Question"| API[API Gateway\nFastAPI]
  API --> VAL[Validation\nType / Size / EXIF / Security]
  VAL --> OBJ[(Object Storage\nS3 / MinIO / Azure Blob)]

  VAL --> Q[Queue\nRedis / Kafka / SQS]
  Q --> W[Processing Worker]

  W --> PRE[Preprocessing\nRotate / Resize / Denoise / Deskew]
  PRE --> OCR[OCR Extractor]
  PRE --> CAP[Caption / Tags Extractor]
  PRE --> EMB[Embedding Extractor]

  OCR --> META[(Metadata DB\nPostgres)]
  CAP --> META
  EMB --> VDB[(Vector DB\npgvector / Milvus / Weaviate)]

  API --> RETR[Retriever]
  RETR --> VDB
  RETR --> META

  RETR --> GEN[Answer Generator\nLLM / Vision LLM]
  OCR --> GEN
  CAP --> GEN

  GEN --> API -->|"Answer + Evidence"| U
```


## 2. Use Case A — User Image → Answer

### 2.1 Sequence diagram (runtime)

```mermaid
sequenceDiagram
  participant User
  participant API as FastAPI
  participant Store as Object Storage
  participant Q as Queue
  participant W as Worker
  participant DB as Postgres
  participant VDB as VectorDB
  participant LLM as LLM/VLM

  User->>API: Upload image + question
  API->>API: Validate (type/size/EXIF)
  API->>Store: Save raw image
  API->>Q: Enqueue processing job (image_id)

  Q->>W: Job
  W->>W: Preprocess image
  W->>W: OCR + Caption + Embedding
  W->>DB: Save text/caption/metadata
  W->>VDB: Upsert embedding vector

  API->>VDB: Retrieve similar items (optional)
  API->>DB: Fetch OCR/caption evidence
  API->>LLM: Ask with evidence + retrieval
  LLM->>API: Answer + citations/evidence
  API->>User: Final response

```


## 3. Use Case B — Existing Images → Vectorize (Batch indexing)
### 3.1 Batch pipeline

```mermaid
flowchart LR
  SRC[Image Sources\nFolder/S3/Drive] --> PRE[Batch Preprocess]
  PRE --> EMB[Embed Generator\nCLIP/OpenCLIP]
  PRE --> CAP[Optional Captioning]
  EMB --> VDB[(Vector DB)]
  CAP --> DB[(Postgres Metadata)]
  SRC --> OBJ[(Object Storage)]
  DB --> API[Search API]
  VDB --> API

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

## 5. Cost Breakdown
### 5.3 A practical cost formula

```mermaid
flowchart TD
    Monthly_Cost["Monthly Cost ≈"]
    OCR["OCR_calls × OCR_unit_price"]
    VLM["VLM_calls × avg_tokens × token_price"]
    Embeddings["Embeddings_generated × embedding_unit_price_or_compute"]
    VectorDB["VectorDB_storage + VectorDB_queries"]
    Storage["Object_storage_GB_month"]
    Compute["Compute_CPU/GPU_hours"]
    
    Monthly_Cost --> OCR
    Monthly_Cost --> VLM
    Monthly_Cost --> Embeddings
    Monthly_Cost --> VectorDB
    Monthly_Cost --> Storage
    Monthly_Cost --> Compute
```
