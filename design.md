# JanConnectAI — System Design Document

> **AI-Powered Community Opportunity Intelligence Platform**
> Version 1.0 · February 2026

---

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Component-Level Design](#2-component-level-design)
3. [Data Flow Design](#3-data-flow-design)
4. [Database Design](#4-database-design)
5. [API Design](#5-api-design)
6. [Ranking Algorithm Design](#6-ranking-algorithm-design)
7. [RAG Architecture Explanation](#7-rag-architecture-explanation)
8. [Deployment Architecture](#8-deployment-architecture)
9. [Error Handling & Edge Cases](#9-error-handling--edge-cases)
10. [Security Design](#10-security-design)

---

## 1. System Architecture Overview

JanConnectAI follows a **layered architecture** with clear separation of concerns across four tiers. Each layer communicates through well-defined interfaces, enabling independent scaling, testing, and deployment.

### 1.1 Architecture Layers

```
┌─────────────────────────────────────────────────────────────────────┐
│                     PRESENTATION LAYER                              │
│          React / Next.js  ·  Responsive UI  ·  i18n  ·  Voice      │
├─────────────────────────────────────────────────────────────────────┤
│                      BACKEND API LAYER                              │
│   Flask / Node.js  ·  REST API  ·  Auth  ·  Rate Limiting  ·  RBAC │
├──────────────────────┬──────────────────────────────────────────────┤
│     AI / ML LAYER    │           DATA LAYER                         │
│  Embedding Service   │   MongoDB (structured data)                  │
│  Vector Search       │   Vector DB (FAISS / Pinecone)               │
│  Ranking Engine      │   Redis (cache / sessions)                   │
│  RAG Pipeline        │   Object Storage (logs / backups)            │
│  LLM Integration     │                                              │
├──────────────────────┴──────────────────────────────────────────────┤
│                    INGESTION LAYER                                   │
│     Web Scrapers  ·  API Connectors  ·  Cron Scheduler              │
│     Data Cleaner  ·  Deduplicator  ·  Embedding Generator           │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 High-Level Description

| Layer | Responsibility | Technologies |
|-------|---------------|-------------|
| **Presentation** | User interface, form inputs, voice capture, result display, language switching | React / Next.js, Tailwind CSS, Web Speech API |
| **Backend API** | Request routing, authentication, business logic orchestration, response formatting | Flask (Python) or Express (Node.js), JWT, Helmet |
| **AI / ML** | Query embedding, semantic search, personalized ranking, RAG pipeline, LLM invocation | Sentence-Transformers, FAISS / Pinecone, Gemini / OpenAI API |
| **Data** | Persistent storage for users, opportunities, logs; vector index; caching | MongoDB Atlas, FAISS / Pinecone, Redis |
| **Ingestion** | Automated data collection, cleaning, normalization, embedding generation | BeautifulSoup / Scrapy, APScheduler / node-cron, Python scripts |

---

## 2. Component-Level Design

### 2.1 Frontend (React / Next.js)

```
src/
├── components/
│   ├── Auth/           # Login, Register, OTP verification
│   ├── Profile/        # Profile form, completeness indicator
│   ├── Search/         # Query input, voice button, results list
│   ├── Opportunity/    # Opportunity card, detail view, save button
│   ├── Admin/          # Dashboard, pipeline logs, user management
│   └── Common/         # Navbar, Footer, LanguageSwitcher, Loader
├── pages/
│   ├── index.js        # Landing page
│   ├── dashboard.js    # User dashboard with recommendations
│   ├── search.js       # Search + results page
│   ├── profile.js      # Profile management
│   └── admin/          # Admin dashboard pages
├── hooks/              # Custom hooks (useAuth, useQuery, useVoice)
├── context/            # AuthContext, LanguageContext
├── services/           # API client (axios instance)
├── i18n/               # Translation files (en, hi, ta)
└── utils/              # Helpers, formatters, validators
```

**Key Design Decisions:**
- **Next.js SSR/SSG** for SEO-optimized landing pages and fast initial load
- **Client-side rendering** for authenticated dashboard/search pages
- **Tailwind CSS** for rapid, responsive UI development
- **React Context** for auth state and language preference propagation
- **Web Speech API** integration with fallback to Whisper API for voice input
- **i18next** for runtime language switching without page reload

### 2.2 Backend API (Flask / Node.js)

```
server/
├── app.py / index.js          # Application entry point
├── config/
│   ├── settings.py            # Environment-based config
│   └── constants.py           # Enums, defaults
├── routes/
│   ├── auth_routes.py         # /auth/*
│   ├── profile_routes.py      # /profile/*
│   ├── query_routes.py        # /query/*
│   ├── opportunity_routes.py  # /opportunities/*
│   ├── admin_routes.py        # /admin/*
│   └── health_routes.py       # /health
├── middleware/
│   ├── auth_middleware.py     # JWT verification
│   ├── rate_limiter.py        # Request rate limiting
│   └── rbac.py                # Role-based access control
├── services/
│   ├── auth_service.py        # Registration, login, OTP
│   ├── profile_service.py     # CRUD operations
│   ├── query_service.py       # Orchestrates search → rank → RAG
│   ├── embedding_service.py   # Generates query embeddings
│   ├── vector_service.py      # Vector DB search interface
│   ├── ranking_service.py     # Personalized ranking logic
│   ├── rag_service.py         # RAG pipeline orchestration
│   └── llm_service.py         # LLM API abstraction layer
├── models/
│   ├── user.py                # User model
│   ├── opportunity.py         # Opportunity model
│   └── pipeline_log.py        # Pipeline log model
├── utils/
│   ├── validators.py          # Input validation
│   ├── sanitizers.py          # XSS / injection prevention
│   └── logger.py              # Structured logging
└── tests/
    ├── test_auth.py
    ├── test_query.py
    └── test_ranking.py
```

**Key Design Decisions:**
- **Stateless API** — all state in JWT + database; enables horizontal scaling
- **Service layer pattern** — routes delegate to services; services are unit-testable
- **LLM abstraction** — `llm_service.py` wraps Gemini/OpenAI behind a common interface; swap providers without touching business logic
- **Structured logging** — JSON-formatted logs with request IDs for traceability

### 2.3 Data Ingestion Pipeline

```
ingestion/
├── scrapers/
│   ├── base_scraper.py        # Abstract base class
│   ├── myscheme_scraper.py    # MyScheme.gov.in
│   ├── nsp_scraper.py         # National Scholarship Portal
│   ├── ncs_scraper.py         # National Career Service
│   └── state_scrapers/       # State-specific scrapers
├── api_connectors/
│   ├── data_gov_connector.py  # data.gov.in API
│   └── ncs_api_connector.py   # NCS REST API
├── processors/
│   ├── cleaner.py             # Text normalization, HTML stripping
│   ├── deduplicator.py        # Fuzzy duplicate detection
│   ├── normalizer.py          # Schema normalization
│   └── validator.py           # Field-level validation
├── embedding_generator.py     # Batch embedding generation
├── vector_indexer.py          # Upsert embeddings to vector DB
├── pipeline_orchestrator.py   # End-to-end pipeline runner
├── scheduler.py               # Cron job configuration
└── config/
    └── sources.yaml           # Scraping targets configuration
```

**Pipeline Execution Order:**
1. `Scraper / API Connector` → Raw HTML / JSON
2. `Cleaner` → Stripped, normalized text
3. `Normalizer` → Unified opportunity schema
4. `Deduplicator` → Unique records only
5. `Validator` → Schema-valid records
6. `MongoDB Upsert` → Persist to database
7. `Embedding Generator` → Vector representations
8. `Vector Indexer` → Upsert to FAISS / Pinecone

### 2.4 Embedding Generation Service

| Aspect | Specification |
|--------|--------------|
| **Model** | `sentence-transformers/all-MiniLM-L6-v2` (384-d, 80MB) for MVP; `intfloat/multilingual-e5-base` (768-d) for multilingual production |
| **Input** | Concatenated text: `{title} [SEP] {description} [SEP] {eligibility} [SEP] {benefits}` |
| **Output** | Dense float vector (384-d or 768-d) |
| **Batch Size** | 64 documents per batch |
| **Hardware** | CPU-only for MVP; GPU-accelerated for production |
| **Caching** | Embeddings cached by document content hash; regenerated only on content change |

```python
# Pseudocode — Embedding Generation
class EmbeddingService:
    def __init__(self, model_name="all-MiniLM-L6-v2"):
        self.model = SentenceTransformer(model_name)

    def generate(self, opportunity: dict) -> List[float]:
        text = f"{opportunity['title']} [SEP] {opportunity['description']} " \
               f"[SEP] {opportunity['eligibility']} [SEP] {opportunity['benefits']}"
        return self.model.encode(text).tolist()

    def batch_generate(self, opportunities: List[dict]) -> List[List[float]]:
        texts = [self._compose_text(opp) for opp in opportunities]
        return self.model.encode(texts, batch_size=64).tolist()
```

### 2.5 Vector Database

| Feature | FAISS (MVP) | Pinecone (Production) |
|---------|-------------|----------------------|
| **Type** | In-memory, local | Managed cloud service |
| **Index** | `IndexFlatIP` → `IndexIVFFlat` (> 50K vectors) | Serverless / Pod-based |
| **Dimensions** | 384 / 768 | 384 / 768 |
| **Similarity** | Inner Product / L2 | Cosine |
| **Metadata Filtering** | Manual post-filter | Native metadata filters |
| **Persistence** | Serialized to disk (`faiss.write_index`) | Fully managed |
| **Max Scale** | ~1M vectors (single node) | Billions of vectors |

### 2.6 Ranking Engine

The ranking engine re-ranks the Top-K vector search results using a **composite weighted scoring function** that combines semantic relevance with user-specific profile matching and temporal urgency.

> Full algorithm design in [Section 6](#6-ranking-algorithm-design).

### 2.7 RAG Pipeline

The RAG (Retrieval-Augmented Generation) pipeline ensures LLM responses are **grounded in actual opportunity data**, minimizing hallucination risk.

> Full architecture in [Section 7](#7-rag-architecture-explanation).

### 2.8 LLM Integration

```python
# LLM Abstraction Layer
class LLMService:
    def __init__(self, provider="gemini"):
        self.provider = self._init_provider(provider)

    def _init_provider(self, name):
        providers = {
            "gemini": GeminiProvider(api_key=GEMINI_KEY),
            "openai": OpenAIProvider(api_key=OPENAI_KEY),
            "local":  OllamaProvider(model="mistral:7b")
        }
        return providers[name]

    def generate(self, prompt: str, max_tokens: int = 1024) -> str:
        return self.provider.complete(prompt, max_tokens=max_tokens)
```

**Provider Fallback Chain:** Gemini → OpenAI → Local (Ollama)

| Config | Value |
|--------|-------|
| **Temperature** | 0.3 (low creativity, factual) |
| **Max Tokens** | 1024 |
| **Top-P** | 0.9 |
| **Timeout** | 10 seconds |
| **Retry** | 2 retries with exponential backoff |

### 2.9 Cron Job Scheduler

```yaml
# scheduler_config.yaml
jobs:
  - name: full_ingestion
    schedule: "0 */12 * * *"    # Every 12 hours
    task: pipeline_orchestrator.run_full
    timeout: 3600               # 1 hour max

  - name: deadline_check
    schedule: "0 * * * *"       # Every hour
    task: opportunity_service.expire_past_deadlines
    timeout: 300

  - name: embedding_refresh
    schedule: "30 */12 * * *"   # 30 min after ingestion
    task: embedding_generator.regenerate_stale
    timeout: 1800

  - name: health_check
    schedule: "*/5 * * * *"     # Every 5 minutes
    task: health_service.check_all
    timeout: 60
```

**Implementation:** APScheduler (Python) or node-cron (Node.js) with persistent job store backed by MongoDB.

---

## 3. Data Flow Design

### 3.1 Data Ingestion Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│  Gov Portals │     │  Public APIs │     │  RSS / Open Data │
│  (HTML)      │     │  (JSON)      │     │  (XML / CSV)     │
└──────┬───────┘     └──────┬───────┘     └────────┬─────────┘
       │                    │                      │
       ▼                    ▼                      ▼
┌──────────────────────────────────────────────────────────────┐
│                   INGESTION PIPELINE                          │
│                                                              │
│  ┌─────────┐  ┌─────────┐  ┌────────────┐  ┌────────────┐  │
│  │ Scrape  │→ │ Clean   │→ │ Normalize  │→ │ Deduplicate│  │
│  └─────────┘  └─────────┘  └────────────┘  └─────┬──────┘  │
│                                                    │         │
│  ┌────────────────┐  ┌────────────────────┐       │         │
│  │ Generate       │← │ Upsert to MongoDB │←──────┘         │
│  │ Embeddings     │  └────────────────────┘                 │
│  └───────┬────────┘                                          │
│          │                                                   │
│  ┌───────▼────────┐                                          │
│  │ Index in       │                                          │
│  │ Vector DB      │                                          │
│  └────────────────┘                                          │
└──────────────────────────────────────────────────────────────┘
```

**Data Volume Estimates:**
- Initial seed: ~10,000 opportunities
- Daily delta: ~200-500 new/updated records
- Full refresh cycle: ~30 minutes

### 3.2 User Query Flow

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│  User    │     │   Frontend   │     │   Backend    │
│          │     │              │     │   API        │
└────┬─────┘     └──────┬───────┘     └──────┬───────┘
     │                  │                    │
     │  Enter query     │                    │
     │  (text/voice)    │                    │
     │─────────────────>│                    │
     │                  │  POST /api/v1/query│
     │                  │───────────────────>│
     │                  │                    │
     │                  │              ┌─────▼─────────────┐
     │                  │              │ 1. Validate input  │
     │                  │              │ 2. Load user       │
     │                  │              │    profile         │
     │                  │              └─────┬─────────────┘
     │                  │                    │
     │                  │              ┌─────▼─────────────┐
     │                  │              │ 3. Generate query  │
     │                  │              │    embedding       │
     │                  │              └─────┬─────────────┘
     │                  │                    │
     │                  │              ┌─────▼─────────────┐
     │                  │              │ 4. Vector search   │
     │                  │              │    (Top-K=20)      │
     │                  │              └─────┬─────────────┘
     │                  │                    │
     │                  │              ┌─────▼─────────────┐
     │                  │              │ 5. Fetch full docs │
     │                  │              │    from MongoDB    │
     │                  │              └─────┬─────────────┘
     │                  │                    │
     │                  │              ┌─────▼─────────────┐
     │                  │              │ 6. Personalized    │
     │                  │              │    ranking (Top-5) │
     │                  │              └─────┬─────────────┘
     │                  │                    │
     │                  │              ┌─────▼─────────────┐
     │                  │              │ 7. RAG: Build      │
     │                  │              │    prompt + LLM    │
     │                  │              └─────┬─────────────┘
     │                  │                    │
     │                  │  JSON response     │
     │                  │<───────────────────│
     │  Display results │                    │
     │<─────────────────│                    │
```

### 3.3 RAG Retrieval + Generation Flow

```
┌─────────────┐       ┌──────────────┐       ┌──────────────┐
│ User Query  │──────>│ Embedding    │──────>│ Vector DB    │
│ "scholarships│       │ Service      │       │ (FAISS)      │
│  for BC     │       │              │       │              │
│  students"  │       │ query → 384-d│       │ Top-20 IDs   │
└─────────────┘       └──────────────┘       └──────┬───────┘
                                                     │
                      ┌──────────────┐               │
                      │ MongoDB      │<──────────────┘
                      │ Fetch full   │  Retrieve full documents
                      │ documents    │  by embedding_id
                      └──────┬───────┘
                             │
                      ┌──────▼───────┐
                      │ Ranking      │
                      │ Engine       │  Apply profile-based scoring
                      │ (Top-5)      │
                      └──────┬───────┘
                             │
                      ┌──────▼───────────────────────────────────┐
                      │ PROMPT CONSTRUCTION                       │
                      │                                          │
                      │ System: "You are , an AI assistant │
                      │ that helps Indian citizens find govt     │
                      │ schemes. Answer ONLY from the provided   │
                      │ context."                                │
                      │                                          │
                      │ Context: [Top-5 opportunity documents]   │
                      │                                          │
                      │ User Profile: {age, location, income...} │
                      │                                          │
                      │ Query: "scholarships for BC students"    │
                      │                                          │
                      │ Instructions: "For each relevant scheme: │
                      │  1. State if user is eligible            │
                      │  2. Explain in simple language           │
                      │  3. Give step-by-step application guide  │
                      │  4. Include deadlines and URLs"          │
                      └──────────────┬───────────────────────────┘
                                     │
                              ┌──────▼───────┐
                              │   LLM API    │
                              │  (Gemini /   │
                              │   OpenAI)    │
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │  Structured  │
                              │  Response    │
                              │  + Sources   │
                              └──────────────┘
```

---

## 4. Database Design

### 4.1 MongoDB Collections

#### 4.1.1 `users` Collection

```javascript
{
  _id: ObjectId("..."),
  name: "Ramesh Kumar",
  email: "ramesh@example.com",          // unique, indexed
  phone: "+919876543210",               // unique, indexed
  password_hash: "$2b$12$...",
  profile: {
    age: 22,
    gender: "male",
    state: "Tamil Nadu",
    district: "Chennai",
    income: 250000,                     // annual in INR
    education: "undergraduate",         // enum: below_10th, 10th, 12th, undergraduate, postgraduate, doctorate
    category: "obc",                    // enum: general, sc, st, obc, ews
    occupation: "student",
    disability: false,
    interests: ["education", "technology", "scholarships"]
  },
  preferred_language: "en",
  saved_opportunities: [ObjectId("..."), ObjectId("...")],
  query_history: [
    {
      query_id: "q_abc123",
      query: "engineering scholarships",
      timestamp: ISODate("2026-02-14T10:30:00Z"),
      results_count: 5
    }
  ],
  role: "citizen",                      // enum: citizen, admin
  is_active: true,
  created_at: ISODate("2026-01-15T08:00:00Z"),
  updated_at: ISODate("2026-02-14T10:30:00Z")
}

// Indexes
db.users.createIndex({ email: 1 }, { unique: true })
db.users.createIndex({ phone: 1 }, { unique: true })
db.users.createIndex({ "profile.state": 1, "profile.category": 1 })
db.users.createIndex({ role: 1 })
```

#### 4.1.2 `opportunities` Collection

```javascript
{
  _id: ObjectId("..."),
  scheme_id: "TN-SCHOL-2026-001",       // external ID for dedup
  title: "Tamil Nadu BC/MBC Scholarship for Engineering Students",
  description: "Financial assistance for backward class students pursuing engineering degrees...",
  category: "scholarship",              // enum: scheme, scholarship, job, subsidy, welfare
  eligibility: {
    age_min: 17,
    age_max: 30,
    income_max: 300000,
    education: ["undergraduate", "postgraduate"],
    gender: "all",
    categories: ["obc", "sc", "st"],
    locations: ["TN"],                  // state codes; empty = national
    disability: null                    // null = not a criterion
  },
  benefits: "Full tuition fee waiver + INR 5,000/month stipend",
  application_process: "1. Visit tnscholarships.gov.in\n2. Register with Aadhaar\n3. Upload income certificate...",
  application_url: "https://tnscholarships.gov.in/apply",
  deadline: ISODate("2026-04-30T23:59:59Z"),
  source_url: "https://tnscholarships.gov.in/bc-engineering-2026",
  source_name: "TN Scholarships Portal",
  status: "active",                     // enum: active, expired, upcoming
  tags: ["engineering", "backward class", "tamil nadu", "tuition fee"],
  embedding_id: "emb_tn_schol_001",     // reference to vector DB
  content_hash: "sha256:abc123...",     // for change detection
  popularity_score: 0.85,              // normalized [0, 1]
  created_at: ISODate("2026-01-20T00:00:00Z"),
  updated_at: ISODate("2026-02-10T12:00:00Z"),
  scraped_at: ISODate("2026-02-14T06:00:00Z")
}

// Indexes
db.opportunities.createIndex({ scheme_id: 1 }, { unique: true })
db.opportunities.createIndex({ status: 1, category: 1 })
db.opportunities.createIndex({ "eligibility.locations": 1 })
db.opportunities.createIndex({ deadline: 1 })
db.opportunities.createIndex({ content_hash: 1 })
db.opportunities.createIndex({ status: 1, deadline: 1 })
```

#### 4.1.3 `pipeline_logs` Collection

```javascript
{
  _id: ObjectId("..."),
  pipeline_name: "myscheme_scraper",
  run_id: "run_20260214_0600",
  run_type: "scheduled",                // enum: scheduled, manual
  status: "success",                    // enum: success, partial, failed
  source: "https://myscheme.gov.in",
  metrics: {
    records_scraped: 150,
    records_new: 12,
    records_updated: 23,
    records_unchanged: 115,
    records_failed: 0,
    embeddings_generated: 35,
    vectors_upserted: 35
  },
  errors: [],
  duration_ms: 45230,
  started_at: ISODate("2026-02-14T06:00:00Z"),
  completed_at: ISODate("2026-02-14T06:00:45Z")
}

// Indexes
db.pipeline_logs.createIndex({ pipeline_name: 1, started_at: -1 })
db.pipeline_logs.createIndex({ status: 1 })
```

### 4.2 Vector DB Storage Logic

```
Vector Entry Structure:
┌─────────────────────────────────────────────────────┐
│  ID:        "emb_tn_schol_001"                      │
│  Vector:    [0.023, -0.145, 0.892, ..., 0.034]      │
│             (384 dimensions)                         │
│  Metadata:  {                                        │
│               "mongo_id": "ObjectId(...)",           │
│               "category": "scholarship",             │
│               "status": "active",                    │
│               "locations": ["TN"],                   │
│               "deadline": "2026-04-30"               │
│             }                                        │
└─────────────────────────────────────────────────────┘
```

**Indexing Strategy:**

| Scale | Index Type | Details |
|-------|-----------|---------|
| < 10K vectors | `IndexFlatIP` | Exact search, no training needed |
| 10K–100K vectors | `IndexIVFFlat` | Inverted file index, `nlist=100`, `nprobe=10` |
| 100K–1M vectors | `IndexIVFPQ` | Product quantization for memory efficiency |
| > 1M vectors | Pinecone (managed) | Auto-scaling serverless index |

**Sync Protocol:**
1. On opportunity insert/update → generate embedding → upsert to vector DB
2. On opportunity deletion/expiry → delete vector from index
3. Full re-index triggered weekly via cron as consistency check
4. Content hash comparison to avoid unnecessary re-embedding

---

## 5. API Design

### 5.1 Authentication APIs

#### `POST /api/v1/auth/register`

```
Request:
{
  "name": "Ramesh Kumar",
  "email": "ramesh@example.com",       // or phone
  "phone": "+919876543210",
  "password": "SecureP@ss123"
}

Response (201):
{
  "success": true,
  "message": "OTP sent to +919876543210",
  "user_id": "usr_abc123",
  "requires_otp": true
}
```

#### `POST /api/v1/auth/login`

```
Request:
{
  "email": "ramesh@example.com",
  "password": "SecureP@ss123"
}

Response (200):
{
  "success": true,
  "token": "eyJhbGciOiJSUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJSUzI1NiIs...",
  "expires_in": 604800,
  "user": {
    "id": "usr_abc123",
    "name": "Ramesh Kumar",
    "role": "citizen",
    "profile_complete": true
  }
}
```

### 5.2 Profile APIs

#### `GET /api/v1/profile`

```
Headers: Authorization: Bearer <JWT>

Response (200):
{
  "success": true,
  "profile": {
    "name": "Ramesh Kumar",
    "age": 22,
    "gender": "male",
    "state": "Tamil Nadu",
    "district": "Chennai",
    "income": 250000,
    "education": "undergraduate",
    "category": "obc",
    "occupation": "student",
    "disability": false,
    "interests": ["education", "technology"],
    "preferred_language": "en",
    "completeness": 95
  }
}
```

#### `PUT /api/v1/profile`

```
Headers: Authorization: Bearer <JWT>

Request:
{
  "age": 23,
  "income": 280000,
  "interests": ["education", "technology", "research"]
}

Response (200):
{
  "success": true,
  "message": "Profile updated successfully",
  "completeness": 100
}
```

### 5.3 Query API

#### `POST /api/v1/query`

```
Headers: Authorization: Bearer <JWT>

Request:
{
  "query": "scholarships for engineering students in Tamil Nadu",
  "language": "en",
  "top_k": 5,
  "filters": {                          // optional
    "category": "scholarship",
    "status": "active"
  }
}

Response (200):
{
  "success": true,
  "query_id": "q_789xyz",
  "results": [
    {
      "opportunity_id": "opp_abc123",
      "title": "Tamil Nadu BC/MBC Engineering Scholarship",
      "category": "scholarship",
      "relevance_score": 0.94,
      "profile_match": 0.91,
      "final_score": 0.93,
      "eligibility_summary": "✅ You are likely eligible. Your age (22), education (undergraduate), category (OBC), and location (Tamil Nadu) match the criteria. Your annual income (₹2.5L) is below the ₹3L threshold.",
      "steps": [
        "Visit tnscholarships.gov.in",
        "Click 'New Registration' and enter your Aadhaar number",
        "Fill the application form with your academic details",
        "Upload income certificate and community certificate",
        "Submit before April 30, 2026"
      ],
      "deadline": "2026-04-30",
      "days_remaining": 75,
      "application_url": "https://tnscholarships.gov.in/apply",
      "source_url": "https://tnscholarships.gov.in/bc-engineering-2026"
    }
    // ... more results
  ],
  "ai_summary": "I found 5 active scholarships matching your profile. The top match is the TN BC/MBC Engineering Scholarship — you appear eligible based on your OBC category and undergraduate status. The deadline is April 30, 2026 (75 days away). I recommend applying to the top 3 immediately.",
  "total_matches": 5,
  "processing_time_ms": 2340
}
```

### 5.4 Admin APIs

#### `POST /api/v1/admin/refresh-data`

```
Headers: Authorization: Bearer <ADMIN_JWT>

Request:
{
  "source": "all",                     // or specific: "myscheme", "nsp"
  "force_reembed": false
}

Response (202):
{
  "success": true,
  "message": "Data refresh initiated",
  "job_id": "job_refresh_20260214",
  "estimated_duration_min": 30
}
```

#### `GET /api/v1/admin/stats`

```
Headers: Authorization: Bearer <ADMIN_JWT>

Response (200):
{
  "success": true,
  "stats": {
    "total_users": 1523,
    "active_users_7d": 487,
    "total_opportunities": 12450,
    "active_opportunities": 8920,
    "queries_today": 2341,
    "avg_response_time_ms": 2100,
    "pipeline_health": "healthy",
    "last_ingestion": "2026-02-14T06:00:45Z",
    "vector_count": 12450
  }
}
```

### 5.5 Error Response Format

```
Response (4xx / 5xx):
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",        // machine-readable
    "message": "Age must be between 1 and 120",
    "field": "profile.age",            // optional
    "request_id": "req_abc123"         // for debugging
  }
}
```

**Standard Error Codes:**

| Code | HTTP Status | Description |
|------|------------|-------------|
| `VALIDATION_ERROR` | 400 | Invalid input data |
| `UNAUTHORIZED` | 401 | Missing or invalid JWT |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `RATE_LIMITED` | 429 | Too many requests |
| `LLM_UNAVAILABLE` | 503 | LLM API temporarily unavailable |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

---

## 6. Ranking Algorithm Design

### 6.1 Overview

The ranking engine takes the Top-K results from vector search and re-ranks them using a **weighted composite score** that combines four factors:

```
Final Score = w₁·S_semantic + w₂·S_profile + w₃·S_urgency + w₄·S_popularity
```

Default weights: `w₁=0.35, w₂=0.40, w₃=0.15, w₄=0.10`

### 6.2 Scoring Components

#### 6.2.1 Semantic Similarity Score ($S_{semantic}$)

The cosine similarity between the user query embedding and the opportunity embedding, as returned by the vector database. Already normalized to $[0, 1]$.

$$S_{semantic} = \frac{\vec{q} \cdot \vec{d}}{|\vec{q}| \cdot |\vec{d}|}$$

#### 6.2.2 Profile Match Score ($S_{profile}$)

A rule-based score evaluating how well the user's profile matches the opportunity's eligibility criteria.

```python
def compute_profile_match(user_profile, opportunity_eligibility) -> float:
    scores = []

    # Age match (binary)
    if elig.age_min <= user.age <= elig.age_max:
        scores.append(1.0)
    else:
        scores.append(0.0)

    # Income match (binary)
    if elig.income_max is None or user.income <= elig.income_max:
        scores.append(1.0)
    else:
        scores.append(0.0)

    # Location match (binary)
    if not elig.locations or user.state_code in elig.locations:
        scores.append(1.0)
    else:
        scores.append(0.0)

    # Education match (binary)
    if not elig.education or user.education in elig.education:
        scores.append(1.0)
    else:
        scores.append(0.0)

    # Category match (binary)
    if not elig.categories or user.category in elig.categories:
        scores.append(1.0)
    else:
        scores.append(0.0)

    # Gender match (binary)
    if elig.gender == "all" or elig.gender == user.gender:
        scores.append(1.0)
    else:
        scores.append(0.0)

    return sum(scores) / len(scores)   # Mean of criteria matches
```

$$S_{profile} = \frac{1}{n} \sum_{i=1}^{n} \text{match}_i \quad \text{where } \text{match}_i \in \{0, 1\}$$

#### 6.2.3 Deadline Urgency Score ($S_{urgency}$)

Opportunities with approaching deadlines receive a higher urgency score, creating a **soft time-decay boost**.

$$S_{urgency} = \begin{cases} 0 & \text{if } d > 90 \text{ days} \\ 1 - \frac{d}{90} & \text{if } 7 \leq d \leq 90 \\ 1.0 & \text{if } d < 7 \text{ days} \end{cases}$$

Where $d$ = days until deadline.

```python
def compute_urgency(deadline: datetime) -> float:
    days_remaining = (deadline - datetime.now()).days
    if days_remaining < 0:
        return 0.0          # Expired
    elif days_remaining < 7:
        return 1.0           # Urgent
    elif days_remaining <= 90:
        return 1 - (days_remaining / 90)
    else:
        return 0.0           # Not urgent
```

#### 6.2.4 Popularity Score ($S_{popularity}$)

Pre-computed score based on historical engagement data (views, saves, application clicks). Normalized to $[0, 1]$.

For MVP: static value based on source authority (e.g., central government schemes = 0.9, state = 0.7, other = 0.5).

### 6.3 Final Score Computation

```python
def compute_final_score(semantic_score, profile, eligibility, deadline, popularity):
    W = {"semantic": 0.35, "profile": 0.40, "urgency": 0.15, "popularity": 0.10}

    s_profile = compute_profile_match(profile, eligibility)
    s_urgency = compute_urgency(deadline)

    final = (
        W["semantic"]   * semantic_score +
        W["profile"]    * s_profile +
        W["urgency"]    * s_urgency +
        W["popularity"] * popularity
    )
    return round(final, 4)
```

### 6.4 Ranking Pipeline

```
Vector Search (Top-20) → Fetch Full Docs → Score Each → Sort Descending → Return Top-5
```

| Step | Input | Output |
|------|-------|--------|
| 1. Vector Search | Query embedding | Top-20 IDs + similarity scores |
| 2. Document Fetch | 20 IDs | 20 full opportunity documents |
| 3. Profile Scoring | User profile + 20 eligibility objects | 20 profile match scores |
| 4. Urgency Scoring | 20 deadlines | 20 urgency scores |
| 5. Composite Score | All sub-scores + weights | 20 final scores |
| 6. Sort + Truncate | 20 scored results | Top-5 ranked results |

---

## 7. RAG Architecture Explanation

### 7.1 What is RAG?

**Retrieval-Augmented Generation (RAG)** is a technique that enhances LLM responses by grounding them in retrieved, factual context — rather than relying solely on the model's training data. This dramatically reduces hallucination and ensures responses reflect actual, current opportunity data.

### 7.2 RAG Pipeline Steps

```
Step 1: QUERY EMBEDDING
────────────────────────
User Query: "scholarships for BC students in Tamil Nadu"
    │
    ▼
Embedding Model (all-MiniLM-L6-v2)
    │
    ▼
Query Vector: [0.023, -0.145, 0.892, ..., 0.034]  (384-d)


Step 2: TOP-K RETRIEVAL
────────────────────────
Query Vector ──► Vector DB (FAISS)
    │
    ▼
Top-K=20 most similar opportunity vectors
    │
    ▼
Fetch full documents from MongoDB (by embedding_id)


Step 3: RANKING + FILTERING
────────────────────────────
20 documents ──► Ranking Engine
    │
    ▼
Top-5 ranked, profile-matched opportunities


Step 4: CONTEXT INJECTION
─────────────────────────
Build structured prompt:

┌───────────────────────────────────────────────────────┐
│  SYSTEM PROMPT                                        │
│  "You are JanConnectAI, an AI assistant helping Indian      │
│   citizens discover government opportunities.         │
│   Answer ONLY based on the provided context.          │
│   If unsure, say 'I don't have enough information.'"  │
├───────────────────────────────────────────────────────┤
│  CONTEXT (Top-5 opportunities as structured text)     │
│                                                       │
│  [1] Title: TN BC/MBC Engineering Scholarship         │
│      Eligibility: Age 17-30, Income < 3L, OBC/SC/ST  │
│      Benefits: Full tuition + ₹5000/month             │
│      Deadline: 2026-04-30                             │
│      URL: https://tnscholarships.gov.in/...           │
│                                                       │
│  [2] Title: ...                                       │
│  [3-5] ...                                            │
├───────────────────────────────────────────────────────┤
│  USER PROFILE                                         │
│  Age: 22, State: Tamil Nadu, Income: ₹2.5L,          │
│  Education: Undergraduate, Category: OBC              │
├───────────────────────────────────────────────────────┤
│  USER QUERY                                           │
│  "scholarships for BC students in Tamil Nadu"         │
├───────────────────────────────────────────────────────┤
│  OUTPUT INSTRUCTIONS                                  │
│  For each relevant opportunity:                       │
│  1. ✅/❌ Eligibility verdict with reasoning          │
│  2. Plain-language summary of benefits                │
│  3. Step-by-step application instructions             │
│  4. Deadline + days remaining                         │
│  5. Direct application link                           │
│  Respond in: {user's preferred language}              │
└───────────────────────────────────────────────────────┘


Step 5: LLM GENERATION
──────────────────────
Prompt ──► Gemini / OpenAI API
    │
    ▼
Structured, grounded response with sources
```

### 7.3 Anti-Hallucination Safeguards

| Safeguard | Implementation |
|-----------|---------------|
| **Context-only grounding** | System prompt explicitly instructs: "Answer ONLY from provided context" |
| **Source attribution** | Every claim is linked to a source URL |
| **Confidence signaling** | LLM must say "I'm not sure" when context is insufficient |
| **Low temperature** | `temperature=0.3` reduces creative/hallucinated outputs |
| **Structured output** | JSON-mode or function-calling ensures consistent format |
| **Post-processing** | Validate that generated URLs match retrieved source URLs |

### 7.4 Prompt Template

```python
RAG_PROMPT = """
You are JanConnectAI, an AI assistant that helps Indian citizens discover and apply for
government schemes, scholarships, jobs, and welfare programs.

RULES:
- Answer ONLY based on the CONTEXT provided below
- If no relevant information exists in the context, say so honestly
- Always include source URLs for verification
- Use simple, easy-to-understand language
- Respond in {language}

---
CONTEXT:
{context_documents}

---
USER PROFILE:
- Age: {age} | Gender: {gender} | State: {state}
- Income: ₹{income}/year | Education: {education}
- Category: {category} | Occupation: {occupation}

---
USER QUERY: {query}

---
For each relevant opportunity, provide:
1. Eligibility Check: ✅ Eligible / ❌ Not Eligible / ⚠️ Partially Eligible (with reason)
2. What You Get: Benefits in simple terms
3. How to Apply: Step-by-step instructions
4. Deadline: Date + days remaining
5. Apply Here: Direct URL
"""
```

---

## 8. Deployment Architecture

### 8.1 Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        CDN (Cloudflare)                          │
│                  Static assets + SSL termination                 │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│                    FRONTEND HOSTING                               │
│              Vercel / Netlify / AWS Amplify                       │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Next.js (SSR + Static)  ·  React  ·  Tailwind CSS         │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────┬───────────────────────────────────────┘
                           │ HTTPS
┌──────────────────────────▼───────────────────────────────────────┐
│                    BACKEND HOSTING                                │
│              Render / Railway / AWS ECS / GCP Cloud Run           │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │  API Server  │  │  API Server  │  │  Ingestion Worker      │ │
│  │  (Instance 1)│  │  (Instance 2)│  │  (Cron + Scrapers)     │ │
│  └──────┬───────┘  └──────┬───────┘  └────────┬───────────────┘ │
│         │                 │                    │                  │
│         └────────┬────────┘                    │                  │
│                  │                             │                  │
└──────────────────┼─────────────────────────────┼─────────────────┘
                   │                             │
    ┌──────────────┼─────────────────────────────┼──────────────┐
    │              │         DATA LAYER          │              │
    │  ┌───────────▼──────────┐  ┌──────────────▼───────────┐  │
    │  │  MongoDB Atlas       │  │  FAISS (in-process)      │  │
    │  │  (M0 Free → M10)    │  │  or Pinecone (managed)   │  │
    │  │                     │  │                           │  │
    │  │  • users            │  │  • 384-d vectors          │  │
    │  │  • opportunities    │  │  • metadata filters       │  │
    │  │  • pipeline_logs    │  │  • ~100K vectors          │  │
    │  └─────────────────────┘  └───────────────────────────┘  │
    │                                                          │
    │  ┌─────────────────────┐  ┌───────────────────────────┐  │
    │  │  Redis (optional)   │  │  LLM API (External)       │  │
    │  │  Upstash / ElastiC  │  │  Gemini / OpenAI          │  │
    │  │  • Session cache    │  │  • API key-based          │  │
    │  │  • Rate limiting    │  │  • Rate-limited           │  │
    │  └─────────────────────┘  └───────────────────────────┘  │
    └──────────────────────────────────────────────────────────┘
```

### 8.2 Hosting Recommendations

| Component | MVP (Free/Low-cost) | Production |
|-----------|-------------------|------------|
| **Frontend** | Vercel (free tier) | Vercel Pro / AWS CloudFront |
| **Backend API** | Render (free tier) | AWS ECS Fargate / GCP Cloud Run |
| **Ingestion Worker** | Same as backend | Separate container / AWS Lambda |
| **MongoDB** | Atlas M0 (free, 512MB) | Atlas M10+ (dedicated) |
| **Vector DB** | FAISS (in-process) | Pinecone Serverless |
| **Redis** | Upstash (free tier) | AWS ElastiCache |
| **LLM API** | Gemini free tier | Gemini Pro / OpenAI GPT-4 |
| **Domain + SSL** | Cloudflare (free) | Cloudflare Pro |

### 8.3 Scalability Considerations

| Concern | Strategy |
|---------|----------|
| **API Throughput** | Horizontally scale stateless API servers behind load balancer |
| **Database Load** | MongoDB Atlas auto-scaling; read replicas for query-heavy workloads |
| **Vector Search** | FAISS → Pinecone migration when exceeding single-node capacity |
| **LLM Costs** | Token budgeting per user; response caching for repeated queries |
| **Scraping Scale** | Distributed scraping with queue (Redis/RabbitMQ) for concurrent sources |
| **Cold Starts** | Keep minimum 1 warm instance on Render/Cloud Run |

### 8.4 CI/CD Pipeline

```
GitHub Push → GitHub Actions → Lint + Test → Build → Deploy

Stages:
1. Lint (ESLint + Pylint)
2. Unit Tests (pytest / jest)
3. Integration Tests
4. Docker Build
5. Deploy to Staging
6. Smoke Tests
7. Deploy to Production (manual approval)
```

---

## 9. Error Handling & Edge Cases

### 9.1 Error Handling Strategy

| Layer | Error Type | Handling |
|-------|-----------|----------|
| **Frontend** | Network timeout | Retry with exponential backoff; show "Retrying..." |
| **Frontend** | API 4xx | Display user-friendly error message |
| **Frontend** | API 5xx | Show "Something went wrong. Please try again." |
| **Backend** | Invalid input | Return 400 with specific validation errors |
| **Backend** | JWT expired | Return 401; frontend triggers token refresh |
| **Backend** | Rate limit exceeded | Return 429 with `Retry-After` header |
| **AI Layer** | LLM API timeout (>10s) | Return raw ranked results without AI summary |
| **AI Layer** | LLM API rate limited | Queue request; retry after backoff |
| **AI Layer** | LLM hallucination detected | Post-process: strip unverified URLs, add disclaimer |
| **AI Layer** | Empty vector search results | Return "No matching opportunities found" + suggestion to broaden query |
| **Ingestion** | Scraper blocked / CAPTCHA | Log error; skip source; alert admin; retry next cycle |
| **Ingestion** | Website structure changed | Log parsing errors; alert admin; disable scraper until fixed |
| **Ingestion** | Duplicate detection failure | Content-hash based dedup as secondary check |
| **Database** | MongoDB connection failure | Connection pool retry; circuit breaker pattern |
| **Database** | Vector DB index corruption | Trigger full re-index from MongoDB source of truth |

### 9.2 Edge Cases

| Edge Case | Handling |
|-----------|----------|
| User profile is empty | Disable personalized ranking; use semantic search only; prompt user to complete profile |
| Query is too vague (e.g., "help me") | Return popular/trending opportunities; ask user to be more specific |
| Query is in unsupported language | Auto-detect language; translate to English for search; respond in detected language |
| No opportunities match all criteria | Relax filters progressively (remove location → remove income → remove category) until results found |
| Deadline has passed between scrape and query | Real-time deadline check before returning results; mark as expired |
| User submits identical query repeatedly | Return cached response (TTL: 1 hour) |
| Extremely long query (>500 chars) | Truncate to first 500 characters; warn user |
| Malicious input (SQL injection, XSS) | Input sanitization at API gateway; parameterized queries |
| LLM returns response in wrong language | Post-process language detection; re-prompt if mismatch |
| Voice input with heavy accent/noise | Return transcription for user confirmation before searching |

### 9.3 Graceful Degradation Matrix

```
Full Service:       Vector Search → Ranking → RAG → AI Response
│
├─ LLM Down:        Vector Search → Ranking → Raw Results (no AI summary)
│
├─ Vector DB Down:  MongoDB text search → Ranking → Raw Results
│
├─ MongoDB Down:    Service Unavailable (503) — critical dependency
│
└─ All AI Down:     MongoDB text search → Unranked results → Basic display
```

---

## 10. Security Design

### 10.1 Authentication & Authorization

```
┌─────────┐          ┌──────────┐          ┌──────────────┐
│  Client  │──Login──>│ Auth API  │──Verify──>│  User DB     │
│          │          │          │           │  (bcrypt)    │
│          │<──JWT────│          │           └──────────────┘
│          │          └──────────┘
│          │
│          │──Request──>┌──────────────┐
│          │ (Bearer)   │  API Gateway │
│          │            │              │
│          │            │  1. Verify JWT│
│          │            │  2. Check RBAC│
│          │            │  3. Rate limit│
│          │            │  4. Forward   │
│          │<──Response──│              │
└─────────┘            └──────────────┘
```

### 10.2 Security Controls

| Domain | Control | Implementation |
|--------|---------|---------------|
| **Transport** | TLS 1.2+ encryption | HTTPS enforced on all endpoints |
| **Authentication** | Password hashing | bcrypt with cost factor 12 |
| **Authentication** | Token-based auth | JWT signed with RS256; 7-day expiry |
| **Authentication** | OTP verification | 6-digit OTP; 5-minute expiry; max 3 attempts |
| **Authorization** | Role-based access | RBAC middleware: `citizen`, `admin` roles |
| **Input Security** | Validation | JSON schema validation on all inputs |
| **Input Security** | Sanitization | DOMPurify (frontend), bleach (backend) |
| **Input Security** | Injection prevention | Parameterized queries; no string concatenation in DB queries |
| **Rate Limiting** | API abuse prevention | 100 requests/min per user; 10 queries/min for AI endpoints |
| **Data Protection** | PII encryption | AES-256 encryption at rest for email, phone, income |
| **Data Protection** | Minimal data collection | Only collect data necessary for personalization |
| **API Security** | CORS | Whitelist: `JanConnectAI.in`, `localhost:3000` |
| **API Security** | Helmet/security headers | `X-Content-Type-Options`, `X-Frame-Options`, `CSP` |
| **Secrets** | Key management | Environment variables; never in code; `.env` in `.gitignore` |
| **Logging** | Audit trail | Log all admin actions, auth events, data modifications |
| **Monitoring** | Anomaly detection | Alert on: >5 failed logins, unusual API patterns, scraper failures |

### 10.3 Data Privacy Compliance

| Principle | Implementation |
|-----------|---------------|
| **Consent** | Explicit consent checkbox at registration |
| **Purpose Limitation** | Data used only for scheme matching + personalization |
| **Data Minimization** | Optional fields clearly marked; platform works with partial profiles |
| **Right to Deletion** | User can request account + data deletion via settings |
| **Data Retention** | Query history retained 90 days; auto-purged after |
| **Third-party Sharing** | No user data shared with external parties |

### 10.4 LLM Security

| Risk | Mitigation |
|------|-----------|
| **Prompt Injection** | Sanitize user input before injecting into prompt; use delimiters |
| **Data Leakage** | System prompt instructs LLM to never reveal other users' data |
| **Sensitive Data in Prompts** | Hash/mask PII before sending to LLM; send only category (e.g., "income < 3L" not "income = 2,50,000") |
| **API Key Exposure** | Server-side only LLM calls; keys never sent to frontend |
| **Cost Attacks** | Per-user token budget; max prompt length enforcement |

---

## Appendix A: Technology Stack Summary

| Component | Technology | Justification |
|-----------|-----------|---------------|
| Frontend | Next.js + React + Tailwind | SSR, fast dev, responsive |
| Backend | Flask (Python) | Rich ML/NLP ecosystem, rapid prototyping |
| Database | MongoDB Atlas | Flexible schema for varied opportunity data |
| Vector DB | FAISS (MVP) → Pinecone | Free local search → managed production scale |
| Embeddings | Sentence-Transformers | Open-source, fast, multilingual models available |
| LLM | Gemini API / OpenAI API | State-of-the-art generation quality |
| Scraping | BeautifulSoup + Requests | Lightweight, sufficient for gov portals |
| Scheduling | APScheduler | Python-native, persistent job store |
| Auth | JWT (PyJWT) + bcrypt | Industry standard, stateless |
| Cache | Redis (Upstash) | Low-latency caching + rate limiting |
| Hosting | Vercel + Render | Free tiers for MVP |
| CI/CD | GitHub Actions | Free for public repos |

---

## Appendix B: Glossary

| Term | Definition |
|------|-----------|
| **RAG** | Retrieval-Augmented Generation — technique that grounds LLM responses in retrieved documents |
| **FAISS** | Facebook AI Similarity Search — library for efficient similarity search of dense vectors |
| **Embedding** | Dense vector representation of text, capturing semantic meaning |
| **Top-K** | The K most similar results from vector search |
| **Cosine Similarity** | Measure of similarity between two vectors (1 = identical, 0 = orthogonal) |
| **OTP** | One-Time Password — time-limited code for authentication |
| **JWT** | JSON Web Token — compact token format for stateless authentication |
| **RBAC** | Role-Based Access Control — permissions tied to user roles |
| **Sentence-Transformer** | Neural network model that converts sentences into fixed-size embeddings |
| **Cron** | Time-based job scheduler for recurring tasks |

---

> **Document Owner:** JanConnectAI Engineering Team
> **Last Updated:** February 14, 2026
> **Status:** Draft — Hackathon Submission
