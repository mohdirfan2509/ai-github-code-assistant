# DevPilot — AI-Powered GitHub Code Assistant

> **Chat with your GitHub codebase using Retrieval-Augmented Generation (RAG).**
>
> DevPilot connects to GitHub, synchronizes repositories, indexes source code into vector embeddings, stores those embeddings in PostgreSQL with pgvector, retrieves the most relevant code for a question, and uses Google Gemini to generate a context-aware answer with source/citation information.

![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1.1-brightgreen?logo=springboot)
![Spring AI](https://img.shields.io/badge/Spring%20AI-2.0.1-brightgreen)
![Next.js](https://img.shields.io/badge/Next.js-16-black?logo=next.js)
![TypeScript](https://img.shields.io/badge/TypeScript-blue?logo=typescript)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-18-blue?logo=postgresql)
![pgvector](https://img.shields.io/badge/pgvector-0.8.1-blue)
![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20RDS-orange?logo=amazonaws)
![Vercel](https://img.shields.io/badge/Frontend-Vercel-black?logo=vercel)
![GitHub OAuth](https://img.shields.io/badge/Auth-GitHub%20OAuth-black?logo=github)

## Table of Contents

- [Overview](#overview)
- [Why DevPilot?](#why-devpilot)
- [Features](#features)
- [Project Flow](#project-flow)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Repository Structure](#repository-structure)
- [Backend Architecture](#backend-architecture)
- [Frontend Architecture](#frontend-architecture)
- [RAG Pipeline](#rag-pipeline)
- [Indexing Pipeline](#indexing-pipeline)
- [Chat Pipeline](#chat-pipeline)
- [Authentication and Security](#authentication-and-security)
- [Database Design](#database-design)
- [Vector Search Configuration](#vector-search-configuration)
- [Configuration](#configuration)
- [Environment Variables](#environment-variables)
- [Local Development](#local-development)
- [Local PostgreSQL with Docker](#local-postgresql-with-docker)
- [Running the Backend](#running-the-backend)
- [Running the Frontend](#running-the-frontend)
- [GitHub OAuth Setup](#github-oauth-setup)
- [Production Deployment](#production-deployment)
- [AWS Architecture](#aws-architecture)
- [AWS Database Setup](#aws-database-setup)
- [AWS Backend Setup](#aws-backend-setup)
- [HTTPS and Reverse Proxy](#https-and-reverse-proxy)
- [Vercel Frontend Setup](#vercel-frontend-setup)
- [API Reference](#api-reference)
- [Indexing Behaviour](#indexing-behaviour)
- [Chat and Streaming](#chat-and-streaming)
- [Important Limits and Trade-offs](#important-limits-and-trade-offs)
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)
- [Development Workflow](#development-workflow)
- [Future Improvements](#future-improvements)
- [Project Learning](#project-learning)
- [Live Demo and Repository](#live-demo-and-repository)
- [Acknowledgements](#acknowledgements)

---

## Overview

**DevPilot** is a full-stack AI developer tool designed to help developers understand an unfamiliar GitHub repository using natural language.

Instead of sending an entire repository directly to an LLM, DevPilot uses a **Retrieval-Augmented Generation (RAG)** pipeline:

1. Authenticate the developer with GitHub.
2. Synchronize the developer's repositories.
3. Let the developer select a repository.
4. Read eligible source files from GitHub.
5. Split source files into manageable chunks.
6. Generate embeddings for each chunk.
7. Store vectors and metadata in PostgreSQL using pgvector.
8. When the user asks a question, retrieve the most relevant code chunks using semantic similarity.
9. Build an AI prompt from the retrieved context.
10. Send the grounded context to Gemini.
11. Stream the answer back to the browser.
12. Surface source/citation information so the user can understand where the answer came from.

The result is a repository-aware AI assistant rather than a generic chatbot.

---

## Why DevPilot?

Large language models can explain code, but their answer quality depends heavily on the context they receive.

A repository can contain thousands of lines across many files. Sending everything to an LLM is inefficient, expensive, and eventually constrained by context limits.

DevPilot addresses this with retrieval:

```text
Question
   ↓
Semantic Search
   ↓
Relevant Code Chunks
   ↓
Context Construction
   ↓
Gemini
   ↓
Grounded Answer
```

This makes it possible to ask questions such as:

- "Where is GitHub authentication implemented?"
- "How does repository indexing work?"
- "Which service retrieves relevant code before a chat response is generated?"
- "Where is the GitHub access token stored?"
- "How does the chat streaming response work?"
- "What happens when a repository is indexed?"

---

## Features

### GitHub Authentication

- GitHub OAuth 2.0 login.
- Requests GitHub permissions required for repository access.
- Creates and maintains an application user record.
- Uses a server-side session for authenticated API calls.
- Production OAuth works through an HTTPS reverse proxy.

### Repository Management

- Synchronizes repositories available to the authenticated GitHub account.
- Lists repositories in the dashboard.
- Opens repository-specific details.
- Starts repository indexing asynchronously.
- Exposes indexing status for the selected repository.

### Code Indexing

- Fetches repository files through the GitHub API.
- Filters files before processing.
- Enforces a maximum file-size limit.
- Splits source files into overlapping chunks.
- Generates embeddings for each chunk.
- Stores the embeddings and metadata in PostgreSQL + pgvector.
- Uses HNSW indexing for vector similarity search.

### AI Chat

- Repository-specific conversations.
- Persistent chat sessions.
- Persistent chat messages.
- Context-aware retrieval before generation.
- Multi-turn conversation support through stored sessions/messages.
- Streaming AI responses using Server-Sent Events (SSE).
- Citation/source mapping for retrieved code context.

### Developer Experience

- Next.js dashboard and chat interface.
- Repository cards and status indicators.
- Indexing progress/status UI.
- Dedicated chat view.
- Markdown rendering for AI responses.
- Theme switching support.
- Frontend API abstraction and streaming client.

### Security and Deployment

- GitHub OAuth instead of storing GitHub credentials directly in the browser.
- Configurable CORS with credential support.
- HTTP-only application session cookie.
- Token encryption configuration for stored GitHub tokens.
- Secrets supplied through environment variables.
- Private RDS database in AWS.
- EC2 → RDS access controlled using AWS security groups.
- HTTPS termination through Caddy.
- Backend managed by `systemd` on EC2.

---

## Project Flow

```text
┌─────────────────────────────┐
│        Next.js Frontend     │
│     Dashboard / Chat UI     │
└──────────────┬──────────────┘
               │ HTTPS
               ▼
┌─────────────────────────────┐
│      Spring Boot Backend    │
│ REST APIs + Security + RAG  │
└───────┬───────────┬─────────┘
        │           │
        │           │ GitHub API
        │           ▼
        │     ┌───────────────┐
        │     │    GitHub     │
        │     └───────────────┘
        │
        │ JDBC
        ▼
┌─────────────────────────────┐
│ PostgreSQL + pgvector       │
│ App Data + Vector Store     │
└──────────────┬──────────────┘
               │
               │ Retrieved Context
               ▼
┌─────────────────────────────┐
│       Google Gemini         │
│   Chat + Code Embeddings    │
└─────────────────────────────┘
```

### Repository Indexing Flow

```text
GitHub OAuth
    ↓
Select Repository
    ↓
Start Indexing
    ↓
Fetch Repository Files
    ↓
Filter Eligible Files
    ↓
Chunk Source Code
    ↓
Generate Embeddings
    ↓
Store in pgvector
    ↓
Mark Indexing Complete
```

### Question Answering Flow

```text
User Question
      ↓
Create / Use Chat Session
      ↓
Retrieve Relevant Vectors
      ↓
Build Code Context
      ↓
Construct Prompt
      ↓
Gemini
      ↓
Streaming Response
      ↓
Citations + Answer in UI
```

---

## Architecture

DevPilot is organized as a monorepo:

```text
ai-github-code-assistant/
├── backend/
├── frontend/
├── docker/
├── docker-compose.yml
└── .gitignore
```

### Local Architecture

```text
Browser
  │
  ▼
Next.js :3000
  │
  ▼
Spring Boot :8080
  │
  ▼
Docker PostgreSQL + pgvector :5433
```

### Production Architecture

```text
Browser
  │
  ▼
Vercel (Next.js)
  │
  │ HTTPS
  ▼
devpilot-api.duckdns.org
  │
  ▼
Caddy :443
  │
  ▼
Spring Boot :8080 on AWS EC2
  │
  │ Private VPC connection / TCP 5432
  ▼
Amazon RDS PostgreSQL + pgvector
  │
  ▼
Google Gemini API
```

---

## Technology Stack

### Frontend

- Next.js
- React
- TypeScript
- Tailwind CSS
- Browser Fetch API
- Server-Sent Events client for streamed chat
- Vercel for deployment

### Backend

- Java 21
- Spring Boot 4.1.1
- Spring Web MVC
- Spring Data JPA / Hibernate
- Spring Security
- OAuth 2.0 Client
- Spring AI 2.0.1
- Lombok
- Maven Wrapper

### AI

- Google Gemini chat model: `gemini-3.8-flash`
- Google Gemini embedding model: `gemini-embedding-001`
- Retrieval-Augmented Generation (RAG)
- Embeddings
- Semantic vector search

### Database

- PostgreSQL
- pgvector
- HNSW index
- Cosine distance
- 1536-dimensional vectors

### DevOps / Cloud

- Git
- GitHub
- Docker Compose for local database development
- AWS EC2
- Amazon RDS for PostgreSQL
- AWS VPC / Security Groups
- Caddy reverse proxy
- systemd
- DuckDNS
- Vercel

---

## Repository Structure

```text
ai-github-code-assistant/
│
├── backend/
│   ├── mvnw
│   ├── mvnw.cmd
│   ├── pom.xml
│   └── src/
│       └── main/
│           ├── java/devpilot/backend/
│           │   ├── config/
│           │   │   ├── AppConfig.java
│           │   │   ├── CorsConfig.java
│           │   │   ├── CryptoConfig.java
│           │   │   └── SecurityConfig.java
│           │   │
│           │   ├── controllers/
│           │   │   ├── AuthController.java
│           │   │   ├── ChatController.java
│           │   │   └── RepoController.java
│           │   │
│           │   ├── dto/
│           │   │   ├── ChatMessageRequest.java
│           │   │   ├── ChatMessageResponse.java
│           │   │   ├── ChatSessionResponse.java
│           │   │   ├── CitationDto.java
│           │   │   ├── CreateChatSessionRequest.java
│           │   │   ├── IndexStatusResponse.java
│           │   │   ├── RepositoryResponse.java
│           │   │   └── UserResponse.java
│           │   │
│           │   ├── entity/
│           │   │   ├── ChatMessage.java
│           │   │   ├── ChatSession.java
│           │   │   ├── IndexStatus.java
│           │   │   ├── MessageRole.java
│           │   │   ├── Repository.java
│           │   │   └── User.java
│           │   │
│           │   ├── exceptions/
│           │   │   ├── BadRequestException.java
│           │   │   ├── GlobalExceptionHandler.java
│           │   │   ├── NotFoundException.java
│           │   │   └── UnauthorizedException.java
│           │   │
│           │   ├── repository/
│           │   │   ├── ChatMessageRepository.java
│           │   │   ├── ChatSessionRepository.java
│           │   │   ├── RepositoryRepository.java
│           │   │   └── UserRepository.java
│           │   │
│           │   ├── security/
│           │   │   ├── AppUserPrincipal.java
│           │   │   ├── CurrentUser.java
│           │   │   └── GithubOAuth2UserService.java
│           │   │
│           │   └── services/
│           │       ├── ChatService.java
│           │       ├── RepoService.java
│           │       ├── UserService.java
│           │       │
│           │       ├── ai/
│           │       │   ├── ChatPromptBuilder.java
│           │       │   ├── ChatStreamHandler.java
│           │       │   ├── CitationMapper.java
│           │       │   ├── CodeContextRetriever.java
│           │       │   ├── RagSettings.java
│           │       │   └── RetrievedContext.java
│           │       │
│           │       ├── github/
│           │       │   ├── GitHubRateLimiter.java
│           │       │   └── GithubApiClient.java
│           │       │
│           │       └── indexing/
│           │           ├── CodeChunker.java
│           │           ├── CodeFileFilter.java
│           │           └── IndexingService.java
│           │
│           └── resources/
│               └── application.properties
│
├── frontend/
│   ├── app/
│   │   ├── auth/callback/
│   │   ├── chat/[repoId]/
│   │   ├── dashboard/
│   │   ├── login/
│   │   ├── globals.css
│   │   └── page.tsx
│   │
│   ├── components/
│   │   ├── chat/
│   │   ├── dashboard/
│   │   ├── icons/
│   │   ├── layout/
│   │   ├── providers/
│   │   └── ui/
│   │
│   ├── hooks/
│   │   ├── use-auth.ts
│   │   ├── use-chat.ts
│   │   └── use-repos.ts
│   │
│   ├── lib/
│   │   ├── api.ts
│   │   ├── dashboard-nav.ts
│   │   ├── query-keys.ts
│   │   └── stream-chat.ts
│   │
│   ├── proxy.ts
│   ├── package.json
│   └── package-lock.json
│
├── docker/
│   └── postgres/
│       └── init-extensions.sql
│
├── docker-compose.yml
└── .gitignore
```

---

## Backend Architecture

### Configuration Layer

`config/` contains application infrastructure configuration:

- **`AppConfig`** — application-level bean/configuration wiring.
- **`CorsConfig`** — reads `CORS_ALLOWED_ORIGINS` and enables credentialed CORS.
- **`CryptoConfig`** — crypto/token-encryption configuration.
- **`SecurityConfig`** — Spring Security, OAuth2, session management, protected routes, logout, and OAuth success/failure behaviour.

### Controllers

- **`AuthController`** — login URL and current-user endpoints.
- **`RepoController`** — repository listing, repository details, indexing, and indexing status.
- **`ChatController`** — chat session creation, history retrieval, and SSE message streaming.

### Services

- **`UserService`** — user-related application logic.
- **`RepoService`** — repository synchronization and repository state handling.
- **`ChatService`** — chat sessions, messages, and AI response orchestration.
- **`IndexingService`** — asynchronous repository indexing.

### GitHub Integration

- **`GithubApiClient`** — GitHub REST/API interaction.
- **`GithubRateLimiter`** — throttling/delay control around GitHub API usage.
- **`GithubOAuth2UserService`** — maps GitHub OAuth users into application users and handles GitHub identity/token data.

### Indexing Components

- **`CodeFileFilter`** — decides which files should be indexed.
- **`CodeChunker`** — splits source files into chunks.
- **`IndexingService`** — coordinates fetching, filtering, chunking, embedding, and vector storage.

### RAG / AI Components

- **`CodeContextRetriever`** — semantic retrieval of repository code chunks.
- **`RetrievedContext`** — represents retrieved code context.
- **`RagSettings`** — retrieval/RAG-related configuration.
- **`ChatPromptBuilder`** — constructs the model prompt from the question and retrieved context.
- **`ChatStreamHandler`** — handles streamed AI output.
- **`CitationMapper`** — maps retrieved material into citation/source DTOs.

### Exception Handling

The backend includes centralized exception handling for API errors, including bad requests, not-found conditions, and unauthorized access.

---

## Frontend Architecture

The frontend is a Next.js application organized around three major areas:

### App Routes

- Landing page.
- Login page.
- OAuth callback page.
- Dashboard.
- Dashboard overview.
- Dashboard settings.
- Repository chat page.

### Components

The UI contains dedicated components for:

- Chat composer.
- Chat messages.
- Chat sidebar.
- Chat view.
- Markdown rendering.
- Citation chips.
- Indexing state.
- Dashboard header.
- Repository cards.
- Repository status.
- Dashboard settings.
- Theme switching.
- Authentication guard.

### Data / API Layer

- `frontend/lib/api.ts` centralizes API requests.
- `frontend/lib/stream-chat.ts` handles streamed chat responses.
- `frontend/hooks/use-auth.ts` manages authentication state.
- `frontend/hooks/use-repos.ts` manages repository data.
- `frontend/hooks/use-chat.ts` manages chat data/state.

The frontend sends authenticated cross-origin requests using credentials so the backend session can be used from the deployed Vercel application.

---

## RAG Pipeline

RAG is the central idea behind DevPilot.

### What is RAG?

Retrieval-Augmented Generation combines two stages:

1. **Retrieval** — find information relevant to the user's question.
2. **Generation** — give that information to an LLM so it can generate a response grounded in the retrieved context.

For DevPilot, the knowledge source is the user's repository.

### Why a Vector Database?

Traditional keyword search can miss semantically related code.

Embeddings convert code chunks into vectors. Similar questions and code concepts tend to have similar vector representations, which makes semantic retrieval possible.

DevPilot uses:

```text
Source Code
   ↓
Chunk
   ↓
Embedding
   ↓
Vector
   ↓
pgvector
```

For a question:

```text
Question
   ↓
Question Embedding
   ↓
Vector Similarity Search
   ↓
Top Relevant Code Chunks
```

The retrieved chunks are then included in the Gemini prompt.

---

## Indexing Pipeline

The current indexing configuration is designed for code-oriented RAG.

### Configuration

```properties
app.indexing.max-file-bytes=102400
app.indexing.chunk-size=800
app.indexing.chunk-overlap=100
app.github.api-delay-ms=50
```

### Meaning

- **Maximum file size:** 100 KB.
- **Chunk size:** 800 units according to the application's chunking implementation.
- **Chunk overlap:** 100 units.
- **GitHub API delay:** 50 ms between controlled API operations according to the configured rate-limiting behaviour.

### Indexing Steps

```text
1. User selects repository
2. Backend verifies repository ownership
3. Indexing request is accepted
4. Indexing starts asynchronously
5. Repository files are fetched from GitHub
6. Unsupported/oversized files are filtered
7. Files are split into overlapping chunks
8. Each chunk receives an embedding
9. Chunk content + metadata + vector are stored in pgvector
10. Index status is updated
```

The asynchronous design prevents the initial HTTP request from waiting for the complete indexing operation.

---

## Chat Pipeline

A chat request follows this high-level sequence:

```text
User Question
      ↓
Authenticated Chat Session
      ↓
Repository-aware Retrieval
      ↓
Relevant Code Chunks
      ↓
Prompt Builder
      ↓
Gemini
      ↓
SSE Stream
      ↓
Frontend Chat UI
```

### Why Streaming?

The backend uses `text/event-stream` for chat responses so the frontend can display the answer progressively instead of waiting for the complete response.

This improves the interactive feel of an AI coding assistant, especially when a response takes several seconds to generate.

---

## Authentication and Security

### GitHub OAuth 2.0

The application uses GitHub OAuth2 for authentication.

The normal production flow is:

```text
Frontend
   ↓
/oauth2/authorization/github
   ↓
GitHub Authorization
   ↓
/login/oauth2/code/github
   ↓
Spring Security
   ↓
Authenticated Session
   ↓
Frontend /auth/callback
```

### Session Cookie

The backend uses a dedicated session cookie:

```text
DEVPILOT_SESSION
```

The deployed configuration uses browser security attributes suitable for the cross-site Vercel → AWS architecture.

### CORS

The allowed frontend origin is configurable through:

```text
CORS_ALLOWED_ORIGINS
```

Credentialed CORS is enabled so the browser can send the authenticated session cookie.

### Secret Management

Secrets are intentionally externalized to environment variables.

Never commit:

- Gemini API keys.
- GitHub client secrets.
- Database passwords.
- Token-encryption passwords.
- Token-encryption salts.
- Private key files.
- `.env` / `.env.local` files.

GitHub push protection previously caught secrets in the project's Git history; the repository history was cleaned before the final public push. Keep this practice going for all future changes.

---

## Database Design

DevPilot uses PostgreSQL for both application state and vector storage.

### Application Tables

The application includes these core entities/tables:

```text
users
repositories
chat_sessions
chat_messages
vector_store
```

### Users

Stores the application-side identity associated with a GitHub user.

Relevant information includes GitHub identity and profile information used by the application.

### Repositories

Stores synchronized GitHub repository information and indexing-related state.

### Chat Sessions

Represents a conversation associated with a repository and authenticated user.

### Chat Messages

Stores individual conversation messages and their roles/content.

### Vector Store

`vector_store` is the Spring AI PGVector-backed table used to store embedded repository chunks and associated metadata.

The production database also has the pgvector extension enabled.

---

## Vector Search Configuration

Current production vector configuration:

```properties
spring.ai.vectorstore.pgvector.table-name=vector_store
spring.ai.vectorstore.pgvector.initialize-schema=true
spring.ai.vectorstore.pgvector.dimensions=1536
spring.ai.vectorstore.pgvector.index-type=HNSW
spring.ai.vectorstore.pgvector.distance-type=COSINE_DISTANCE
```

### Configuration Summary

| Setting | Value |
|---|---|
| Vector table | `vector_store` |
| Dimensions | `1536` |
| Index | HNSW |
| Distance | Cosine distance |
| Database | PostgreSQL + pgvector |
| Embedding model | `gemini-embedding-001` |

### Important: Do Not Mix Embedding Models

Embeddings from different models are not interchangeable even when they have the same dimensionality.

If you change the embedding model, re-index the repository so that all stored vectors belong to the same embedding space.

---

## Configuration

The Spring Boot application externalizes environment-specific configuration from `application.properties`.

The production configuration follows the pattern:

```properties
DB_URL=${DB_URL}
DB_USERNAME=${DB_USERNAME}
DB_PASSWORD=${DB_PASSWORD}

spring.ai.google.genai.api-key=${GEMINI_API_KEY}

spring.security.oauth2.client.registration.github.client-id=${GITHUB_CLIENT_ID}
spring.security.oauth2.client.registration.github.client-secret=${GITHUB_CLIENT_SECRET}

app.frontend-url=${FRONTEND_URL}
app.cors.allowed-origins=${CORS_ALLOWED_ORIGINS}
app.token-encryptor-password=${TOKEN_ENCRYPTOR_PASSWORD}
app.token-encryptor-salt=${TOKEN_ENCRYPTOR_SALT}
```

For the production reverse proxy, the backend also uses forwarded-header handling so Spring Security can generate HTTPS OAuth redirect URIs correctly behind Caddy.

---

## Environment Variables

### Backend Variables

Create these only in the runtime environment:

```text
DB_URL
DB_USERNAME
DB_PASSWORD
GEMINI_API_KEY
GITHUB_CLIENT_ID
GITHUB_CLIENT_SECRET
FRONTEND_URL
CORS_ALLOWED_ORIGINS
TOKEN_ENCRYPTOR_PASSWORD
TOKEN_ENCRYPTOR_SALT
```

### Example: Local Backend

```bash
export DB_URL="jdbc:postgresql://localhost:5433/devpilot"
export DB_USERNAME="postgres"
export DB_PASSWORD="postgres"

export GEMINI_API_KEY="YOUR_GEMINI_API_KEY"

export GITHUB_CLIENT_ID="YOUR_GITHUB_CLIENT_ID"
export GITHUB_CLIENT_SECRET="YOUR_GITHUB_CLIENT_SECRET"

export FRONTEND_URL="http://localhost:3000"
export CORS_ALLOWED_ORIGINS="http://localhost:3000"

export TOKEN_ENCRYPTOR_PASSWORD="YOUR_RANDOM_SECRET"
export TOKEN_ENCRYPTOR_SALT="YOUR_RANDOM_SALT"
```

### Example: Production Backend

```bash
export DB_URL="jdbc:postgresql://YOUR_RDS_ENDPOINT:5432/devpilot"
export DB_USERNAME="devpilot_admin"
export DB_PASSWORD="YOUR_RDS_PASSWORD"

export GEMINI_API_KEY="YOUR_GEMINI_API_KEY"

export GITHUB_CLIENT_ID="YOUR_GITHUB_CLIENT_ID"
export GITHUB_CLIENT_SECRET="YOUR_GITHUB_CLIENT_SECRET"

export FRONTEND_URL="https://frontend-psi-steel-22.vercel.app"
export CORS_ALLOWED_ORIGINS="https://frontend-psi-steel-22.vercel.app"

export TOKEN_ENCRYPTOR_PASSWORD="YOUR_RANDOM_SECRET"
export TOKEN_ENCRYPTOR_SALT="YOUR_RANDOM_SALT"
```

### Frontend Variable

The frontend uses:

```text
NEXT_PUBLIC_API_BASE_URL
```

Local development:

```text
NEXT_PUBLIC_API_BASE_URL=http://localhost:8080
```

Production:

```text
NEXT_PUBLIC_API_BASE_URL=https://devpilot-api.duckdns.org
```

`NEXT_PUBLIC_*` values are intended for browser-side configuration and therefore must never be used for secrets.

---

## Local Development

### Prerequisites

Install:

- JDK 21.
- Git.
- Maven (the project also contains a Maven wrapper).
- Node.js and npm compatible with the Next.js project.
- Docker and Docker Compose.
- A GitHub OAuth application.
- A Gemini API key for AI functionality.

### Clone

```bash
git clone https://github.com/mohdirfan2509/ai-github-code-assistant.git
cd ai-github-code-assistant
```

---

## Local PostgreSQL with Docker

The repository includes a `docker-compose.yml` for local PostgreSQL + pgvector development.

The local database is configured around:

```text
Database:  devpilot
User:      postgres
Password:  postgres
Host port: 5433
```

The compose setup uses a pgvector-enabled PostgreSQL image and mounts the extension initialization script.

Start it with:

```bash
docker compose up -d postgres
```

Check:

```bash
docker compose ps
```

Stop it with:

```bash
docker compose down
```

To remove the local database volume as well:

```bash
docker compose down -v
```

> **Warning:** `docker compose down -v` deletes the local PostgreSQL volume and all local database data.

---

## Running the Backend

From the repository root:

```bash
cd backend
```

Make the Maven wrapper executable if necessary:

```bash
chmod +x ./mvnw
```

Set your environment variables, then run:

```bash
./mvnw spring-boot:run
```

Or build a JAR:

```bash
./mvnw clean package
```

Then run:

```bash
java -jar target/backend-0.0.1-SNAPSHOT.jar
```

The backend listens on:

```text
http://localhost:8080
```

### Verify the Backend

A public authentication helper endpoint is:

```text
GET /api/auth/login-url
```

Example:

```bash
curl http://localhost:8080/api/auth/login-url
```

Expected response:

```json
{"url":"/oauth2/authorization/github"}
```

---

## Running the Frontend

From the repository root:

```bash
cd frontend
npm install
```

Create/update `.env.local`:

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8080
```

Start development mode:

```bash
npm run dev
```

Open:

```text
http://localhost:3000
```

The local login callback should point back to your local Spring Boot backend.

---

## GitHub OAuth Setup

Create a GitHub OAuth App and configure:

### Local Development

Homepage URL:

```text
http://localhost:3000
```

Authorization callback URL:

```text
http://localhost:8080/login/oauth2/code/github
```

### Production

Homepage URL:

```text
https://frontend-psi-steel-22.vercel.app
```

Authorization callback URL:

```text
https://devpilot-api.duckdns.org/login/oauth2/code/github
```

The production callback must be HTTPS because the deployed backend is served through Caddy.

---

## Production Deployment

The application is deployed as a simple cloud architecture intended for a student/demo project rather than a large production system.

### Production Components

| Layer | Service |
|---|---|
| Frontend | Vercel / Next.js |
| Backend | AWS EC2 / Java 21 / Spring Boot |
| Reverse Proxy | Caddy |
| Database | Amazon RDS PostgreSQL |
| Vector Store | PostgreSQL + pgvector |
| Authentication | GitHub OAuth 2.0 |
| AI | Google Gemini |
| DNS | DuckDNS |
| Process Manager | systemd |
| Source Control | GitHub |

### Current Production URLs

Frontend:

```text
https://frontend-psi-steel-22.vercel.app
```

Backend:

```text
https://devpilot-api.duckdns.org
```

Repository:

```text
https://github.com/mohdirfan2509/ai-github-code-assistant
```

---

## AWS Architecture

### EC2

The backend runs on an ARM64 Amazon Linux 2023 EC2 instance using an eligible burstable T4g class for the student/demo setup.

The deployed server uses:

- Java 21 / Amazon Corretto.
- Maven Wrapper.
- Git.
- Caddy.
- systemd.

### Security Group

The backend security group exposes:

```text
22   → My IP only
80   → Public
443  → Public
```

Port `8080` is not publicly exposed in the final setup.

Spring Boot continues to listen internally on `localhost:8080`, while Caddy handles public HTTPS traffic.

### RDS

The production database is:

- Amazon RDS for PostgreSQL.
- PostgreSQL 18.x server line used by the deployment.
- Private access (`Public access = No`).
- Database name: `devpilot`.
- Master/application database user: `devpilot_admin`.
- PostgreSQL port: `5432`.
- pgvector extension enabled.

### RDS Security Group

The RDS security group permits PostgreSQL traffic from the backend EC2 security group:

```text
EC2 devpilot-backend-sg
        │
        │ TCP 5432
        ▼
RDS devpilot-rds-sg
```

The RDS database is not exposed directly to the public internet.

---

## AWS Database Setup

The production database setup was:

```text
Amazon RDS
   ↓
PostgreSQL
   ↓
Private VPC access
   ↓
EC2 application
```

After connecting to RDS from the EC2 instance, pgvector was enabled with:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

The deployed database currently reports pgvector `0.8.1`.

Verify it with:

```sql
SELECT extname, extversion
FROM pg_extension
WHERE extname = 'vector';
```

---

## AWS Backend Setup

### Build

On EC2:

```bash
cd ~/ai-github-code-assistant/backend
./mvnw clean package -DskipTests
```

### Run Manually

```bash
java -Xms128m -Xmx700m -jar target/backend-0.0.1-SNAPSHOT.jar
```

### systemd

Production uses a `systemd` service so the backend can continue running after an SSH session ends and can restart after a host reboot.

Example service shape:

```ini
[Unit]
Description=DevPilot Spring Boot Backend
After=network-online.target
Wants=network-online.target

[Service]
User=ec2-user
WorkingDirectory=/home/ec2-user/ai-github-code-assistant/backend
EnvironmentFile=/home/ec2-user/devpilot.env
ExecStart=/usr/lib/jvm/java-21-amazon-corretto.aarch64/bin/java -Xms128m -Xmx700m -jar /home/ec2-user/ai-github-code-assistant/backend/target/backend-0.0.1-SNAPSHOT.jar
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable devpilot
sudo systemctl start devpilot
```

Check:

```bash
sudo systemctl status devpilot
```

Logs:

```bash
sudo journalctl -u devpilot -n 100 --no-pager
```

---

## HTTPS and Reverse Proxy

The Spring Boot application listens internally on port `8080`.

Caddy terminates HTTPS and proxies requests to Spring Boot:

```text
Internet
   ↓ HTTPS :443
Caddy
   ↓ HTTP localhost:8080
Spring Boot
```

Example Caddy configuration:

```caddyfile
devpilot-api.duckdns.org {
    reverse_proxy localhost:8080
}
```

The public API is therefore accessed through:

```text
https://devpilot-api.duckdns.org
```

Forwarded headers are enabled in Spring Boot so OAuth redirect URIs are generated with `https://` when the application is behind Caddy.

---

## Vercel Frontend Setup

The production frontend is deployed on Vercel.

Set the Vercel production environment variable:

```text
NEXT_PUBLIC_API_BASE_URL=https://devpilot-api.duckdns.org
```

After changing a `NEXT_PUBLIC_*` variable, redeploy the frontend so the build receives the new value.

The frontend's production flow is:

```text
https://frontend-psi-steel-22.vercel.app
                    ↓
https://devpilot-api.duckdns.org
```

---

## API Reference

The backend exposes REST endpoints under `/api`.

Authentication endpoints:

| Method | Endpoint | Auth | Purpose |
|---|---|---:|---|
| GET | `/api/auth/login-url` | No | Returns the GitHub OAuth authorization path |
| GET | `/api/auth/me` | Yes | Returns the current authenticated user |
| POST | `/api/auth/logout` | Yes | Logs out the current user and invalidates the session |

OAuth endpoints handled by Spring Security:

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/oauth2/authorization/github` | Starts GitHub OAuth |
| GET | `/login/oauth2/code/github` | GitHub OAuth callback |

Repository endpoints:

| Method | Endpoint | Auth | Purpose |
|---|---|---:|---|
| GET | `/api/repos` | Yes | Lists/synchronizes the user's repositories |
| GET | `/api/repos/{id}` | Yes | Gets one owned repository |
| POST | `/api/repos/{id}/index` | Yes | Starts asynchronous indexing |
| GET | `/api/repos/{id}/status` | Yes | Gets indexing status |

Chat endpoints:

| Method | Endpoint | Auth | Purpose |
|---|---|---:|---|
| POST | `/api/chat/sessions` | Yes | Creates a repository chat session |
| GET | `/api/chat/sessions?repositoryId={id}` | Yes | Lists sessions for a repository |
| GET | `/api/chat/sessions/{id}` | Yes | Gets messages for a chat session |
| POST | `/api/chat/sessions/{id}/messages` | Yes | Sends a message and streams the AI response |

The streaming chat endpoint returns:

```text
Content-Type: text/event-stream
```

---

## Indexing Behaviour

Indexing is asynchronous.

When the client calls:

```text
POST /api/repos/{id}/index
```

the backend:

1. Validates the current user.
2. Verifies repository ownership.
3. Marks/starts indexing state.
4. Returns an accepted response.
5. Runs the actual indexing work asynchronously.
6. Updates the index status as processing completes.

This allows the UI to continue operating while a repository is being processed.

Use:

```text
GET /api/repos/{id}/status
```

to monitor progress.

---

## Chat and Streaming

Chat is repository-specific.

A typical sequence is:

```text
Create Session
   ↓
Select Repository
   ↓
Ask Question
   ↓
Semantic Retrieval
   ↓
Gemini Prompt
   ↓
SSE Stream
   ↓
Render Markdown + Citations
```

The frontend keeps the streaming connection open and renders text progressively.

Stored chat sessions and messages allow conversations to be continued rather than treating every question as an isolated request.

---

## Important Limits and Trade-offs

DevPilot is a learning/demo application and intentionally favors understandable architecture over production-scale infrastructure.

### AI API Quotas

The project currently uses Gemini API access intended for low-cost/free development and demo use.

**Free quotas can be exhausted at any time.** If the model stops responding while the rest of the application is healthy, check the provider's current quota/rate-limit status first.

### EC2 Size

The backend is running on a small burstable EC2 instance. This is suitable for a demo workload but is not intended for large-scale concurrent indexing or chat traffic.

### RDS

The RDS instance is configured for a small application/demo deployment and is not a multi-AZ/high-availability architecture.

### Single Backend Instance

The current design has a single Spring Boot server. There is no load balancer or horizontal auto-scaling layer.

### Runtime Indexing

Repository indexing happens inside the application process. A large repository can therefore consume CPU, memory, network bandwidth, and AI embedding quota.

### Public Demo Dependency

The live deployment depends on:

- The EC2 instance remaining running.
- The current DNS record pointing to the instance.
- Valid Gemini API quota.
- GitHub OAuth configuration.
- RDS availability.
- The current Vercel deployment.

---

## Troubleshooting

### 1. `401 Unauthorized` from `/api/auth/me`

Typical causes:

- Session cookie not being sent.
- Cross-site cookie attributes are incorrect.
- Frontend request does not include credentials.
- CORS origin does not match the Vercel origin.
- The OAuth callback completed but the browser did not retain/send the backend session.

The frontend already uses credentialed fetch requests, and production session cookies are configured for the cross-site Vercel → AWS architecture.

### 2. GitHub says redirect URI mismatch

Verify that the callback URI is exactly:

```text
https://devpilot-api.duckdns.org/login/oauth2/code/github
```

Also verify the application is honoring forwarded HTTPS headers when running behind Caddy.

### 3. Backend returns `http://` as the OAuth redirect scheme

This means the reverse-proxy protocol information is not being honored.

Ensure the application includes forwarded-header strategy configuration appropriate for the proxy deployment.

### 4. PostgreSQL connection fails

Check:

```text
DB_URL
DB_USERNAME
DB_PASSWORD
```

For production, also verify:

- EC2 and RDS are in the same VPC.
- RDS is reachable on TCP 5432 from the EC2 security group.
- RDS remains private.

Test the network path from EC2:

```bash
nc -vz YOUR_RDS_ENDPOINT 5432
```

### 5. `vector` extension is missing

Connect to PostgreSQL and run:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

Then verify:

```sql
SELECT extname, extversion
FROM pg_extension
WHERE extname = 'vector';
```

### 6. RAG returns poor or unrelated answers

Check:

- Whether the repository was actually indexed.
- Index status endpoint.
- Whether the vector store contains data.
- Whether the repository was indexed using the current embedding model.
- Whether embedding model and vector dimensions match.
- Retrieval configuration and chunking.

If the embedding model changes, re-index the repository rather than mixing old and new vectors.

### 7. Gemini returns quota/rate-limit errors

Check your Gemini quota and account limits first.

This project is intentionally configured for low-cost/demo usage, so a free API quota can be exhausted even when the application itself is healthy.

### 8. Backend service is down

Check:

```bash
sudo systemctl status devpilot
```

Then:

```bash
sudo journalctl -u devpilot -n 100 --no-pager
```

### 9. Caddy is down

Check:

```bash
sudo systemctl status caddy
```

Then:

```bash
sudo journalctl -u caddy -n 100 --no-pager
```

Validate configuration:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

### 10. Vercel still calls `localhost:8080`

Verify the Vercel production variable:

```text
NEXT_PUBLIC_API_BASE_URL=https://devpilot-api.duckdns.org
```

Then redeploy the Vercel application.

---

## Security Notes

This application handles GitHub repository access, which can include private code.

### Never Commit Secrets

Keep these out of Git:

```text
.env
.env.local
*.pem
API keys
GitHub client secrets
Database passwords
encryption passwords
```

### Protect Private Repositories

The application requests the GitHub repository scope required for its repository integration. Only connect repositories you are comfortable sending relevant code context to the configured AI provider.

### AI Data Flow

When a user asks a question, DevPilot retrieves relevant repository chunks and sends the retrieved context to Gemini to generate the response.

Do not index highly sensitive code unless you understand the data-handling/privacy implications of the configured AI provider and account.

### Database Exposure

RDS should remain private. Do not add a public `5432` inbound rule such as:

```text
0.0.0.0/0 → TCP 5432
```

The intended production rule is:

```text
EC2 backend security group → RDS security group → TCP 5432
```

---

## Development Workflow

A simple development workflow is:

```text
1. Create/modify feature locally
2. Run backend and frontend locally
3. Test authentication
4. Test repository synchronization
5. Test indexing
6. Test RAG chat
7. Commit changes
8. Push to GitHub
9. Redeploy Vercel if frontend changed
10. Pull latest commit on EC2 if backend changed
11. Rebuild backend JAR
12. Restart systemd service
13. Smoke-test production
```

### Backend Update on EC2

```bash
cd ~/ai-github-code-assistant
git pull origin main
cd backend
./mvnw clean package -DskipTests
sudo systemctl restart devpilot
sudo systemctl status devpilot
```

### Frontend Update

If Vercel is connected to the GitHub repository, pushing to the configured branch can trigger a deployment according to the Vercel project's deployment settings.

---

## Future Improvements

The current architecture is functional for a demo, but there are several natural next steps.

### Infrastructure

- Use a custom domain instead of a temporary DuckDNS hostname.
- Add managed DNS and certificate management.
- Add an Application Load Balancer if the application becomes multi-instance.
- Move backend deployment to ECS/Fargate or another managed container platform.
- Introduce automated CI/CD from GitHub Actions.
- Add centralized logging and metrics.
- Add backups/restore procedures and stronger production database policies.

### Application

- Add repository-level re-index/reset controls.
- Add incremental indexing instead of full re-indexing.
- Track commit/branch/version metadata for indexed content.
- Add more sophisticated code-aware chunking.
- Add language-specific parsing/AST-aware retrieval.
- Add hybrid keyword + vector retrieval.
- Add reranking after vector retrieval.
- Add configurable top-K retrieval.
- Add configurable model/provider selection.
- Add more explicit source/file/line citations.
- Add repository search and code navigation.
- Add chat export/share capabilities.

### Reliability

- Move indexing to a durable job queue.
- Add retries with backoff for GitHub/AI API failures.
- Add idempotent indexing jobs.
- Add concurrency controls.
- Add request-level timeouts and circuit breakers.
- Add observability around indexing duration, chunk counts, embedding failures, and retrieval quality.

---

## Project Learning

DevPilot was built as a hands-on exploration of modern Java backend development combined with AI.

The project provided practical experience with:

- Java 21.
- Spring Boot REST APIs.
- Spring Security.
- GitHub OAuth2.
- Spring Data JPA and Hibernate.
- Spring AI.
- RAG architecture.
- Embeddings.
- Vector databases.
- pgvector.
- HNSW vector indexing.
- Semantic retrieval.
- Streaming AI responses with SSE.
- GitHub API integration.
- Asynchronous indexing.
- Frontend/backend session integration.
- CORS and cross-site cookies.
- Docker Compose.
- AWS EC2.
- AWS RDS.
- VPC and security groups.
- Linux systemd services.
- Reverse proxying with Caddy.
- HTTPS certificates.
- Vercel deployment.

This project was also the first hands-on introduction to RAG and vector databases, and the deployment was the first AWS backend deployment for the project.

---

## Live Demo and Repository

### GitHub Repository

https://github.com/mohdirfan2509/ai-github-code-assistant

### Live Frontend

https://frontend-psi-steel-22.vercel.app/

### Production Backend

https://devpilot-api.duckdns.org

### Example Public API Endpoint

https://devpilot-api.duckdns.org/api/auth/login-url

> **Demo note:** The AI provider is currently accessed through a free/development API plan. Model availability and free quota/rate limits can change or be exhausted.

---

## Acknowledgements

Special thanks to my Java mentors:

- **Nitin Sir**
- **Zabi Sir**

Their guidance and teaching helped me build a stronger foundation in Java and Spring Boot and gave me the confidence to take the project from a local application to a deployed AI-enabled cloud application.

---

## Final Summary

DevPilot brings together several layers of modern application engineering in one project:

```text
GitHub OAuth
     ↓
Repository Synchronization
     ↓
Code Indexing
     ↓
Embeddings
     ↓
PostgreSQL + pgvector
     ↓
Semantic Retrieval
     ↓
Spring AI
     ↓
Gemini
     ↓
Streaming RAG Answer
     ↓
Next.js Chat UI
```

The project demonstrates how a traditional **Java + Spring Boot backend** can be extended into a practical **AI/RAG application**, while also covering authentication, persistence, cloud networking, HTTPS, and deployment.

---

## License

No license file is currently specified for this repository.
