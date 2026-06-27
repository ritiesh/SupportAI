# SupportAI — Intelligent Customer Support Chatbot

> A Spring Boot + Spring AI + RAG application that lets companies upload their support documents and gives customers an AI-powered chat interface that answers questions grounded strictly in that knowledge base — no hallucinations, with source citations and full conversation memory.

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Solution Overview](#solution-overview)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [RAG Pipeline](#rag-pipeline)
- [Project Structure](#project-structure)
- [Database Schema](#database-schema)
- [API Endpoints](#api-endpoints)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Key Spring AI Components](#key-spring-ai-components)
- [Features](#features)
- [Future Enhancements](#future-enhancements)

---

## Problem Statement

Support teams waste hours answering the same repetitive questions. Customers wait too long for responses. Companies have knowledge spread across FAQs, product manuals, and policy documents — but no smart way to query it in real time.

Traditional rule-based chatbots fail because they match keywords. Ask *"how do I get my money back?"* and it won't find the answer filed under *"refund policy."*

**SupportAI** solves this with semantic understanding — it finds meaning, not just words.

---

## Solution Overview

- An **Admin** uploads support documents (PDFs, DOCX, TXT) into the knowledge base
- Documents are automatically **chunked, embedded, and stored** as vectors in PostgreSQL
- A **Customer** opens a chat session and asks questions in natural language
- The system performs **semantic vector search** to find the most relevant document chunks
- Those chunks are passed as context to **Claude AI**, which generates a grounded, cited answer
- **Conversation memory** is maintained per session so follow-up questions work naturally
- When the AI cannot answer, it **escalates to a support ticket** for a human agent

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Spring Boot 3.x |
| AI Orchestration | Spring AI 1.1.0 |
| LLM | Anthropic Claude (`claude-sonnet-4-6`) |
| Embeddings | Spring AI `EmbeddingModel` (OpenAI `text-embedding-3-small`) |
| Vector Store | PostgreSQL + `pgvector` extension (HNSW index) |
| Relational DB | PostgreSQL 16 (JPA / Hibernate) |
| Document Parsing | Spring AI `TikaDocumentReader` (PDF, DOCX, TXT) |
| Chat Memory | Spring AI `JdbcChatMemory` (persisted in PostgreSQL) |
| Build Tool | Maven |
| Language | Java 21 |
| Containerisation | Docker + Docker Compose |

---

## Architecture

```
┌─────────────────────┐        ┌──────────────────────────────────────────┐
│     Admin Portal    │        │              Spring Boot App              │
│                     │        │                                           │
│  Upload documents   │───────▶│  DocumentIngestionService                 │
│  (PDF, DOCX, TXT)   │        │    → TikaDocumentReader                   │
│                     │        │    → TokenTextSplitter                    │
│  Manage knowledge   │        │    → EmbeddingModel                       │
│  base               │        │    → VectorStore (pgvector)               │
└─────────────────────┘        │                                           │
                                │  SupportChatService                      │
┌─────────────────────┐        │    → QuestionAnswerAdvisor (RAG)          │
│    Customer Chat    │        │    → MessageChatMemoryAdvisor             │
│                     │        │    → ChatClient → Claude AI               │
│  Ask a question     │───────▶│                                           │
│  View chat history  │        │  TicketService                            │
│  Raise a ticket     │        │    → Escalation to human agents           │
└─────────────────────┘        └──────────────────────────────────────────┘
                                                    │
                                    ┌───────────────┴───────────────┐
                                    │         PostgreSQL 16          │
                                    │                               │
                                    │  Relational tables (JPA)      │
                                    │  knowledge_documents          │
                                    │  chat_sessions                │
                                    │  chat_messages                │
                                    │  support_tickets              │
                                    │                               │
                                    │  pgvector extension           │
                                    │  vector_store (embeddings)    │
                                    └───────────────────────────────┘
```

---

## RAG Pipeline

### Phase 1 — Ingestion (one-time, triggered by admin upload)

```
PDF / DOCX / TXT
      │
      ▼
TikaDocumentReader        ← Parses file content
      │
      ▼
TokenTextSplitter         ← Breaks into ~500 token chunks
      │
      ▼
EmbeddingModel            ← Converts each chunk to a 1536-dim vector
      │
      ▼
VectorStore (pgvector)    ← Stored with HNSW index for fast similarity search
```

### Phase 2 — Query (every customer message)

```
User question
      │
      ▼
EmbeddingModel            ← Embeds the question into same vector space
      │
      ▼
VectorStore.search()      ← Top-K most similar chunks (cosine distance)
      │
      ▼
QuestionAnswerAdvisor     ← Injects chunks into prompt as context
      │
      ▼
MessageChatMemoryAdvisor  ← Adds conversation history to prompt
      │
      ▼
Claude AI                 ← Generates answer strictly from context
      │
      ▼
Answer + source document citation
```

---

## Project Structure

```
supportai/
├── src/
│   ├── main/
│   │   ├── java/com/supportai/
│   │   │   ├── SupportAiApplication.java
│   │   │   │
│   │   │   ├── config/
│   │   │   │   ├── AiConfig.java               ← ChatClient + VectorStore beans
│   │   │   │   └── SecurityConfig.java         ← Basic auth for admin endpoints
│   │   │   │
│   │   │   ├── controller/
│   │   │   │   ├── AdminController.java        ← Document upload & management
│   │   │   │   ├── ChatController.java         ← Customer chat endpoints
│   │   │   │   └── TicketController.java       ← Support ticket escalation
│   │   │   │
│   │   │   ├── service/
│   │   │   │   ├── DocumentIngestionService.java  ← Chunk + embed + store
│   │   │   │   ├── SupportChatService.java        ← RAG chat with memory
│   │   │   │   └── TicketService.java             ← Create & manage tickets
│   │   │   │
│   │   │   ├── entity/
│   │   │   │   ├── KnowledgeDocument.java      ← Uploaded doc metadata
│   │   │   │   ├── ChatSession.java            ← One per customer conversation
│   │   │   │   ├── ChatMessage.java            ← Individual messages
│   │   │   │   └── SupportTicket.java          ← Escalated issues
│   │   │   │
│   │   │   ├── repository/
│   │   │   │   ├── KnowledgeDocumentRepository.java
│   │   │   │   ├── ChatSessionRepository.java
│   │   │   │   ├── ChatMessageRepository.java
│   │   │   │   └── SupportTicketRepository.java
│   │   │   │
│   │   │   └── dto/
│   │   │       ├── ChatRequest.java
│   │   │       ├── ChatResponse.java
│   │   │       └── TicketRequest.java
│   │   │
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       └── db/migration/                  ← Flyway SQL migrations
│   │           ├── V1__init_schema.sql
│   │           └── V2__add_pgvector.sql
│   │
│   └── test/
│       └── java/com/supportai/
│           ├── service/
│           │   ├── DocumentIngestionServiceTest.java
│           │   └── SupportChatServiceTest.java
│           └── controller/
│               └── ChatControllerTest.java
│
├── docker-compose.yml                         ← PostgreSQL + pgvector
├── pom.xml
└── README.md
```

---

## Database Schema

### `knowledge_documents`
Tracks uploaded files and their ingestion status.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `file_name` | VARCHAR | Original filename |
| `file_type` | VARCHAR | pdf / docx / txt |
| `uploaded_at` | TIMESTAMP | Upload timestamp |
| `chunk_count` | INT | Number of chunks generated |
| `status` | VARCHAR | PENDING / INGESTED / FAILED |

### `chat_sessions`
One row per customer conversation.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key (also the Spring AI conversation ID) |
| `customer_email` | VARCHAR | Optional identifier |
| `created_at` | TIMESTAMP | Session start time |
| `last_active_at` | TIMESTAMP | Last message timestamp |

### `chat_messages`
Full message history per session.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `session_id` | UUID | Foreign key → chat_sessions |
| `role` | VARCHAR | USER / ASSISTANT |
| `content` | TEXT | Message text |
| `source_doc` | VARCHAR | Source document (for AI messages) |
| `created_at` | TIMESTAMP | Message timestamp |

### `support_tickets`
Escalated issues that the AI could not resolve.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `session_id` | UUID | Foreign key → chat_sessions |
| `issue_summary` | TEXT | Auto-generated or user-provided |
| `status` | VARCHAR | OPEN / IN_PROGRESS / RESOLVED |
| `created_at` | TIMESTAMP | Ticket creation time |

### `vector_store` (auto-created by Spring AI)
Stores document chunk embeddings.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `content` | TEXT | Raw chunk text |
| `metadata` | JSONB | Source doc name, chunk index, etc. |
| `embedding` | VECTOR(1536) | Embedding vector (HNSW indexed) |

---

## API Endpoints

### Admin endpoints (requires auth)

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/admin/documents` | Upload and ingest a document |
| `GET` | `/api/admin/documents` | List all documents in knowledge base |
| `DELETE` | `/api/admin/documents/{id}` | Remove document and its vectors |
| `GET` | `/api/admin/documents/{id}/chunks` | View chunks for a document |

### Customer chat endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/chat/session` | Start a new chat session |
| `POST` | `/api/chat/{sessionId}/message` | Send a message, get AI reply |
| `GET` | `/api/chat/{sessionId}/history` | Retrieve full conversation history |

### Ticket endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/tickets` | Raise a support ticket |
| `GET` | `/api/tickets/{id}` | Get ticket status |

---

## Getting Started

### Prerequisites

- Java 21+
- Maven 3.9+
- Docker and Docker Compose
- Anthropic API key
- OpenAI API key (for embeddings)

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/supportai.git
cd supportai
```

### 2. Start PostgreSQL with pgvector

```bash
docker-compose up -d
```

`docker-compose.yml`:
```yaml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: supportai
      POSTGRES_USER: supportai
      POSTGRES_PASSWORD: supportai
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

### 3. Set environment variables

```bash
export ANTHROPIC_API_KEY=your_anthropic_key_here
export OPENAI_API_KEY=your_openai_key_here
```

### 4. Run the application

```bash
mvn spring-boot:run
```

The app starts on `http://localhost:8080`.

### 5. Upload your first document

```bash
curl -X POST http://localhost:8080/api/admin/documents \
  -H "Authorization: Basic YWRtaW46YWRtaW4=" \
  -F "file=@returns-policy.pdf"
```

### 6. Ask a question

```bash
curl -X POST http://localhost:8080/api/chat/session \
  -H "Content-Type: application/json" \
  -d '{"customerEmail": "customer@example.com"}'

# Use the returned sessionId below
curl -X POST http://localhost:8080/api/chat/{sessionId}/message \
  -H "Content-Type: application/json" \
  -d '{"message": "What is your return policy?"}'
```

---

## Configuration

`src/main/resources/application.yml`:

```yaml
spring:
  application:
    name: supportai

  datasource:
    url: jdbc:postgresql://localhost:5432/supportai
    username: supportai
    password: supportai

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false

  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-6
          temperature: 0.1
          max-tokens: 1024

    openai:
      api-key: ${OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-3-small

    vectorstore:
      pgvector:
        initialize-schema: true
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1536
```

---

## Key Spring AI Components

| Component | Role in this project |
|---|---|
| `ChatClient` | Fluent API for sending prompts to Claude AI |
| `QuestionAnswerAdvisor` | Handles the full RAG retrieval + prompt augmentation automatically |
| `MessageChatMemoryAdvisor` | Injects conversation history per session into every prompt |
| `VectorStore` | Abstraction over pgvector — stores and searches embeddings |
| `EmbeddingModel` | Converts text to vector embeddings |
| `TikaDocumentReader` | Parses PDF, DOCX, TXT files into Spring AI `Document` objects |
| `TokenTextSplitter` | Splits documents into chunks for embedding |

---

## Features

- **RAG-powered answers** — AI answers are grounded in uploaded documents only
- **No hallucination** — system prompt explicitly restricts answers to provided context
- **Source citations** — every AI response includes the source document name
- **Conversation memory** — follow-up questions work naturally across a session
- **Multi-format ingestion** — supports PDF, DOCX, and TXT documents
- **Admin knowledge base management** — upload, list, and delete documents at runtime
- **Human escalation** — when AI cannot answer, customer can raise a support ticket
- **Session-based chat** — each customer gets an isolated conversation session

---


