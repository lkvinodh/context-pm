# Full Implementation Guide

> **⚡ COPY-PASTE READY: This document is designed to be copied directly into Claude or any AI assistant. The AI will automatically generate all code files, configurations, and documentation based on these instructions. Just paste and ask the AI to "implement this system".**

> **A step-by-step guide to building production-grade AI agent systems from scratch**

This guide walks through the complete implementation of both single-agent and multi-agent architectures, with production-ready code examples, error handling patterns, and real-world integration strategies.

**How to use this guide with AI assistants:**
1. Copy the relevant section (or entire document)
2. Paste into Claude Code, ChatGPT, or any AI assistant
3. Say: "Please implement this system following the guide"
4. The AI will generate all necessary files, code, and configurations automatically

All code examples are production-ready and can be directly used in your implementation.

---

## Table of Contents

1. [Prerequisites & Environment Setup](#prerequisites--environment-setup)
2. [Phase 1: Foundation (Single-Agent)](#phase-1-foundation-single-agent)
3. [Phase 2: Data Integration Layer](#phase-2-data-integration-layer)
4. [Phase 3: Incremental Sync Engine](#phase-3-incremental-sync-engine)
5. [Phase 4: Hybrid Retrieval System](#phase-4-hybrid-retrieval-system)
6. [Phase 5: Multi-Agent Architecture](#phase-5-multi-agent-architecture)
7. [Phase 6: Production Hardening](#phase-6-production-hardening)
8. [Testing & Validation](#testing--validation)
9. [Troubleshooting Common Issues](#troubleshooting-common-issues)

---

## Prerequisites & Environment Setup

### Required Tools & Versions

```bash
# Python 3.8+ (tested on 3.9, 3.10, 3.11)
python --version

# Virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Core dependencies
pip install anthropic==0.25.0
pip install sentence-transformers==2.5.1
pip install rank-bm25==0.2.2
pip install requests==2.31.0
pip install python-dotenv==1.0.0
pip install numpy==1.24.0
pip install pandas==2.0.0
```

### Project Structure Setup

```bash
mkdir -p production-ai-agent/{src/{agents,data_integration/{adapters},retrieval,api},config,data/{documents,embeddings,state},benchmark/{results},tests,logs}

cd production-ai-agent

# Create essential files
touch src/__init__.py
touch src/agents/__init__.py
touch src/data_integration/__init__.py
touch src/data_integration/adapters/__init__.py
touch src/retrieval/__init__.py
touch .env
touch requirements.txt
```

### Environment Configuration

Create `.env` file:

```bash
# Anthropic API
ANTHROPIC_API_KEY=your_api_key_here

# GitHub Enterprise
GITHUB_URL=https://github.yourcompany.com
GITHUB_TOKEN=ghp_your_personal_access_token
GITHUB_REPOS=org/repo1,org/repo2,org/repo3

# Confluence
CONFLUENCE_URL=https://yourcompany.atlassian.net
CONFLUENCE_TOKEN=your_bearer_token
CONFLUENCE_SPACES=SPACE1,SPACE2

# Jira
JIRA_URL=https://yourcompany.atlassian.net
JIRA_TOKEN=your_bearer_token
JIRA_PROJECTS=PROJ1,PROJ2

# Office Documents
OFFICE_DOCS_PATH=/path/to/documents

# System Settings
LOG_LEVEL=INFO
MAX_WORKERS=4
CACHE_DIR=./data
STATE_FILE=./data/state/sync_state.json
```

---

## Phase 1: Foundation (Single-Agent)

### Step 1: Configuration Management

Create `config/settings.py`:

```python
import os
from dataclasses import dataclass
from typing import List, Dict
from dotenv import load_dotenv

load_dotenv()

@dataclass
class AnthropicConfig:
    """Anthropic API configuration"""
    api_key: str
    model_sonnet: str = "claude-sonnet-4.6"
    model_opus: str = "claude-opus-4.7"
    model_haiku: str = "claude-haiku-4-5-20251001"
    max_tokens: int = 4096
    temperature: float = 0.7

@dataclass
class GitHubConfig:
    """GitHub Enterprise configuration"""
    url: str
    token: str
    repos: List[str]
    
    @property
    def headers(self) -> Dict[str, str]:
        return {
            "Authorization": f"token {self.token}",
            "Accept": "application/vnd.github.v3+json"
        }

@dataclass
class ConfluenceConfig:
    """Confluence configuration"""
    url: str
    token: str
    spaces: List[str]
    
    @property
    def headers(self) -> Dict[str, str]:
        return {
            "Authorization": f"Bearer {self.token}",
            "Content-Type": "application/json"
        }

@dataclass
class SystemConfig:
    """System-wide configuration"""
    cache_dir: str
    state_file: str
    log_level: str
    max_workers: int
    chunk_size: int = 1000
    chunk_overlap: int = 200

class Config:
    """Master configuration class"""
    
    def __init__(self):
        self.anthropic = AnthropicConfig(
            api_key=os.getenv("ANTHROPIC_API_KEY")
        )
        
        self.github = GitHubConfig(
            url=os.getenv("GITHUB_URL"),
            token=os.getenv("GITHUB_TOKEN"),
            repos=os.getenv("GITHUB_REPOS", "").split(",")
        )
        
        self.confluence = ConfluenceConfig(
            url=os.getenv("CONFLUENCE_URL"),
            token=os.getenv("CONFLUENCE_TOKEN"),
            spaces=os.getenv("CONFLUENCE_SPACES", "").split(",")
        )
        
        self.system = SystemConfig(
            cache_dir=os.getenv("CACHE_DIR", "./data"),
            state_file=os.getenv("STATE_FILE", "./data/state/sync_state.json"),
            log_level=os.getenv("LOG_LEVEL", "INFO"),
            max_workers=int(os.getenv("MAX_WORKERS", "4"))
        )
    
    def validate(self) -> bool:
        """Validate all required configuration"""
        errors = []
        
        if not self.anthropic.api_key:
            errors.append("ANTHROPIC_API_KEY not set")
        
        if not self.github.token:
            errors.append("GITHUB_TOKEN not set")
        
        if errors:
            raise ValueError(f"Configuration errors: {', '.join(errors)}")
        
        return True

# Global config instance
config = Config()
```

### Step 2: Logging Setup

Create `src/utils/logging_config.py`:

```python
import logging
import sys
from pathlib import Path
from datetime import datetime

def setup_logging(log_level: str = "INFO", log_dir: str = "./logs"):
    """Configure application logging"""
    
    # Create logs directory
    Path(log_dir).mkdir(parents=True, exist_ok=True)
    
    # Log file with timestamp
    log_file = Path(log_dir) / f"agent_{datetime.now().strftime('%Y%m%d_%H%M%S')}.log"
    
    # Configure root logger
    logging.basicConfig(
        level=getattr(logging, log_level),
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        handlers=[
            logging.FileHandler(log_file),
            logging.StreamHandler(sys.stdout)
        ]
    )
    
    # Set specific logger levels
    logging.getLogger("anthropic").setLevel(logging.WARNING)
    logging.getLogger("urllib3").setLevel(logging.WARNING)
    
    return logging.getLogger(__name__)

logger = setup_logging()
```

### Step 3: Basic Single-Agent Implementation

Create `src/agents/single_agent.py`:

```python
import anthropic
from typing import Dict, List, Optional
import logging
from config.settings import config

logger = logging.getLogger(__name__)

class SingleAgent:
    """Monolithic agent handling all query types"""
    
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=config.anthropic.api_key
        )
        self.model = config.anthropic.model_sonnet
        self.conversation_history = []
    
    def query(
        self, 
        user_query: str,
        context_documents: Optional[List[Dict]] = None,
        system_prompt: Optional[str] = None
    ) -> Dict:
        """
        Process user query with optional context documents
        
        Args:
            user_query: The user's question
            context_documents: Retrieved documents for context
            system_prompt: Custom system prompt (optional)
        
        Returns:
            Dict with response, token usage, and metadata
        """
        try:
            # Build context from documents
            context_text = self._build_context(context_documents) if context_documents else ""
            
            # Default system prompt
            if not system_prompt:
                system_prompt = self._get_default_system_prompt()
            
            # Build user message
            user_message = self._build_user_message(user_query, context_text)
            
            # API call
            logger.info(f"Querying single agent: {user_query[:100]}...")
            
            response = self.client.messages.create(
                model=self.model,
                max_tokens=config.anthropic.max_tokens,
                temperature=config.anthropic.temperature,
                system=system_prompt,
                messages=[
                    {"role": "user", "content": user_message}
                ]
            )
            
            # Extract response
            result = {
                "response": response.content[0].text,
                "model": self.model,
                "tokens_input": response.usage.input_tokens,
                "tokens_output": response.usage.output_tokens,
                "stop_reason": response.stop_reason
            }
            
            logger.info(f"Single agent response: {response.usage.input_tokens} input, {response.usage.output_tokens} output tokens")
            
            return result
            
        except anthropic.APIError as e:
            logger.error(f"Anthropic API error: {e}")
            raise
        except Exception as e:
            logger.error(f"Single agent query failed: {e}")
            raise
    
    def _get_default_system_prompt(self) -> str:
        """Default system prompt for single agent"""
        return """You are an expert AI assistant specializing in enterprise knowledge management.

Your capabilities:
- Research and information retrieval from technical documentation
- Technical analysis and feasibility assessment
- Content creation (emails, presentations, reports)
- Business analysis and risk assessment

When responding:
1. Base answers on provided context documents
2. Cite sources when possible (document titles, URLs)
3. Be concise but thorough
4. Structure responses clearly with headings and bullet points
5. Acknowledge uncertainty when information is incomplete

Output format:
- Use markdown formatting
- Include relevant quotes from source documents
- Provide actionable insights
- Flag any assumptions or gaps in information"""
    
    def _build_context(self, documents: List[Dict]) -> str:
        """Build context string from documents"""
        if not documents:
            return ""
        
        context_parts = []
        for i, doc in enumerate(documents, 1):
            context_parts.append(f"""
Document {i}: {doc.get('title', 'Untitled')}
Source: {doc.get('source', 'Unknown')}
Relevance: {doc.get('score', 0):.2f}

Content:
{doc.get('content', '')}

---
""")
        
        return "\n".join(context_parts)
    
    def _build_user_message(self, query: str, context: str) -> str:
        """Build complete user message with context"""
        if context:
            return f"""Context Documents:
{context}

User Question:
{query}

Please answer the question based on the provided context documents. Cite specific documents when possible."""
        else:
            return query

# Example usage
if __name__ == "__main__":
    agent = SingleAgent()
    
    # Simple query without context
    result = agent.query("What is the purpose of this system?")
    print(result["response"])
```

---

## Phase 2: Data Integration Layer

### Step 1: Base Adapter Interface

Create `src/data_integration/adapters/base_adapter.py`:

```python
from abc import ABC, abstractmethod
from typing import List, Dict, Optional
import logging
from dataclasses import dataclass
from datetime import datetime

logger = logging.getLogger(__name__)

@dataclass
class Document:
    """Standard document representation"""
    id: str
    title: str
    content: str
    source: str
    url: Optional[str]
    metadata: Dict
    created_at: datetime
    updated_at: datetime
    content_hash: str

class BaseAdapter(ABC):
    """Base class for all data source adapters"""
    
    def __init__(self, config):
        self.config = config
        self.logger = logging.getLogger(self.__class__.__name__)
    
    @abstractmethod
    def authenticate(self) -> bool:
        """Authenticate with the data source"""
        pass
    
    @abstractmethod
    def fetch_documents(self, **kwargs) -> List[Document]:
        """Fetch documents from the data source"""
        pass
    
    @abstractmethod
    def get_document_by_id(self, doc_id: str) -> Optional[Document]:
        """Retrieve a specific document by ID"""
        pass
    
    def health_check(self) -> bool:
        """Verify connection to data source"""
        try:
            return self.authenticate()
        except Exception as e:
            self.logger.error(f"Health check failed: {e}")
            return False
```

### Step 2: GitHub Enterprise Adapter

Create `src/data_integration/adapters/github_adapter.py`:

```python
import requests
import hashlib
import base64
from typing import List, Dict, Optional
from datetime import datetime
from .base_adapter import BaseAdapter, Document
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

class GitHubAdapter(BaseAdapter):
    """GitHub Enterprise API adapter with production error handling"""
    
    def __init__(self, config):
        super().__init__(config)
        self.session = self._create_session()
        self.base_url = f"{config.github.url}/api/v3"
    
    def _create_session(self) -> requests.Session:
        """Create session with retry logic"""
        session = requests.Session()
        
        # Retry strategy
        retry_strategy = Retry(
            total=3,
            backoff_factor=1,
            status_forcelist=[429, 500, 502, 503, 504],
            allowed_methods=["HEAD", "GET", "OPTIONS"]
        )
        
        adapter = HTTPAdapter(max_retries=retry_strategy)
        session.mount("https://", adapter)
        session.mount("http://", adapter)
        
        session.headers.update(self.config.github.headers)
        
        return session
    
    def authenticate(self) -> bool:
        """Verify GitHub authentication"""
        try:
            response = self.session.get(f"{self.base_url}/user")
            response.raise_for_status()
            self.logger.info(f"Authenticated as: {response.json()['login']}")
            return True
        except requests.exceptions.RequestException as e:
            self.logger.error(f"GitHub authentication failed: {e}")
            return False
    
    def fetch_documents(self, **kwargs) -> List[Document]:
        """Fetch markdown documents from configured repositories"""
        all_documents = []
        
        for repo in self.config.github.repos:
            try:
                self.logger.info(f"Fetching documents from {repo}")
                docs = self._fetch_repo_documents(repo)
                all_documents.extend(docs)
                self.logger.info(f"Fetched {len(docs)} documents from {repo}")
            except Exception as e:
                self.logger.error(f"Failed to fetch from {repo}: {e}")
                continue
        
        return all_documents
    
    def _fetch_repo_documents(self, repo: str) -> List[Document]:
        """Fetch all markdown files from a repository"""
        documents = []
        
        # Search for markdown files
        query = f"repo:{repo} extension:md"
        search_url = f"{self.base_url}/search/code"
        
        params = {
            "q": query,
            "per_page": 100
        }
        
        try:
            response = self.session.get(search_url, params=params)
            response.raise_for_status()
            results = response.json()
            
            for item in results.get("items", []):
                doc = self._fetch_file_content(repo, item["path"])
                if doc:
                    documents.append(doc)
            
        except requests.exceptions.RequestException as e:
            self.logger.error(f"Search failed for {repo}: {e}")
        
        return documents
    
    def _fetch_file_content(self, repo: str, path: str) -> Optional[Document]:
        """Fetch content of a specific file"""
        url = f"{self.base_url}/repos/{repo}/contents/{path}"
        
        try:
            response = self.session.get(url)
            response.raise_for_status()
            data = response.json()
            
            # Decode content
            content = base64.b64decode(data["content"]).decode("utf-8")
            
            # Create document
            doc = Document(
                id=data["sha"],
                title=data["name"],
                content=content,
                source="github",
                url=data["html_url"],
                metadata={
                    "repo": repo,
                    "path": path,
                    "size": data["size"]
                },
                created_at=datetime.now(),  # GitHub doesn't provide creation date via this API
                updated_at=datetime.now(),
                content_hash=self._compute_hash(content)
            )
            
            return doc
            
        except Exception as e:
            self.logger.error(f"Failed to fetch {path} from {repo}: {e}")
            return None
    
    def get_document_by_id(self, doc_id: str) -> Optional[Document]:
        """Retrieve document by SHA"""
        # Implementation depends on how you store ID-to-repo mapping
        # This is a simplified version
        for repo in self.config.github.repos:
            # Search logic here
            pass
        return None
    
    @staticmethod
    def _compute_hash(content: str) -> str:
        """Compute SHA-256 hash of content"""
        return hashlib.sha256(content.encode()).hexdigest()

# Example usage
if __name__ == "__main__":
    from config.settings import config
    
    adapter = GitHubAdapter(config)
    
    if adapter.authenticate():
        docs = adapter.fetch_documents()
        print(f"Fetched {len(docs)} documents")
        
        if docs:
            print(f"\nFirst document:")
            print(f"Title: {docs[0].title}")
            print(f"Source: {docs[0].source}")
            print(f"Content preview: {docs[0].content[:200]}...")
```

### Step 3: Confluence Adapter

Create `src/data_integration/adapters/confluence_adapter.py`:

```python
import requests
import hashlib
from typing import List, Optional
from datetime import datetime
from .base_adapter import BaseAdapter, Document
from html import unescape
import re

class ConfluenceAdapter(BaseAdapter):
    """Confluence Cloud/Server adapter"""
    
    def __init__(self, config):
        super().__init__(config)
        self.session = requests.Session()
        self.session.headers.update(config.confluence.headers)
        self.base_url = f"{config.confluence.url}/wiki/rest/api"
    
    def authenticate(self) -> bool:
        """Verify Confluence authentication"""
        try:
            response = self.session.get(f"{self.base_url}/user/current")
            response.raise_for_status()
            user = response.json()
            self.logger.info(f"Authenticated as: {user.get('displayName', 'Unknown')}")
            return True
        except requests.exceptions.RequestException as e:
            self.logger.error(f"Confluence authentication failed: {e}")
            return False
    
    def fetch_documents(self, **kwargs) -> List[Document]:
        """Fetch pages from configured Confluence spaces"""
        all_documents = []
        
        for space_key in self.config.confluence.spaces:
            try:
                self.logger.info(f"Fetching pages from space: {space_key}")
                docs = self._fetch_space_pages(space_key)
                all_documents.extend(docs)
                self.logger.info(f"Fetched {len(docs)} pages from {space_key}")
            except Exception as e:
                self.logger.error(f"Failed to fetch from space {space_key}: {e}")
                continue
        
        return all_documents
    
    def _fetch_space_pages(self, space_key: str) -> List[Document]:
        """Fetch all pages from a Confluence space"""
        documents = []
        start = 0
        limit = 50
        
        while True:
            url = f"{self.base_url}/space/{space_key}/content/page"
            params = {
                "start": start,
                "limit": limit,
                "expand": "body.storage,version,metadata.labels"
            }
            
            try:
                response = self.session.get(url, params=params)
                response.raise_for_status()
                data = response.json()
                
                for page in data.get("results", []):
                    doc = self._convert_page_to_document(page, space_key)
                    if doc:
                        documents.append(doc)
                
                # Check if more pages exist
                if len(data.get("results", [])) < limit:
                    break
                
                start += limit
                
            except requests.exceptions.RequestException as e:
                self.logger.error(f"Failed to fetch pages from {space_key}: {e}")
                break
        
        return documents
    
    def _convert_page_to_document(self, page: dict, space_key: str) -> Optional[Document]:
        """Convert Confluence page to Document object"""
        try:
            # Extract content
            body = page.get("body", {}).get("storage", {}).get("value", "")
            content = self._html_to_text(body)
            
            # Extract metadata
            version = page.get("version", {})
            labels = [label["name"] for label in page.get("metadata", {}).get("labels", {}).get("results", [])]
            
            doc = Document(
                id=page["id"],
                title=page["title"],
                content=content,
                source="confluence",
                url=f"{self.config.confluence.url}/wiki{page['_links']['webui']}",
                metadata={
                    "space": space_key,
                    "version": version.get("number", 1),
                    "labels": labels,
                    "created_by": version.get("by", {}).get("displayName", "Unknown")
                },
                created_at=datetime.fromisoformat(version.get("when", "").replace("Z", "+00:00")) if version.get("when") else datetime.now(),
                updated_at=datetime.now(),
                content_hash=hashlib.sha256(content.encode()).hexdigest()
            )
            
            return doc
            
        except Exception as e:
            self.logger.error(f"Failed to convert page {page.get('id')}: {e}")
            return None
    
    @staticmethod
    def _html_to_text(html: str) -> str:
        """Convert HTML to plain text"""
        # Remove HTML tags
        text = re.sub(r'<[^>]+>', '', html)
        # Unescape HTML entities
        text = unescape(text)
        # Clean up whitespace
        text = re.sub(r'\s+', ' ', text).strip()
        return text
    
    def get_document_by_id(self, doc_id: str) -> Optional[Document]:
        """Retrieve specific Confluence page by ID"""
        url = f"{self.base_url}/content/{doc_id}"
        params = {"expand": "body.storage,version,metadata.labels"}
        
        try:
            response = self.session.get(url, params=params)
            response.raise_for_status()
            page = response.json()
            
            space_key = page.get("space", {}).get("key", "")
            return self._convert_page_to_document(page, space_key)
            
        except requests.exceptions.RequestException as e:
            self.logger.error(f"Failed to fetch page {doc_id}: {e}")
            return None
```

---

## Phase 3: Incremental Sync Engine

Create `src/data_integration/sync_engine.py`:

```python
import json
import hashlib
from pathlib import Path
from typing import List, Dict, Optional
from datetime import datetime
import logging
from concurrent.futures import ThreadPoolExecutor, as_completed
from .adapters.base_adapter import Document, BaseAdapter

logger = logging.getLogger(__name__)

class SyncEngine:
    """Incremental sync with hash-based change detection"""
    
    def __init__(self, state_file: str, cache_dir: str, max_workers: int = 4):
        self.state_file = Path(state_file)
        self.cache_dir = Path(cache_dir)
        self.max_workers = max_workers
        
        # Ensure directories exist
        self.state_file.parent.mkdir(parents=True, exist_ok=True)
        self.cache_dir.mkdir(parents=True, exist_ok=True)
        
        self.state = self._load_state()
    
    def _load_state(self) -> Dict:
        """Load sync state from disk"""
        if self.state_file.exists():
            try:
                with open(self.state_file, 'r') as f:
                    return json.load(f)
            except Exception as e:
                logger.error(f"Failed to load state: {e}")
                return {}
        return {}
    
    def _save_state(self):
        """Save sync state with atomic write"""
        temp_file = self.state_file.with_suffix('.tmp')
        
        try:
            # Write to temp file
            with open(temp_file, 'w') as f:
                json.dump(self.state, f, indent=2, default=str)
            
            # Atomic rename
            temp_file.replace(self.state_file)
            logger.info(f"State saved: {len(self.state)} documents tracked")
            
        except Exception as e:
            logger.error(f"Failed to save state: {e}")
            if temp_file.exists():
                temp_file.unlink()
    
    def sync(self, adapters: List[BaseAdapter], force: bool = False) -> Dict:
        """
        Sync documents from all adapters
        
        Args:
            adapters: List of data source adapters
            force: Force re-sync all documents
        
        Returns:
            Dict with sync statistics
        """
        stats = {
            "total_fetched": 0,
            "new": 0,
            "updated": 0,
            "unchanged": 0,
            "errors": 0,
            "start_time": datetime.now(),
            "adapters": {}
        }
        
        for adapter in adapters:
            adapter_name = adapter.__class__.__name__
            logger.info(f"Syncing from {adapter_name}...")
            
            try:
                # Fetch documents
                documents = adapter.fetch_documents()
                stats["total_fetched"] += len(documents)
                
                # Process documents
                adapter_stats = self._process_documents(documents, force)
                stats.update({
                    "new": stats["new"] + adapter_stats["new"],
                    "updated": stats["updated"] + adapter_stats["updated"],
                    "unchanged": stats["unchanged"] + adapter_stats["unchanged"]
                })
                stats["adapters"][adapter_name] = adapter_stats
                
            except Exception as e:
                logger.error(f"Sync failed for {adapter_name}: {e}")
                stats["errors"] += 1
                stats["adapters"][adapter_name] = {"error": str(e)}
        
        # Save state
        self._save_state()
        
        stats["end_time"] = datetime.now()
        stats["duration"] = (stats["end_time"] - stats["start_time"]).total_seconds()
        
        self._log_summary(stats)
        
        return stats
    
    def _process_documents(self, documents: List[Document], force: bool) -> Dict:
        """Process documents with change detection"""
        stats = {"new": 0, "updated": 0, "unchanged": 0}
        
        with ThreadPoolExecutor(max_workers=self.max_workers) as executor:
            futures = {
                executor.submit(self._process_single_document, doc, force): doc
                for doc in documents
            }
            
            for future in as_completed(futures):
                try:
                    status = future.result()
                    stats[status] += 1
                except Exception as e:
                    logger.error(f"Document processing failed: {e}")
        
        return stats
    
    def _process_single_document(self, doc: Document, force: bool) -> str:
        """Process a single document with change detection"""
        doc_key = f"{doc.source}:{doc.id}"
        
        # Check if document changed
        if not force and doc_key in self.state:
            if self.state[doc_key].get("content_hash") == doc.content_hash:
                return "unchanged"
        
        # Save document
        doc_path = self.cache_dir / f"{doc.source}_{doc.id}.json"
        
        try:
            with open(doc_path, 'w') as f:
                json.dump({
                    "id": doc.id,
                    "title": doc.title,
                    "content": doc.content,
                    "source": doc.source,
                    "url": doc.url,
                    "metadata": doc.metadata,
                    "created_at": doc.created_at.isoformat(),
                    "updated_at": doc.updated_at.isoformat(),
                    "content_hash": doc.content_hash
                }, f, indent=2)
            
            # Update state
            status = "new" if doc_key not in self.state else "updated"
            self.state[doc_key] = {
                "content_hash": doc.content_hash,
                "last_synced": datetime.now().isoformat(),
                "file_path": str(doc_path)
            }
            
            return status
            
        except Exception as e:
            logger.error(f"Failed to save document {doc_key}: {e}")
            raise
    
    def get_all_documents(self) -> List[Document]:
        """Load all cached documents"""
        documents = []
        
        for doc_key, doc_state in self.state.items():
            try:
                file_path = Path(doc_state["file_path"])
                if file_path.exists():
                    with open(file_path, 'r') as f:
                        data = json.load(f)
                        doc = Document(
                            id=data["id"],
                            title=data["title"],
                            content=data["content"],
                            source=data["source"],
                            url=data.get("url"),
                            metadata=data.get("metadata", {}),
                            created_at=datetime.fromisoformat(data["created_at"]),
                            updated_at=datetime.fromisoformat(data["updated_at"]),
                            content_hash=data["content_hash"]
                        )
                        documents.append(doc)
            except Exception as e:
                logger.error(f"Failed to load document {doc_key}: {e}")
        
        return documents
    
    def _log_summary(self, stats: Dict):
        """Log sync summary"""
        logger.info("=" * 60)
        logger.info("SYNC SUMMARY")
        logger.info("=" * 60)
        logger.info(f"Duration: {stats['duration']:.2f}s")
        logger.info(f"Total fetched: {stats['total_fetched']}")
        logger.info(f"New documents: {stats['new']}")
        logger.info(f"Updated documents: {stats['updated']}")
        logger.info(f"Unchanged documents: {stats['unchanged']}")
        logger.info(f"Errors: {stats['errors']}")
        logger.info("=" * 60)
```

---

**Continue reading:**
- Next sections cover Phase 4-6 (Hybrid Retrieval, Multi-Agent, Production Hardening)
- Complete code examples in repository
- See [ARCHITECTURE.md](ARCHITECTURE.md) for system design details
- See [BENCHMARKS.md](BENCHMARKS.md) for performance testing methodology

---

*This implementation guide is actively maintained. Last updated: April 2026*
