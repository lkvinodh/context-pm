# Architecture Deep Dive

> **Technical architecture and design decisions for production-grade AI agent systems**

This document provides comprehensive architectural documentation covering system design, component interactions, data flow patterns, and the rationale behind key technical decisions.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Architectural Patterns](#architectural-patterns)
3. [Component Architecture](#component-architecture)
4. [Data Flow & Processing](#data-flow--processing)
5. [Single-Agent vs Multi-Agent Architecture](#single-agent-vs-multi-agent-architecture)
6. [Integration Architecture](#integration-architecture)
7. [State Management & Persistence](#state-management--persistence)
8. [Security Architecture](#security-architecture)
9. [Scalability & Performance](#scalability--performance)
10. [Design Decisions & Trade-offs](#design-decisions--trade-offs)

---

## System Overview

### High-Level Architecture

The system follows a **layered architecture** with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Presentation Layer                           │
│                (CLI / Web UI / REST API)                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    Agent Orchestration Layer                     │
│  ┌──────────────────┐              ┌──────────────────────┐    │
│  │  Single Agent    │              │  Multi-Agent         │    │
│  │  (Monolithic)    │              │  (Orchestrated)      │    │
│  └────────┬─────────┘              └──────────┬───────────┘    │
│           │                                    │                 │
│           │         ┌──────────────────────────▼──────┐         │
│           │         │    Orchestrator                 │         │
│           │         │  (Intent Classification)        │         │
│           │         └──────────────────────────┬──────┘         │
│           │                                    │                 │
│           │         ┌──────────────────────────▼──────┐         │
│           │         │   Specialized Agents:           │         │
│           │         │   - Research Agent              │         │
│           │         │   - Content Generator           │         │
│           │         │   - Technical Advisor           │         │
│           │         │   - Business Analyst            │         │
│           │         └─────────────────────────────────┘         │
└────────────────────────────┬───────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                     Retrieval Layer                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐   │
│  │   Semantic     │  │    Keyword     │  │    Hybrid      │   │
│  │    Search      │  │    Search      │  │   Re-ranker    │   │
│  │ (Embeddings)   │  │     (BM25)     │  │                │   │
│  └────────────────┘  └────────────────┘  └────────────────┘   │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐   │
│  │            Context Optimizer                            │   │
│  │      (Token Budget Management)                          │   │
│  └────────────────────────────────────────────────────────┘   │
└────────────────────────────┬───────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                   Data Integration Layer                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   GitHub     │  │  Confluence  │  │    Jira      │         │
│  │   Adapter    │  │   Adapter    │  │   Adapter    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐   │
│  │           Unified Adapter Interface                     │   │
│  │    (Authentication, Rate Limiting, Error Handling)      │   │
│  └────────────────────────────────────────────────────────┘   │
└────────────────────────────┬───────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    Persistence Layer                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Sync State  │  │   Document   │  │  Embeddings  │         │
│  │   (JSON)     │  │    Cache     │  │    Cache     │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐   │
│  │      Incremental Sync Engine                            │   │
│  │  (Hash-based Change Detection, Atomic Writes)           │   │
│  └────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Core Design Principles

1. **Separation of Concerns**: Each layer has a single, well-defined responsibility
2. **Loose Coupling**: Components communicate through interfaces, not implementations
3. **Fail-Fast**: Errors are caught early and propagated clearly
4. **Graceful Degradation**: System continues operating with reduced functionality when components fail
5. **Observability**: Comprehensive logging and metrics at every layer
6. **Cost Consciousness**: Every API call tracked, optimized for minimal token usage

---

## Architectural Patterns

### 1. Adapter Pattern (Data Integration)

**Problem**: Need to integrate with multiple heterogeneous enterprise systems with different APIs, authentication mechanisms, and data formats.

**Solution**: Unified adapter interface with source-specific implementations.

```python
# Base interface
class BaseAdapter(ABC):
    @abstractmethod
    def authenticate(self) -> bool: pass
    
    @abstractmethod
    def fetch_documents(self) -> List[Document]: pass
    
    @abstractmethod
    def get_document_by_id(self, doc_id: str) -> Optional[Document]: pass

# Source-specific implementations
class GitHubAdapter(BaseAdapter):
    # GitHub-specific OAuth2 implementation
    pass

class ConfluenceAdapter(BaseAdapter):
    # Confluence-specific Bearer token implementation
    pass
```

**Benefits**:
- Add new data sources without modifying existing code
- Consistent error handling across all sources
- Easy to test (mock individual adapters)
- Clear contract for what each adapter must provide

### 2. Strategy Pattern (Agent Selection)

**Problem**: Need to switch between single-agent and multi-agent architectures dynamically based on task type or user preference.

**Solution**: Agent strategy interface with pluggable implementations.

```python
class AgentStrategy(ABC):
    @abstractmethod
    def process_query(self, query: str, context: List[Document]) -> Response: pass

class SingleAgentStrategy(AgentStrategy):
    def process_query(self, query: str, context: List[Document]) -> Response:
        return self.agent.query(query, context)

class MultiAgentStrategy(AgentStrategy):
    def process_query(self, query: str, context: List[Document]) -> Response:
        intent = self.orchestrator.classify(query)
        specialist = self.get_specialist(intent)
        return specialist.process(query, context)
```

**Benefits**:
- Easy A/B testing between architectures
- User can choose architecture per-query
- Benchmark mode can run both and compare
- New strategies can be added without touching existing code

### 3. Template Method Pattern (Document Processing)

**Problem**: All data sources follow similar processing pipeline but differ in specific steps.

**Solution**: Base class defines pipeline skeleton, subclasses implement specific steps.

```python
class BaseAdapter:
    def fetch_documents(self) -> List[Document]:
        # Template method defining pipeline
        self.authenticate()
        raw_data = self._fetch_raw_data()
        documents = self._parse_documents(raw_data)
        validated = self._validate_documents(documents)
        return validated
    
    @abstractmethod
    def _fetch_raw_data(self): pass
    
    @abstractmethod
    def _parse_documents(self, raw): pass
```

**Benefits**:
- Consistent processing across all sources
- Shared validation and error handling
- Easy to understand data flow

### 4. Repository Pattern (Data Persistence)

**Problem**: Need to abstract away storage details (file system, database) from business logic.

**Solution**: Repository interface provides CRUD operations without exposing storage mechanism.

```python
class DocumentRepository:
    def save(self, doc: Document) -> None: pass
    def get(self, doc_id: str) -> Optional[Document]: pass
    def get_all(self) -> List[Document]: pass
    def delete(self, doc_id: str) -> None: pass
    def exists(self, doc_id: str) -> bool: pass
```

**Benefits**:
- Easy to swap storage backend (JSON → SQLite → PostgreSQL)
- Simplified testing (in-memory repository for tests)
- Clear data access patterns

---

## Component Architecture

### 1. Agent Orchestration Layer

#### Single-Agent Architecture

```
User Query → Single Agent → LLM (Claude Sonnet) → Response
                ↑
                │
          Context Documents
```

**Characteristics**:
- One generalist agent handles all query types
- Direct path from query to response
- Single LLM call per query
- No routing overhead

**Component Details**:
```python
class SingleAgent:
    def __init__(self):
        self.client = anthropic.Anthropic()
        self.model = "claude-sonnet-4.6"
        self.system_prompt = self._load_generalist_prompt()
    
    def query(self, user_query: str, context: List[Document]) -> Response:
        # Build context string
        context_text = self._format_context(context)
        
        # Single API call
        response = self.client.messages.create(
            model=self.model,
            messages=[{
                "role": "user",
                "content": f"Context: {context_text}\n\nQuery: {user_query}"
            }]
        )
        
        return Response(
            text=response.content[0].text,
            tokens=response.usage.input_tokens + response.usage.output_tokens
        )
```

#### Multi-Agent Architecture

```
User Query → Orchestrator → Intent Classification
                │
                ├─→ Research Agent (analytical queries)
                ├─→ Content Generator (writing tasks)
                ├─→ Technical Advisor (technical analysis)
                └─→ Business Analyst (business questions)
                       ↓
                    Response
```

**Characteristics**:
- Lightweight orchestrator (Haiku) routes to specialists
- Each specialist has domain-specific system prompt
- Model selection based on task complexity
- Fallback to single-agent on routing failures

**Component Details**:
```python
class Orchestrator:
    def __init__(self):
        self.client = anthropic.Anthropic()
        self.model = "claude-haiku-4-5"  # Fast, cheap routing
        self.specialists = {
            "research": ResearchAgent(),
            "content": ContentGenerator(),
            "technical": TechnicalAdvisor(),
            "business": BusinessAnalyst()
        }
    
    def process(self, query: str, context: List[Document]) -> Response:
        # Step 1: Classify intent (cheap Haiku call)
        intent = self._classify_intent(query)
        
        # Step 2: Route to specialist
        specialist = self.specialists.get(intent, self.specialists["research"])
        
        # Step 3: Specialist processes with domain-specific prompt
        return specialist.process(query, context)
    
    def _classify_intent(self, query: str) -> str:
        prompt = f"""Classify this query into one category:
- research: Information lookup, fact-finding
- content: Writing emails, presentations, documents
- technical: Code review, architecture, feasibility
- business: Analysis, risks, business decisions

Query: {query}

Return ONLY the category name."""
        
        response = self.client.messages.create(
            model=self.model,
            max_tokens=50,
            messages=[{"role": "user", "content": prompt}]
        )
        
        return response.content[0].text.strip().lower()

class ContentGenerator:
    def __init__(self):
        self.client = anthropic.Anthropic()
        self.model = "claude-sonnet-4.6"
        self.system_prompt = """You are a professional content writer.
        
Create clear, concise, and professional content:
- Emails: Professional tone, clear subject, actionable
- Presentations: Structured slides, bullet points, visual suggestions
- Reports: Executive summary, sections, conclusions

Always match the tone to the audience."""
    
    def process(self, query: str, context: List[Document]) -> Response:
        # Specialist has focused prompt for better quality
        response = self.client.messages.create(
            model=self.model,
            system=self.system_prompt,
            messages=[{
                "role": "user",
                "content": f"Context: {context}\n\nTask: {query}"
            }]
        )
        
        return Response(text=response.content[0].text)
```

### 2. Retrieval Layer

#### Three-Stage Hybrid Retrieval

```
Query
  │
  ├─→ Semantic Search (Embeddings) ──┐
  │                                   │
  ├─→ Keyword Search (BM25) ─────────┤
  │                                   │
  └─→ Recency Boost ─────────────────┤
                                      │
                                      ▼
                              Hybrid Re-ranker
                                      │
                                      ▼
                             Context Optimizer
                              (Token Budget)
                                      │
                                      ▼
                              Ranked Documents
```

**Stage 1: Semantic Search**

```python
class SemanticSearch:
    def __init__(self):
        self.model = SentenceTransformer('all-MiniLM-L6-v2')
        self.embeddings_cache = {}
    
    def search(self, query: str, documents: List[Document], top_k: int = 50) -> List[ScoredDocument]:
        # Embed query
        query_embedding = self.model.encode(query)
        
        # Compute similarity with all documents
        scores = []
        for doc in documents:
            doc_embedding = self._get_embedding(doc)
            similarity = cosine_similarity(query_embedding, doc_embedding)
            scores.append((doc, similarity))
        
        # Return top-k
        scores.sort(key=lambda x: x[1], reverse=True)
        return scores[:top_k]
    
    def _get_embedding(self, doc: Document):
        if doc.id not in self.embeddings_cache:
            self.embeddings_cache[doc.id] = self.model.encode(doc.content)
        return self.embeddings_cache[doc.id]
```

**Stage 2: Keyword Search (BM25)**

```python
from rank_bm25 import BM25Okapi

class KeywordSearch:
    def __init__(self):
        self.tokenizer = self._create_tokenizer()
        self.bm25 = None
        self.documents = []
    
    def index(self, documents: List[Document]):
        self.documents = documents
        tokenized_corpus = [self.tokenizer(doc.content) for doc in documents]
        self.bm25 = BM25Okapi(tokenized_corpus)
    
    def search(self, query: str, top_k: int = 50) -> List[ScoredDocument]:
        tokenized_query = self.tokenizer(query)
        scores = self.bm25.get_scores(tokenized_query)
        
        # Get top-k
        top_indices = np.argsort(scores)[-top_k:][::-1]
        return [(self.documents[i], scores[i]) for i in top_indices]
    
    @staticmethod
    def _create_tokenizer():
        def tokenize(text):
            # Technical term preservation
            tokens = text.lower().split()
            # Keep acronyms, camelCase, etc.
            return tokens
        return tokenize
```

**Stage 3: Hybrid Re-ranking**

```python
class HybridReranker:
    def __init__(self, semantic_weight: float = 0.7, keyword_weight: float = 0.3):
        self.semantic_weight = semantic_weight
        self.keyword_weight = keyword_weight
    
    def rerank(
        self, 
        semantic_results: List[ScoredDocument],
        keyword_results: List[ScoredDocument],
        recency_boost: bool = True
    ) -> List[ScoredDocument]:
        # Normalize scores to [0, 1]
        semantic_scores = self._normalize_scores(semantic_results)
        keyword_scores = self._normalize_scores(keyword_results)
        
        # Merge scores
        combined = {}
        for doc, score in semantic_scores.items():
            combined[doc.id] = score * self.semantic_weight
        
        for doc, score in keyword_scores.items():
            if doc.id in combined:
                combined[doc.id] += score * self.keyword_weight
            else:
                combined[doc.id] = score * self.keyword_weight
        
        # Apply recency boost
        if recency_boost:
            combined = self._apply_recency_boost(combined)
        
        # Sort and return
        ranked = sorted(combined.items(), key=lambda x: x[1], reverse=True)
        return [(doc_id, score) for doc_id, score in ranked]
    
    def _apply_recency_boost(self, scores: Dict) -> Dict:
        # Exponential decay: newer documents get higher boost
        now = datetime.now()
        for doc_id, score in scores.items():
            doc = self._get_document(doc_id)
            age_days = (now - doc.updated_at).days
            # Boost recent docs (decay half-life = 30 days)
            recency_factor = 0.5 ** (age_days / 30)
            scores[doc_id] = score * (1 + 0.2 * recency_factor)
        return scores
```

**Stage 4: Context Optimizer**

```python
class ContextOptimizer:
    def __init__(self, max_tokens: int = 12000):
        self.max_tokens = max_tokens
        self.tokenizer = tiktoken.encoding_for_model("claude-sonnet-4.6")
    
    def optimize(self, documents: List[ScoredDocument]) -> List[Document]:
        """Select documents that fit within token budget"""
        selected = []
        total_tokens = 0
        
        for doc, score in documents:
            # Count tokens
            doc_tokens = len(self.tokenizer.encode(doc.content))
            
            # Check if fits in budget
            if total_tokens + doc_tokens <= self.max_tokens:
                selected.append(doc)
                total_tokens += doc_tokens
            else:
                # Try truncation
                remaining_tokens = self.max_tokens - total_tokens
                if remaining_tokens > 500:  # Worth including partial
                    truncated = self._truncate_document(doc, remaining_tokens)
                    selected.append(truncated)
                break
        
        return selected
    
    def _truncate_document(self, doc: Document, max_tokens: int) -> Document:
        """Intelligently truncate document to fit token budget"""
        tokens = self.tokenizer.encode(doc.content)
        truncated_tokens = tokens[:max_tokens]
        truncated_content = self.tokenizer.decode(truncated_tokens)
        
        # Create copy with truncated content
        return Document(
            id=doc.id,
            title=doc.title,
            content=truncated_content + "\n\n[Content truncated...]",
            source=doc.source,
            url=doc.url,
            metadata={**doc.metadata, "truncated": True},
            created_at=doc.created_at,
            updated_at=doc.updated_at,
            content_hash=doc.content_hash
        )
```

### 3. Data Integration Layer

#### Unified Adapter Interface

All adapters implement the same interface but handle source-specific details internally:

```python
class DataIntegrationLayer:
    def __init__(self, config):
        self.adapters = [
            GitHubAdapter(config),
            ConfluenceAdapter(config),
            JiraAdapter(config),
            OfficeDocsAdapter(config)
        ]
        self.sync_engine = SyncEngine(config)
    
    def sync_all(self, force: bool = False) -> Dict:
        """Sync from all data sources"""
        return self.sync_engine.sync(self.adapters, force=force)
    
    def get_all_documents(self) -> List[Document]:
        """Retrieve all cached documents"""
        return self.sync_engine.get_all_documents()
    
    def health_check(self) -> Dict[str, bool]:
        """Check health of all data sources"""
        return {
            adapter.__class__.__name__: adapter.health_check()
            for adapter in self.adapters
        }
```

#### Authentication Strategy

Different sources require different authentication:

| Source | Auth Method | Token Type | Refresh Strategy |
|--------|-------------|------------|------------------|
| GitHub Enterprise | OAuth2 | Personal Access Token | Manual rotation (90 days) |
| Confluence | Bearer | API Token | No expiration (until revoked) |
| Jira | OAuth2 | Access Token + Refresh Token | Automatic refresh (60 min expiry) |
| Office Docs | File System | N/A | N/A |

```python
class AuthenticationManager:
    def __init__(self):
        self.token_store = TokenStore()
    
    def authenticate_github(self, config) -> bool:
        # PAT-based authentication (no refresh needed)
        return self._verify_github_token(config.github.token)
    
    def authenticate_jira(self, config) -> bool:
        # OAuth2 with automatic token refresh
        access_token = self.token_store.get("jira_access_token")
        
        if self._is_token_expired(access_token):
            refresh_token = self.token_store.get("jira_refresh_token")
            access_token = self._refresh_jira_token(refresh_token)
            self.token_store.save("jira_access_token", access_token)
        
        return self._verify_jira_token(access_token)
```

---

## Data Flow & Processing

### End-to-End Query Flow

```
1. User submits query
   │
2. Retrieve relevant documents (Hybrid Search)
   │
   ├─→ Semantic search (top-50)
   ├─→ Keyword search (top-50)
   └─→ Merge & re-rank → top-10
   │
3. Optimize context (Token Budget)
   │
4. Route to agent
   │
   ├─→ Single-Agent: Direct LLM call
   │   └─→ Response
   │
   └─→ Multi-Agent: Orchestrator classifies
       │
       ├─→ Research Agent (analytical)
       ├─→ Content Generator (writing)
       ├─→ Technical Advisor (technical)
       └─→ Business Analyst (business)
           └─→ Response
   │
5. Format response
   │
6. Log metrics (tokens, cost, latency)
   │
7. Return to user
```

### Sync Flow (Incremental)

```
1. For each data source:
   │
2. Authenticate
   │
3. Fetch document list
   │
4. For each document:
   │
   ├─→ Compute content hash (SHA-256)
   │
   ├─→ Compare with stored hash
   │   │
   │   ├─→ Match: Skip (unchanged)
   │   └─→ Different: Process
   │       │
   │       ├─→ Parse content
   │       ├─→ Generate embeddings
   │       ├─→ Save to cache
   │       └─→ Update sync state
   │
5. Atomic save of sync state
   │
6. Log statistics
```

---

## Single-Agent vs Multi-Agent Architecture

### Architectural Comparison

| Aspect | Single-Agent | Multi-Agent |
|--------|-------------|-------------|
| **Latency** | Low (1 API call) | Higher (2-3 API calls) |
| **Cost** | Moderate (larger model) | Lower (smaller specialist models) |
| **Complexity** | Simple | Complex (orchestration + specialists) |
| **Quality** | Good (general responses) | Better (specialized responses) |
| **Maintainability** | Easy | Harder (multiple prompts to tune) |
| **Token Usage** | Lower (single context) | Higher (context duplication) |
| **Best For** | Analytical queries | Creative tasks |

### When Each Architecture Wins

**Single-Agent Wins:**
- Research queries (direct retrieval → response)
- Low latency requirements (< 5s)
- Simple deployment (one model, one prompt)
- Token efficiency critical

**Multi-Agent Wins:**
- Writing tasks (emails, presentations)
- High-volume scenarios (cost matters more than latency)
- Quality-critical outputs (specialized prompts)
- Mixed workload with clear task categories

### Hybrid Approach (Recommended)

```python
class HybridAgentSystem:
    def __init__(self):
        self.single_agent = SingleAgent()
        self.multi_agent = MultiAgentSystem()
        self.router = TaskRouter()
    
    def process(self, query: str, context: List[Document]) -> Response:
        # Classify task type
        task_type = self.router.classify(query)
        
        # Route to appropriate architecture
        if task_type in ["research", "technical_qa"]:
            # Fast single-agent for analytical tasks
            return self.single_agent.query(query, context)
        else:
            # Multi-agent for creative tasks
            return self.multi_agent.process(query, context)
```

---

## Integration Architecture

### Enterprise System Integration Patterns

#### 1. Circuit Breaker Pattern

Prevent cascading failures when external services are down:

```python
class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, timeout: int = 60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if self._should_attempt_reset():
                self.state = "HALF_OPEN"
            else:
                raise CircuitBreakerOpenError("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
    
    def _on_success(self):
        self.failure_count = 0
        self.state = "CLOSED"
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
    
    def _should_attempt_reset(self):
        return time.time() - self.last_failure_time >= self.timeout
```

#### 2. Retry with Exponential Backoff

```python
class RetryStrategy:
    def __init__(self, max_retries: int = 3, base_delay: float = 1.0):
        self.max_retries = max_retries
        self.base_delay = base_delay
    
    def execute(self, func, *args, **kwargs):
        for attempt in range(self.max_retries):
            try:
                return func(*args, **kwargs)
            except (ConnectionError, TimeoutError) as e:
                if attempt == self.max_retries - 1:
                    raise
                
                delay = self.base_delay * (2 ** attempt)
                logger.warning(f"Attempt {attempt + 1} failed, retrying in {delay}s")
                time.sleep(delay)
```

#### 3. Rate Limiting

```python
class RateLimiter:
    def __init__(self, requests_per_minute: int = 60):
        self.requests_per_minute = requests_per_minute
        self.requests = []
    
    def acquire(self):
        now = time.time()
        
        # Remove old requests (> 1 minute ago)
        self.requests = [req for req in self.requests if now - req < 60]
        
        # Check if we've hit the limit
        if len(self.requests) >= self.requests_per_minute:
            sleep_time = 60 - (now - self.requests[0])
            logger.info(f"Rate limit reached, sleeping {sleep_time:.2f}s")
            time.sleep(sleep_time)
        
        self.requests.append(now)
```

---

## State Management & Persistence

### Sync State Schema

```json
{
  "github:abc123": {
    "content_hash": "sha256_hash_here",
    "last_synced": "2026-04-19T10:30:00Z",
    "file_path": "./data/documents/github_abc123.json",
    "size_bytes": 15420,
    "error_count": 0
  },
  "confluence:456def": {
    "content_hash": "sha256_hash_here",
    "last_synced": "2026-04-19T10:31:15Z",
    "file_path": "./data/documents/confluence_456def.json",
    "size_bytes": 8934,
    "error_count": 0
  }
}
```

### Atomic Write Pattern

```python
def atomic_save(self, data: Dict, file_path: Path):
    """Save data with atomic write to prevent corruption"""
    temp_file = file_path.with_suffix('.tmp')
    
    try:
        # Write to temp file
        with open(temp_file, 'w') as f:
            json.dump(data, f, indent=2)
        
        # Atomic rename (platform-specific)
        temp_file.replace(file_path)
        
    except Exception as e:
        # Clean up temp file on error
        if temp_file.exists():
            temp_file.unlink()
        raise
```

---

## Security Architecture

### 1. Credential Management

```python
class SecureCredentialStore:
    def __init__(self):
        self.keyring = keyring.get_keyring()
    
    def store(self, service: str, key: str, value: str):
        self.keyring.set_password(service, key, value)
    
    def retrieve(self, service: str, key: str) -> Optional[str]:
        return self.keyring.get_password(service, key)
```

### 2. Data Sovereignty

All data processing happens locally:
- No external SaaS services
- Documents cached locally
- Embeddings computed locally
- Only LLM API calls leave the environment

### 3. Audit Logging

```python
class AuditLogger:
    def log_query(self, user: str, query: str, documents_accessed: List[str]):
        audit_entry = {
            "timestamp": datetime.now().isoformat(),
            "user": user,
            "query_hash": hashlib.sha256(query.encode()).hexdigest(),
            "documents_accessed": documents_accessed,
            "event_type": "QUERY"
        }
        self._append_to_audit_log(audit_entry)
```

---

## Scalability & Performance

### Performance Optimization Strategies

1. **Embedding Caching**: Store embeddings to avoid recomputation
2. **Document Chunking**: Process large documents in chunks
3. **Parallel Processing**: ThreadPoolExecutor for I/O-bound operations
4. **Lazy Loading**: Load documents only when needed
5. **Token Budget Management**: Truncate context to fit API limits

### Scaling Considerations

| Component | Current | Scale to 1K docs | Scale to 10K docs |
|-----------|---------|------------------|-------------------|
| Storage | JSON files | SQLite | PostgreSQL |
| Embeddings | In-memory | Redis | Vector DB (Pinecone) |
| Sync | Sequential | Parallel | Distributed queue |
| Search | Python | Elasticsearch | Dedicated search service |

---

## Design Decisions & Trade-offs

### 1. JSON vs Database for Document Storage

**Decision**: Start with JSON files

**Rationale**:
- ✅ Simple deployment (no database setup)
- ✅ Easy inspection and debugging
- ✅ Sufficient for < 1000 documents
- ❌ Doesn't scale to 10K+ documents
- ❌ No transactional guarantees

**Migration Path**: SQLite → PostgreSQL when needed

### 2. Local Embeddings vs Cloud Vector DB

**Decision**: Local embeddings with sentence-transformers

**Rationale**:
- ✅ No additional cost
- ✅ Data stays local (compliance)
- ✅ Fast for < 1000 documents
- ❌ Slower for large-scale similarity search
- ❌ No advanced vector DB features

**Migration Path**: Add Pinecone/Weaviate when scale requires

### 3. Single Anthropic Provider vs Multi-Provider

**Decision**: Anthropic-only (Claude models)

**Rationale**:
- ✅ Best-in-class quality for enterprise tasks
- ✅ Extended context windows (200K tokens)
- ✅ Prompt caching for cost optimization
- ❌ Vendor lock-in risk
- ❌ No fallback if Anthropic has outage

**Migration Path**: Abstract behind LLM interface for easy provider swap

### 4. Sync Strategy: Pull vs Push

**Decision**: Pull-based incremental sync

**Rationale**:
- ✅ Simple implementation (no webhooks)
- ✅ Works with all data sources
- ✅ Hash-based change detection is efficient
- ❌ Not real-time (polling delay)
- ❌ Wastes API calls checking for changes

**Future**: Add webhook support for real-time sources

---

## Conclusion

This architecture balances **simplicity with production readiness**:

- **Start simple**: JSON storage, local embeddings, single-agent
- **Scale pragmatically**: Add complexity only when needed
- **Measure everything**: Benchmarks drive architectural decisions
- **Fail gracefully**: Circuit breakers, retries, degraded modes

The system is designed for **incremental evolution**: each component can be replaced independently as requirements grow.

---

*Last Updated: April 2026*
