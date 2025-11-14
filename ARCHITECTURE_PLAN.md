# LawDB: Court Opinion IRAC Analysis System
## Architecture & Implementation Plan

## Overview
A system to ingest, analyze, and semantically search court opinions from US federal courts (Supreme Court, Circuit Courts of Appeals, District Courts) using IRAC (Issue, Rule, Application, Conclusion) analysis framework.

---

## 1. Data Sources

### Primary Sources
1. **CourtListener API** (Recommended - Free & Comprehensive)
   - Run by Free Law Project
   - REST API with opinions from all federal courts
   - Metadata: citations, judges, dates, docket info
   - ~13M+ opinions available
   - Free tier available

2. **Harvard Caselaw Access Project (CAP)**
   - 6.5M+ published US court decisions
   - Historical coverage (1600s to 2018)
   - API available with registration
   - Free for research use

3. **SCOTUS Opinions** (Supreme Court specific)
   - Official supreme court website
   - Most recent opinions
   - Can be scraped or use CourtListener

### Data Coverage Strategy
- **Supreme Court**: Complete coverage (highest priority)
- **Circuit Courts**: All 13 circuits (high priority)
- **District Courts**: Major districts initially, expand as needed (medium priority)

---

## 2. System Architecture

### Components

```
┌─────────────────────────────────────────────────────────────┐
│                     Data Ingestion Layer                     │
├─────────────────────────────────────────────────────────────┤
│  - API Clients (CourtListener, CAP)                         │
│  - Opinion Parser (extract text, metadata)                  │
│  - Deduplication & Validation                               │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│                      IRAC Extraction Layer                   │
├─────────────────────────────────────────────────────────────┤
│  - LLM-based Extraction (Claude/GPT-4)                      │
│  - Structured Prompting System                              │
│  - Validation & Quality Control                             │
│  - Citation Extraction                                       │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│                    Embedding & Storage Layer                 │
├─────────────────────────────────────────────────────────────┤
│  - Embedding Generation (sentence-transformers)             │
│  - Vector Database (Qdrant/Weaviate)                        │
│  - Relational DB (PostgreSQL)                               │
│  - Document Store (full opinion text)                       │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│                      Search & Query Layer                    │
├─────────────────────────────────────────────────────────────┤
│  - Semantic Search (vector similarity)                      │
│  - Filtered Search (court, date, citations)                │
│  - IRAC Component Search                                    │
│  - API Endpoints (FastAPI/Flask)                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Technology Stack

### Core Technologies

**Backend**
- Python 3.11+
- FastAPI (REST API)
- Pydantic (data validation)
- SQLAlchemy (ORM)

**Databases**
- PostgreSQL (structured data, metadata)
- Qdrant or Weaviate (vector database for semantic search)
- Optional: Redis (caching)

**ML/NLP**
- Sentence Transformers (embeddings)
- OpenAI/Anthropic API (IRAC extraction)
- Alternative: Local LLMs (Llama, Mistral) for cost efficiency

**Data Processing**
- Celery (async task queue)
- Apache Airflow or Prefect (workflow orchestration)

**Infrastructure**
- Docker & Docker Compose
- Poetry or uv (dependency management)

---

## 4. Database Schema

### PostgreSQL Tables

```sql
-- Court Opinions
opinions (
    id UUID PRIMARY KEY,
    case_name TEXT NOT NULL,
    citation TEXT,  -- e.g., "410 U.S. 113"
    court_type VARCHAR(50),  -- 'supreme', 'appellate', 'district'
    court_identifier TEXT,  -- e.g., 'scotus', '9th-circuit', 'sdny'
    decision_date DATE,
    judges TEXT[],
    docket_number TEXT,
    opinion_text TEXT,
    opinion_url TEXT,
    source VARCHAR(50),  -- 'courtlistener', 'cap', etc.
    metadata JSONB,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
)

-- IRAC Analysis
irac_analyses (
    id UUID PRIMARY KEY,
    opinion_id UUID REFERENCES opinions(id),
    issue TEXT NOT NULL,
    rule TEXT NOT NULL,
    application TEXT NOT NULL,
    conclusion TEXT NOT NULL,
    legal_topics TEXT[],  -- e.g., ['constitutional', 'fourth amendment']
    cited_cases TEXT[],
    cited_statutes TEXT[],
    confidence_score FLOAT,  -- extraction quality score
    extraction_model VARCHAR(100),
    created_at TIMESTAMP
)

-- Legal Citations (for cross-referencing)
citations (
    id UUID PRIMARY KEY,
    opinion_id UUID REFERENCES opinions(id),
    cited_opinion_id UUID REFERENCES opinions(id),
    citation_text TEXT,
    citation_type VARCHAR(50),  -- 'case', 'statute', 'regulation'
    context TEXT  -- surrounding text
)

-- Search Metadata (for analytics)
searches (
    id UUID PRIMARY KEY,
    query_text TEXT,
    search_type VARCHAR(50),
    results_count INT,
    timestamp TIMESTAMP
)
```

### Vector Database (Qdrant) Collections

```python
# Collection 1: Full Opinion Embeddings
opinions_collection:
    - vector: [1536 dimensions]  # embedding
    - payload: {
        opinion_id: UUID,
        case_name: str,
        court_type: str,
        decision_date: str,
        citation: str
      }

# Collection 2: IRAC Component Embeddings
irac_collection:
    - vector: [1536 dimensions]
    - payload: {
        irac_id: UUID,
        opinion_id: UUID,
        component_type: str,  # 'issue', 'rule', 'application', 'conclusion'
        component_text: str,
        case_name: str,
        legal_topics: List[str]
      }

# Collection 3: Legal Topic Embeddings
topics_collection:
    - vector: [1536 dimensions]
    - payload: {
        topic: str,
        related_opinions: List[UUID],
        statutes: List[str]
      }
```

---

## 5. IRAC Extraction Strategy

### Approach 1: LLM-based Extraction (Primary)

**Prompt Engineering**
```python
SYSTEM_PROMPT = """
You are a legal analysis expert. Analyze the court opinion and extract
the IRAC components:
1. Issue: The legal question(s) presented
2. Rule: The applicable law, statutes, or legal principles
3. Application: How the court applied the law to the facts
4. Conclusion: The court's decision and reasoning

Provide structured JSON output with high fidelity to the original text.
"""
```

**Process**
1. Chunk long opinions if needed (>100k tokens)
2. Extract IRAC components with structured output
3. Validate completeness and coherence
4. Extract citations using regex + NER
5. Assign confidence score

### Approach 2: Fine-tuned Model (Future Enhancement)
- Collect labeled IRAC examples
- Fine-tune smaller model (Llama 3, Mistral)
- Deploy for cost-effective bulk processing

### Citation Extraction
- Use regex patterns for citations (e.g., `\d+ U.S. \d+`)
- Named Entity Recognition for case names
- Link to existing opinions in database

---

## 6. Semantic Search Implementation

### Embedding Strategy
1. **Full Opinion Embeddings**
   - Use sentence-transformers (e.g., `all-MiniLM-L6-v2` or legal-specific model)
   - Store in vector DB

2. **IRAC Component Embeddings**
   - Separate embeddings for each component
   - Enables targeted search (e.g., "find similar legal rules")

3. **Hybrid Search**
   - Combine vector similarity with metadata filters
   - Example: "4th Amendment cases from 9th Circuit after 2020"

### Search Features
- **Semantic Similarity**: Find cases with similar legal reasoning
- **IRAC Component Search**: Search within specific IRAC parts
- **Citation Network**: Find citing/cited cases
- **Topic Clustering**: Group cases by legal topic
- **Multi-filter**: Court + date range + legal area

---

## 7. API Design

### Endpoints

```
POST   /api/opinions/ingest          # Ingest new opinion
GET    /api/opinions/{id}            # Get opinion details
GET    /api/opinions/{id}/irac       # Get IRAC analysis
POST   /api/search/semantic          # Semantic search
POST   /api/search/irac              # Search by IRAC component
GET    /api/search/citations/{id}    # Find related cases
POST   /api/batch/process            # Batch process opinions
GET    /api/stats                    # System statistics
```

### Example Search Request
```json
POST /api/search/semantic
{
  "query": "search and seizure without warrant in vehicle",
  "filters": {
    "court_type": ["supreme", "appellate"],
    "date_range": {"start": "2010-01-01", "end": "2024-12-31"},
    "legal_topics": ["fourth amendment"]
  },
  "search_type": "full_opinion",  // or "issue", "rule", "application"
  "limit": 10
}
```

---

## 8. Implementation Phases

### Phase 1: Foundation (Weeks 1-2)
- [ ] Set up project structure & dependencies
- [ ] Configure PostgreSQL & Qdrant
- [ ] Implement CourtListener API client
- [ ] Basic data models & schemas
- [ ] Docker setup

### Phase 2: Data Ingestion (Weeks 2-3)
- [ ] Opinion parser & text extraction
- [ ] Metadata extraction & validation
- [ ] Deduplication logic
- [ ] Batch ingestion scripts
- [ ] Initial dataset: 1000 Supreme Court opinions

### Phase 3: IRAC Extraction (Weeks 3-4)
- [ ] LLM integration (Claude/GPT-4)
- [ ] IRAC extraction prompts
- [ ] Citation extraction
- [ ] Quality validation
- [ ] Error handling & retry logic

### Phase 4: Semantic Search (Weeks 4-5)
- [ ] Embedding generation pipeline
- [ ] Vector database integration
- [ ] Search algorithms implementation
- [ ] Similarity ranking
- [ ] Hybrid search (vector + filters)

### Phase 5: API & Interface (Week 5-6)
- [ ] FastAPI endpoints
- [ ] Request validation
- [ ] Response formatting
- [ ] API documentation (OpenAPI)
- [ ] Basic CLI for testing

### Phase 6: Testing & Optimization (Week 6-7)
- [ ] Unit tests
- [ ] Integration tests
- [ ] Performance optimization
- [ ] Caching strategy
- [ ] Load testing

### Phase 7: Expansion (Week 7+)
- [ ] Add appellate court opinions
- [ ] Add district court opinions
- [ ] Advanced analytics features
- [ ] Web UI (optional)
- [ ] Export capabilities

---

## 9. Key Considerations

### Legal Accuracy
- Preserve original text fidelity in IRAC extraction
- Include confidence scores
- Allow manual review/correction
- Version IRAC analyses

### Scalability
- Process ~10,000 opinions/day target
- Async processing with Celery
- Batch operations for efficiency
- Caching for frequent queries

### Cost Management
- Use local embeddings models (sentence-transformers)
- Cache LLM results
- Consider local LLMs for bulk processing
- Rate limiting on API calls

### Data Quality
- Validation at each stage
- Deduplication
- Handle incomplete/malformed opinions
- Track processing errors

### Legal Ethics & Usage
- Respect data source terms of service
- Attribution to sources
- Research/educational use focus
- No unauthorized redistribution

---

## 10. Technology Alternatives

### Vector Databases
| Option | Pros | Cons |
|--------|------|------|
| **Qdrant** | Fast, Python-native, easy setup | Newer project |
| **Weaviate** | Feature-rich, good docs | More complex |
| **Pinecone** | Managed service, scalable | Cost, vendor lock-in |
| **pgvector** | PostgreSQL extension, simple | Less optimized |

**Recommendation**: Start with Qdrant for simplicity

### LLM for IRAC
| Option | Pros | Cons |
|--------|------|------|
| **Claude (Anthropic)** | Excellent at long documents, accurate | Cost, API required |
| **GPT-4** | Strong reasoning, structured output | Cost, rate limits |
| **Local (Llama 3, Mistral)** | No API cost, privacy | Requires GPU, lower quality |

**Recommendation**: Claude for initial development, consider local for scaling

---

## 11. Project Structure

```
LawDB/
├── src/
│   ├── api/                  # FastAPI application
│   │   ├── routes/
│   │   ├── dependencies.py
│   │   └── main.py
│   ├── ingestion/           # Data ingestion
│   │   ├── clients/         # API clients
│   │   ├── parsers/
│   │   └── validators.py
│   ├── irac/                # IRAC extraction
│   │   ├── extractor.py
│   │   ├── prompts.py
│   │   └── validators.py
│   ├── search/              # Search functionality
│   │   ├── embeddings.py
│   │   ├── vector_db.py
│   │   └── query.py
│   ├── models/              # Data models
│   │   ├── opinion.py
│   │   ├── irac.py
│   │   └── citation.py
│   ├── db/                  # Database
│   │   ├── migrations/
│   │   ├── session.py
│   │   └── init.py
│   └── utils/
│       ├── citations.py
│       └── text_processing.py
├── tests/
├── scripts/                 # Utility scripts
│   ├── ingest_scotus.py
│   └── generate_embeddings.py
├── docs/
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── .env.example
├── pyproject.toml
├── README.md
└── ARCHITECTURE_PLAN.md
```

---

## 12. Next Steps

1. **Validate Plan**: Review with stakeholders
2. **Set up Development Environment**: Docker, databases
3. **Select Data Source**: CourtListener API registration
4. **Select LLM Provider**: API keys for Claude/OpenAI
5. **Create Proof of Concept**: Process 10 Supreme Court cases end-to-end
6. **Iterate & Scale**: Based on POC results

---

## Questions for Discussion

1. **LLM Provider**: Preference for Claude vs GPT-4 vs local models?
2. **Scale Priority**: How many opinions to target initially?
3. **Court Priority**: Supreme Court first, or include appellate courts from start?
4. **Cost Budget**: API costs for LLM processing?
5. **Interface**: CLI only, or also web UI?
6. **Hosting**: Local development or cloud deployment?
7. **Additional Features**:
   - Export to PDF/Word with IRAC breakdown?
   - Visualization of citation networks?
   - Timeline views of legal development?

---

**Status**: Draft for review and discussion
**Version**: 1.0
**Date**: 2024-11-14
