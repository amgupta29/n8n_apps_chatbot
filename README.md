# API RAG Document QA Bot

An n8n workflow that exposes a REST API for uploading documents and querying them with an AI assistant powered by Google Gemini and Qdrant vector search.

## Overview

This workflow provides two HTTP endpoints:

| Endpoint | Method | Purpose |
|---|---|---|
| `/rag-api/upload` | POST | Upload and index a document |
| `/rag-api/chat` | POST | Ask questions about indexed documents |

Documents are chunked, embedded with `gemini-embedding-001`, and stored in a Qdrant collection scoped to a `session_id`. The chat agent uses semantic retrieval to answer questions strictly based on the uploaded content.

## Architecture

```
Upload flow:
  POST /rag-api/upload
    → Validate input
    → Extract text via Gemini 2.0 Flash (multimodal)
    → Split into 1000-char chunks (100-char overlap)
    → Embed each chunk via gemini-embedding-001
    → Upsert all points into Qdrant
    → Return indexing summary

Chat flow:
  POST /rag-api/chat
    → Validate input
    → AI Agent (Gemini 2.0 Flash)
        ├── retrieve_documents  — semantic search in Qdrant
        ├── list_documents      — list documents for the session
        └── delete_document     — remove a document from the knowledge base
    → Return AI response
```

## Tech Stack

- **Orchestration**: [n8n](https://n8n.io)
- **LLM**: Google Gemini 2.0 Flash
- **Embeddings**: Google Gemini Embedding 001 (768-dim)
- **Vector DB**: [Qdrant](https://qdrant.tech)

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `GEMINI_API_KEY` | Yes | — | Google AI Studio API key |
| `QDRANT_URL` | No | `http://localhost:6333` | Qdrant instance URL |
| `QDRANT_API_KEY` | No | _(empty)_ | Qdrant API key (for cloud deployments) |
| `QDRANT_COLLECTION` | No | `test_bot_knowledge` | Qdrant collection name |

## API Reference

### POST `/rag-api/upload`

Upload and index a document into the session's knowledge base.

**Request body (JSON)**

```json
{
  "session_id": "user-123",
  "file_base64": "<base64-encoded file content>",
  "mime_type": "application/pdf",
  "file_name": "report.pdf"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `session_id` | string | Yes | Unique identifier for the user/session |
| `file_base64` | string | Yes | Base64-encoded file content |
| `mime_type` | string | No | MIME type of the file (default: `application/pdf`) |
| `file_name` | string | No | Original file name (used as document label) |

**Supported MIME types**

| MIME type | Format |
|---|---|
| `application/pdf` | PDF |
| `text/plain` | Plain text |
| `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | DOCX |
| `application/msword` | DOC |
| `application/vnd.openxmlformats-officedocument.presentationml.presentation` | PPTX |
| `application/vnd.ms-powerpoint` | PPT |

**Response (200)**

```json
{
  "status": "indexed",
  "session_id": "user-123",
  "document_name": "report",
  "doc_type": "PDF",
  "chunks_inserted": 42,
  "message": "Successfully indexed \"report\" (PDF) with 42 chunks.",
  "timestamp": "2026-03-22T10:00:00.000Z"
}
```

---

### POST `/rag-api/chat`

Send a message and receive an AI-generated answer based on documents indexed for the session.

**Request body (JSON)**

```json
{
  "session_id": "user-123",
  "message": "What are the main conclusions of the report?"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `session_id` | string | Yes | Same session identifier used during upload |
| `message` | string | Yes | User's question or instruction |

**Response (200)**

```json
{
  "session_id": "user-123",
  "response": "According to [Source: report], the main conclusions are ...",
  "timestamp": "2026-03-22T10:01:00.000Z"
}
```

**Agent capabilities via natural language**

| User intent | What the agent does |
|---|---|
| Ask a question about a document | Calls `retrieve_documents` and answers based on returned chunks |
| "List my documents" | Calls `list_documents` and returns indexed document names |
| "Delete the report document" | Calls `delete_document` to remove it from Qdrant |

The agent maintains a **10-message conversation window** per `session_id`.

---

## Quick Start

### 1. Prerequisites

- n8n instance running (self-hosted or cloud)
- Qdrant instance running and a collection created with **768-dimensional** vectors
- Google AI Studio API key

### 2. Create the Qdrant collection

```bash
curl -X PUT http://localhost:6333/collections/test_bot_knowledge \
  -H 'Content-Type: application/json' \
  -d '{
    "vectors": {
      "size": 768,
      "distance": "Cosine"
    }
  }'
```

### 3. Import the workflow

1. Open your n8n instance.
2. Go to **Workflows → Import from file**.
3. Select `API_RAG_Document_QA_Bot.json`.
4. Set the `GEMINI_API_KEY` environment variable (and optionally `QDRANT_URL`, `QDRANT_API_KEY`, `QDRANT_COLLECTION`) in your n8n environment.
5. Activate the workflow.

### 4. Upload a document

```bash
# Encode the file
FILE_B64=$(base64 -i report.pdf)

curl -X POST https://<your-n8n-host>/webhook/rag-api/upload \
  -H 'Content-Type: application/json' \
  -d '{
    "session_id": "user-123",
    "file_base64": "'"$FILE_B64"'",
    "mime_type": "application/pdf",
    "file_name": "report.pdf"
  }'
```

### 5. Chat with the document

```bash
curl -X POST https://<your-n8n-host>/webhook/rag-api/chat \
  -H 'Content-Type: application/json' \
  -d '{
    "session_id": "user-123",
    "message": "Summarise the key findings."
  }'
```

## CORS

Both endpoints respond with the following headers, making them safe to call from browser-based frontends:

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: Content-Type, Authorization
```

## Notes

- The `session_id` acts as a **namespace** in the vector store. Documents uploaded under one `session_id` are only visible to chat requests using the same `session_id`.
- Text extraction is capped at **50,000 characters** per document.
- The agent is instructed to answer **only** from retrieved document content and will not hallucinate information not present in the knowledge base.
- The score threshold for retrieval is **0.3** (cosine similarity); results below this threshold are excluded.
