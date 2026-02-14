# Civora — Requirements Document

> **AI-Powered Community Access Platform**
> Version 1.0 · February 2026

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Problem Statement](#2-problem-statement)
3. [Objectives](#3-objectives)
4. [Scope](#4-scope)
5. [User Roles](#5-user-roles)
6. [Functional Requirements](#6-functional-requirements)
7. [Non-Functional Requirements](#7-non-functional-requirements)
8. [Data Requirements](#8-data-requirements)
9. [API Requirements](#9-api-requirements)
10. [System Constraints](#10-system-constraints)
11. [Assumptions](#11-assumptions)
12. [Future Enhancements](#12-future-enhancements)

---

## 1. Project Overview

**Civora** is an AI-powered community opportunity intelligence platform that bridges the gap between citizens and the government welfare ecosystem. It aggregates data on government schemes, scholarships, jobs, subsidies, and welfare programs from heterogeneous public sources and delivers **personalized, contextually relevant** recommendations to each user.

The platform leverages:

- **Web Scraping & Public APIs** for real-time data ingestion
- **MongoDB** for structured opportunity and user data storage
- **Vector Database (FAISS / Pinecone)** for semantic search over opportunity embeddings
- **Retrieval-Augmented Generation (RAG)** for natural-language, context-aware responses
- **Personalized Ranking Engine** to score and surface the most relevant results
- **LLM Integration (Gemini / OpenAI)** to generate simplified eligibility checks and step-by-step application guidance

### High-Level System Flow

```
Data Sources ──► Ingestion Pipeline ──► MongoDB + Vector DB
                                              │
User Login ──► Profile Creation ──► Query ──► Embedding ──► Vector Search
                                              │
                                      Ranking Engine ──► RAG + LLM ──► Response
```

---

## 2. Problem Statement

India operates **700+** central and state government schemes, thousands of scholarships, and lakhs of job openings across multiple portals. Despite this abundance:

- **Citizens lack awareness** — over 40% of eligible beneficiaries never apply because they don't know a scheme exists.
- **Information is fragmented** — data is spread across dozens of ministry websites, each with different formats, languages, and navigation structures.
- **Eligibility criteria are complex** — legalese and bureaucratic language deter users from understanding whether they qualify.
- **No personalization exists** — existing portals show the same content to everyone regardless of age, income, location, or education.
- **Updates are missed** — deadlines pass, new schemes launch, and criteria change without users being notified.

There is a critical need for a **single, intelligent, personalized interface** that aggregates all opportunities and translates them into actionable, user-specific guidance.

---

## 3. Objectives

| # | Objective | Success Metric |
|---|-----------|---------------|
| O1 | Aggregate government schemes, scholarships, jobs, and welfare programs from 50+ sources | ≥ 10,000 unique opportunities indexed |
| O2 | Deliver personalized recommendations based on user profile | ≥ 85% relevance rating in user feedback |
| O3 | Simplify eligibility information using AI-generated plain-language summaries | Avg. readability score ≤ Grade 8 (Flesch-Kincaid) |
| O4 | Provide step-by-step application guidance | ≥ 90% of listed opportunities include guidance |
| O5 | Support multilingual access (Hindi, English, regional languages) | ≥ 3 languages at launch |
| O6 | Ensure data freshness via automated cron-based updates | Data staleness < 24 hours |
| O7 | Enable voice-based input for low-literacy users | Voice query success rate ≥ 80% |

---

## 4. Scope

### 4.1 In-Scope

- User registration, authentication (email/phone + OTP), and profile management
- Web scraping pipelines for government portals (central + select state portals)
- Public API integrations (NCS, scholarship portals, open data APIs)
- Data normalization, deduplication, and storage in MongoDB
- Embedding generation using sentence-transformer models
- Vector database indexing and semantic similarity search
- Personalized ranking engine combining semantic similarity, profile match, and deadline urgency
- RAG pipeline with LLM-generated responses (eligibility summaries, application steps)
- Multilingual support (English, Hindi, and at least one regional language)
- Voice input via browser-based speech-to-text
- Admin dashboard for monitoring data pipelines and content moderation
- Cron-based scheduled data refresh (scraping + API pulls)
- RESTful API backend
- Responsive web frontend

### 4.2 Out-of-Scope

- Mobile native applications (iOS/Android) — web-responsive only at launch
- Direct application submission to government portals on behalf of users
- Payment processing or financial transactions
- Aadhaar-based eKYC verification
- Offline-first or PWA capabilities (Phase 2)
- Real-time chat with government officials
- Legal advisory or dispute resolution features

---

## 5. User Roles

### 5.1 Citizen (End User)

| Attribute | Details |
|-----------|---------|
| **Description** | Primary user seeking government schemes, scholarships, or jobs |
| **Access Level** | Authenticated |
| **Capabilities** | Register/login, create/edit profile, submit queries, view recommendations, save opportunities, receive notifications |
| **Profile Fields** | Name, age, gender, location (state/district), income bracket, education level, caste/category, occupation, disabilities, interests |

### 5.2 Admin

| Attribute | Details |
|-----------|---------|
| **Description** | Platform administrator managing data quality and system operations |
| **Access Level** | Privileged |
| **Capabilities** | Monitor scraping pipelines, review/flag/edit opportunities, manage users, view analytics, trigger manual data refresh, configure ranking weights |

### 5.3 System (Automated Agent)

| Attribute | Details |
|-----------|---------|
| **Description** | Automated services performing background tasks |
| **Access Level** | Internal |
| **Capabilities** | Execute cron jobs, run scrapers, generate embeddings, update vector index, send notification triggers, log pipeline metrics |

---

## 6. Functional Requirements

### 6.1 User Authentication

| ID | Requirement | Priority |
|----|------------|----------|
| FR-AUTH-01 | System shall support user registration via email or phone number | P0 |
| FR-AUTH-02 | System shall authenticate users via OTP-based verification | P0 |
| FR-AUTH-03 | System shall support social login (Google OAuth 2.0) | P1 |
| FR-AUTH-04 | System shall issue JWT tokens with configurable expiry (default: 7 days) | P0 |
| FR-AUTH-05 | System shall support password reset via email/SMS OTP | P0 |
| FR-AUTH-06 | System shall enforce rate limiting on authentication endpoints (max 5 attempts / minute) | P0 |

### 6.2 Profile Management

| ID | Requirement | Priority |
|----|------------|----------|
| FR-PROF-01 | System shall collect user profile: age, gender, location (state + district), annual income, education level, category (SC/ST/OBC/General), occupation, disabilities | P0 |
| FR-PROF-02 | System shall allow users to update their profile at any time | P0 |
| FR-PROF-03 | System shall support interest tagging (e.g., agriculture, education, healthcare, housing) | P1 |
| FR-PROF-04 | System shall compute a profile completeness score and prompt users to fill missing fields | P1 |
| FR-PROF-05 | System shall use profile data as context for personalized ranking and RAG responses | P0 |

### 6.3 Data Ingestion (Web Scraping + Public APIs)

| ID | Requirement | Priority |
|----|------------|----------|
| FR-ING-01 | System shall scrape data from government scheme portals (e.g., MyScheme, NCS, NSP, state portals) | P0 |
| FR-ING-02 | System shall integrate with public APIs where available (e.g., data.gov.in, NCS API) | P0 |
| FR-ING-03 | System shall normalize scraped data into a unified opportunity schema | P0 |
| FR-ING-04 | System shall deduplicate opportunities based on title + source + scheme ID | P0 |
| FR-ING-05 | System shall log all ingestion runs with status, record count, errors, and duration | P0 |
| FR-ING-06 | System shall support configurable scraping targets (URL list, selectors, frequency) | P1 |
| FR-ING-07 | System shall handle anti-scraping measures gracefully (retries, backoff, proxy rotation) | P1 |

### 6.4 Opportunity Storage

| ID | Requirement | Priority |
|----|------------|----------|
| FR-STORE-01 | System shall store all opportunities in MongoDB with full metadata | P0 |
| FR-STORE-02 | Each opportunity record shall include: title, description, eligibility criteria, benefits, application process, deadlines, source URL, category, location scope, last updated timestamp | P0 |
| FR-STORE-03 | System shall maintain a status field (active / expired / upcoming) for each opportunity | P0 |
| FR-STORE-04 | System shall index opportunities for fast lookup by category, location, and status | P0 |

### 6.5 Embedding Generation

| ID | Requirement | Priority |
|----|------------|----------|
| FR-EMB-01 | System shall generate vector embeddings for each opportunity using a sentence-transformer model (e.g., `all-MiniLM-L6-v2` or `multilingual-e5-base`) | P0 |
| FR-EMB-02 | Embeddings shall encode a composite text: `title + description + eligibility + benefits` | P0 |
| FR-EMB-03 | System shall regenerate embeddings when opportunity data is updated | P0 |
| FR-EMB-04 | Embedding dimension shall be consistent across the pipeline (e.g., 384-d or 768-d) | P0 |
| FR-EMB-05 | System shall support batch embedding generation for bulk ingestion | P1 |

### 6.6 Vector Search

| ID | Requirement | Priority |
|----|------------|----------|
| FR-VEC-01 | System shall store opportunity embeddings in a vector database (FAISS for MVP; Pinecone for production) | P0 |
| FR-VEC-02 | System shall convert user queries into embeddings using the same model as opportunity embeddings | P0 |
| FR-VEC-03 | System shall retrieve Top-K (default K=20) semantically similar opportunities for a given query | P0 |
| FR-VEC-04 | System shall support metadata filtering during vector search (e.g., location, category, status=active) | P1 |
| FR-VEC-05 | Vector search latency shall be < 200ms for 100K indexed records | P0 |

### 6.7 Personalized Ranking

| ID | Requirement | Priority |
|----|------------|----------|
| FR-RANK-01 | System shall re-rank vector search results using a composite scoring function | P0 |
| FR-RANK-02 | Ranking factors shall include: semantic similarity score, profile match score, deadline urgency score, scheme popularity score | P0 |
| FR-RANK-03 | Profile match score shall evaluate: age eligibility, income eligibility, location match, education match, category match | P0 |
| FR-RANK-04 | Deadline urgency score shall increase as the application deadline approaches | P1 |
| FR-RANK-05 | Admin shall be able to configure ranking weights via admin dashboard | P1 |
| FR-RANK-06 | System shall return Top-N (default N=5) ranked results to the user | P0 |

### 6.8 RAG-based Response Generation

| ID | Requirement | Priority |
|----|------------|----------|
| FR-RAG-01 | System shall use RAG architecture: retrieve relevant opportunities, inject as context into an LLM prompt | P0 |
| FR-RAG-02 | LLM shall generate a plain-language summary of eligibility for the user based on their profile | P0 |
| FR-RAG-03 | LLM shall generate step-by-step application guidance for each recommended opportunity | P0 |
| FR-RAG-04 | System shall include source URLs in generated responses for verifiability | P0 |
| FR-RAG-05 | System shall use prompt engineering to prevent hallucination (ground responses in retrieved context only) | P0 |
| FR-RAG-06 | System shall support LLM provider abstraction (Gemini / OpenAI / open-source fallback) | P1 |
| FR-RAG-07 | LLM response latency shall be < 5 seconds for a single query | P1 |

### 6.9 Multilingual Support

| ID | Requirement | Priority |
|----|------------|----------|
| FR-LANG-01 | System shall support English and Hindi at launch | P0 |
| FR-LANG-02 | System shall support at least one additional regional language (e.g., Tamil, Telugu, Bengali) | P1 |
| FR-LANG-03 | User queries in any supported language shall be processed correctly | P0 |
| FR-LANG-04 | LLM responses shall be generated in the user's preferred language | P0 |
| FR-LANG-05 | UI elements shall be translatable via i18n framework | P1 |

### 6.10 Voice Input

| ID | Requirement | Priority |
|----|------------|----------|
| FR-VOICE-01 | System shall accept voice input via browser-based Web Speech API or Whisper API | P1 |
| FR-VOICE-02 | Voice input shall be transcribed to text and processed as a standard query | P1 |
| FR-VOICE-03 | System shall support voice input in Hindi and English at minimum | P1 |
| FR-VOICE-04 | Voice transcription accuracy shall be ≥ 80% for clear audio input | P1 |

### 6.11 Cron-based Updates

| ID | Requirement | Priority |
|----|------------|----------|
| FR-CRON-01 | System shall execute data ingestion pipelines on a configurable cron schedule (default: every 12 hours) | P0 |
| FR-CRON-02 | Cron jobs shall trigger: scraping → cleaning → MongoDB upsert → embedding regeneration → vector DB index update | P0 |
| FR-CRON-03 | System shall mark opportunities as expired when deadlines have passed | P0 |
| FR-CRON-04 | System shall log all cron executions with status and metrics | P0 |
| FR-CRON-05 | Admin shall be able to trigger manual data refresh from the dashboard | P1 |

---

## 7. Non-Functional Requirements

### 7.1 Scalability

| ID | Requirement | Target |
|----|------------|--------|
| NFR-SCALE-01 | System shall support up to 10,000 concurrent users at launch | 10K CCU |
| NFR-SCALE-02 | MongoDB shall handle up to 500K opportunity documents | 500K docs |
| NFR-SCALE-03 | Vector DB shall support up to 500K embeddings with sub-200ms search | 500K vectors |
| NFR-SCALE-04 | Backend API shall be horizontally scalable (stateless architecture) | Auto-scaling |
| NFR-SCALE-05 | System shall use connection pooling for database connections | Pool size: 50 |

### 7.2 Performance

| ID | Requirement | Target |
|----|------------|--------|
| NFR-PERF-01 | API response time for non-AI endpoints | < 300ms (p95) |
| NFR-PERF-02 | Vector search latency | < 200ms (p95) |
| NFR-PERF-03 | End-to-end query response time (including LLM generation) | < 5 seconds (p95) |
| NFR-PERF-04 | Page load time (frontend) | < 2 seconds (LCP) |
| NFR-PERF-05 | Data ingestion throughput | ≥ 100 records/minute |

### 7.3 Security

| ID | Requirement | Details |
|----|------------|---------|
| NFR-SEC-01 | All API communication shall use HTTPS/TLS 1.2+ | Mandatory |
| NFR-SEC-02 | User passwords shall be hashed using bcrypt (cost factor ≥ 12) | Mandatory |
| NFR-SEC-03 | JWT tokens shall be signed with RS256 and include expiry claims | Mandatory |
| NFR-SEC-04 | API shall enforce rate limiting to prevent abuse | 100 req/min per user |
| NFR-SEC-05 | User PII shall be encrypted at rest (AES-256) | Mandatory |
| NFR-SEC-06 | System shall implement CORS policy restricting origins | Whitelist only |
| NFR-SEC-07 | Admin endpoints shall require role-based access control (RBAC) | Mandatory |
| NFR-SEC-08 | Input validation and sanitization on all user inputs | Mandatory |

### 7.4 Data Freshness

| ID | Requirement | Target |
|----|------------|--------|
| NFR-FRESH-01 | Opportunity data shall be refreshed at least every 24 hours | < 24h staleness |
| NFR-FRESH-02 | Expired opportunities shall be marked/removed within 1 hour of deadline | < 1h lag |
| NFR-FRESH-03 | New schemes from monitored sources shall appear within 24 hours of publication | < 24h delay |

### 7.5 Reliability

| ID | Requirement | Target |
|----|------------|--------|
| NFR-REL-01 | System uptime | ≥ 99.5% |
| NFR-REL-02 | Scraping pipeline shall retry on failure (max 3 retries with exponential backoff) | 3 retries |
| NFR-REL-03 | System shall gracefully degrade if LLM API is unavailable (return raw results without AI summary) | Graceful fallback |
| NFR-REL-04 | Database backups shall be performed daily | Daily automated |
| NFR-REL-05 | System shall implement health check endpoints for all services | /health on each |

---

## 8. Data Requirements

### 8.1 Opportunity Data Schema

```json
{
  "_id": "ObjectId",
  "title": "String (required)",
  "description": "String (required)",
  "category": "Enum [scheme, scholarship, job, subsidy, welfare]",
  "eligibility": {
    "age_min": "Number",
    "age_max": "Number",
    "income_max": "Number",
    "education": ["String"],
    "gender": "Enum [all, male, female, other]",
    "category": ["Enum [general, sc, st, obc, ews]"],
    "location": ["String (state codes)"],
    "disability": "Boolean"
  },
  "benefits": "String",
  "application_process": "String",
  "application_url": "String (URL)",
  "deadline": "Date",
  "source_url": "String (URL)",
  "source_name": "String",
  "status": "Enum [active, expired, upcoming]",
  "tags": ["String"],
  "embedding_id": "String (reference to vector DB)",
  "created_at": "Date",
  "updated_at": "Date",
  "scraped_at": "Date"
}
```

### 8.2 User Data Schema

```json
{
  "_id": "ObjectId",
  "name": "String",
  "email": "String (unique)",
  "phone": "String (unique)",
  "password_hash": "String",
  "profile": {
    "age": "Number",
    "gender": "Enum [male, female, other]",
    "state": "String",
    "district": "String",
    "income": "Number (annual)",
    "education": "String",
    "category": "Enum [general, sc, st, obc, ews]",
    "occupation": "String",
    "disability": "Boolean",
    "interests": ["String"]
  },
  "preferred_language": "String (default: en)",
  "saved_opportunities": ["ObjectId"],
  "query_history": [
    {
      "query": "String",
      "timestamp": "Date",
      "results_count": "Number"
    }
  ],
  "role": "Enum [citizen, admin]",
  "created_at": "Date",
  "updated_at": "Date"
}
```

### 8.3 Pipeline Log Schema

```json
{
  "_id": "ObjectId",
  "pipeline_name": "String",
  "run_type": "Enum [scheduled, manual]",
  "status": "Enum [success, partial, failed]",
  "records_processed": "Number",
  "records_added": "Number",
  "records_updated": "Number",
  "errors": ["String"],
  "duration_ms": "Number",
  "started_at": "Date",
  "completed_at": "Date"
}
```

---

## 9. API Requirements

### 9.1 API Standards

- **Protocol:** RESTful over HTTPS
- **Format:** JSON request/response bodies
- **Authentication:** Bearer JWT tokens
- **Versioning:** URL-based (`/api/v1/`)
- **Error Format:** Standardized error response `{ "error": { "code": "string", "message": "string" } }`

### 9.2 Core Endpoints

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/api/v1/auth/register` | User registration | Public |
| POST | `/api/v1/auth/login` | User login (email/phone + OTP) | Public |
| POST | `/api/v1/auth/refresh` | Refresh JWT token | Authenticated |
| GET | `/api/v1/profile` | Get user profile | Authenticated |
| PUT | `/api/v1/profile` | Update user profile | Authenticated |
| POST | `/api/v1/query` | Submit a natural-language query | Authenticated |
| GET | `/api/v1/opportunities` | List opportunities (with filters) | Authenticated |
| GET | `/api/v1/opportunities/:id` | Get opportunity details | Authenticated |
| POST | `/api/v1/opportunities/save` | Save an opportunity | Authenticated |
| GET | `/api/v1/query/history` | Get user query history | Authenticated |
| POST | `/api/v1/admin/refresh-data` | Trigger manual data ingestion | Admin |
| GET | `/api/v1/admin/pipeline-logs` | View pipeline execution logs | Admin |
| GET | `/api/v1/admin/stats` | View platform analytics | Admin |
| GET | `/api/v1/health` | Health check | Public |

### 9.3 Query Endpoint Detail

**POST `/api/v1/query`**

Request:
```json
{
  "query": "scholarships for engineering students in Tamil Nadu",
  "language": "en",
  "top_k": 5
}
```

Response:
```json
{
  "results": [
    {
      "opportunity_id": "abc123",
      "title": "Tamil Nadu Engineering Scholarship 2026",
      "relevance_score": 0.94,
      "eligibility_summary": "You are eligible based on your profile...",
      "steps": ["Step 1: Visit ...", "Step 2: Fill ...", "Step 3: Upload ..."],
      "deadline": "2026-04-30",
      "source_url": "https://..."
    }
  ],
  "ai_summary": "Based on your profile, I found 5 scholarships...",
  "query_id": "q_789xyz"
}
```

---

## 10. System Constraints

| Constraint | Details |
|-----------|---------|
| **LLM API Dependency** | System depends on external LLM APIs (Gemini / OpenAI); subject to rate limits, latency, and cost |
| **Scraping Fragility** | Government websites may change structure without notice, breaking scrapers |
| **Data Accuracy** | Scraped data accuracy depends on source quality; no guarantee of 100% correctness |
| **Budget** | LLM API costs must be monitored; implement token-level budgeting |
| **Embedding Model Size** | Model must fit within deployment memory constraints (prefer lightweight models for MVP) |
| **Rate Limits** | External API rate limits must be respected during ingestion |
| **Browser Compatibility** | Web Speech API for voice input is not supported in all browsers (Chrome recommended) |

---

## 11. Assumptions

1. Government portals remain publicly accessible and do not block automated scraping within reasonable request rates.
2. Users have access to a modern web browser and internet connectivity.
3. LLM API keys (Gemini / OpenAI) are provisioned and budgeted for the project duration.
4. MongoDB Atlas free/shared tier is sufficient for MVP data volumes.
5. FAISS is acceptable for MVP vector search; migration to Pinecone for production scalability.
6. Users will provide accurate profile information for personalization.
7. Opportunity data from government sources is reasonably well-structured for extraction.
8. The team has access to cloud infrastructure (AWS / GCP / Render / Vercel) for deployment.
9. English and Hindi language models are sufficient for MVP coverage.
10. The platform will operate within Indian regulatory and data protection norms.

---

## 12. Future Enhancements

| Phase | Enhancement | Description |
|-------|------------|-------------|
| **Phase 2** | Mobile App | Native Android app for wider reach in rural areas |
| **Phase 2** | Push Notifications | Alert users when new matching opportunities are published or deadlines approach |
| **Phase 2** | Offline Mode (PWA) | Service worker-based caching for low-connectivity areas |
| **Phase 2** | Document Assistance | Help users fill application forms with pre-populated profile data |
| **Phase 3** | Community Forum | Peer-to-peer knowledge sharing on scheme applications |
| **Phase 3** | Aadhaar Integration | eKYC-based identity verification for streamlined applications |
| **Phase 3** | Analytics Dashboard (Citizen)** | Personal analytics: applications submitted, accepted, savings gained |
| **Phase 3** | Regional Language Expansion | Support for 10+ Indian languages |
| **Phase 3** | Feedback Loop | User feedback on recommendation quality to improve ranking model |
| **Phase 4** | Government Partnership API | Allow government agencies to directly publish schemes via API |
| **Phase 4** | ML-based Ranking | Replace rule-based ranking with a trained ML model using user interaction data |
| **Phase 4** | Chatbot Interface | WhatsApp / Telegram bot for scheme discovery |

---

## Appendix: Priority Legend

| Priority | Meaning |
|----------|---------|
| **P0** | Must-have for MVP launch |
| **P1** | Important, target for v1.0 release |
| **P2** | Nice-to-have, roadmap item |

---

> **Document Owner:** Civora Engineering Team
> **Last Updated:** February 14, 2026
> **Status:** Draft — Hackathon Submission
