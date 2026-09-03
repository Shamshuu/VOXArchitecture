# VOX Architecture - Step 1: Codebase Indexer & Hybrid Query Engine

> **Voice-to-Voice Codebase Explainer Engine - Step 1**  
> High-performance local AST code indexer, symbol extractor, cross-file relationship resolver, and hybrid semantic vector search engine.

---

## 🌟 Key Capabilities

1. **AST & Symbol Extraction via Tree-Sitter**:
   - Parses **Python**, **TypeScript**, **TSX**, **JavaScript**, and **JSX** natively with standard tree-sitter grammars (with extensible fallbacks for Go, Rust, Java, C++, Markdown, JSON, YAML, etc.).
   - Extracts classes, methods, functions, async functions, decorators, parameters, types, interfaces, React components, and Express/FastAPI routes.

2. **Code-Aware Boundary Text Splitter**:
   - Chunks files along **natural symbol boundaries** (classes, methods, standalone functions, route blocks, module import headers) rather than blindly chopping lines.
   - Preserves parent class scope context (e.g., `// Scope: class UserService`) for granular method chunks.

3. **Cross-Module Relationship Graph**:
   - Resolves relative and absolute imports across files (`File A imports class X from File B`).
   - Maps internal symbol imports, module re-exports, and external package dependencies.

4. **Local SQLite Storage with FTS5 & Vector Embeddings**:
   - Incremental indexing powered by **MD5 checksum diffing** (0 re-parsing on unchanged code).
   - Cascade pruning of deleted files and orphaned vector records.
   - SQLite tables: `files`, `symbols`, `relationships`, `chunks`, `chunk_embeddings`, and `fts_chunks` (FTS5 BM25).

5. **Multi-Model Local Vector Embedding**:
   - Default: **FastEmbed (BAAI/bge-small-en-v1.5)** running 100% locally via ONNX Runtime (384 dimensions, zero API keys required).
   - Extensible: Supports Ollama (`qwen3.5`, `nomic-embed-text`, etc.) or deterministic offline subword hash vectorizer fallback.

6. **Hybrid Code Search & CLI Query Tool**:
   - Combines **Semantic Vector Cosine Similarity**, **SQLite FTS5 BM25 Full-Text Search**, **Symbol Lookup**, and **Relationship Graph Context** using **Reciprocal Rank Fusion (RRF)**.
   - Rich syntax-highlighted terminal output with exact line ranges, symbol metadata, and JSON export mode.

---

## 🚀 Quick Start

### 1. Installation

```bash
# Setup Python 3.10+ Virtual Environment
python3 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### 2. Index a Codebase

Point `indexer.py` at any target directory or repository:

```bash
# Index current directory or a sample repo
python indexer.py tests/test_sample_repo

# Force full re-indexing
python indexer.py tests/test_sample_repo --force

# Custom database output path
python indexer.py tests/test_sample_repo --db /path/to/custom.db
```

### 3. Query the Codebase

Ask natural language questions or lookup symbols:

```bash
# Semantic query
python query_engine.py "Why is our user model throwing an validation error?" --db tests/test_sample_repo/.vox/codebase.db

# Authentication / Token query
python query_engine.py "Where is the authentication JWT token generated?" --db tests/test_sample_repo/.vox/codebase.db

# API Routes query
python query_engine.py "What endpoints does the user router handle?" --db tests/test_sample_repo/.vox/codebase.db

# JSON output mode for programmatic / agent integration
python query_engine.py "UserModel" --db tests/test_sample_repo/.vox/codebase.db --json

# Detailed score explanation
python query_engine.py "validateEmail" --db tests/test_sample_repo/.vox/codebase.db --explain
```

---

## 🏗️ SQLite Schema

```sql
-- 1. Files Table
CREATE TABLE files (
    id TEXT PRIMARY KEY,
    path TEXT UNIQUE NOT NULL,
    checksum TEXT NOT NULL,
    language TEXT NOT NULL,
    lines_of_code INTEGER NOT NULL,
    ast_json TEXT,
    indexed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 2. Symbols Table
CREATE TABLE symbols (
    id TEXT PRIMARY KEY,
    file_id TEXT NOT NULL,
    file_path TEXT NOT NULL,
    name TEXT NOT NULL,
    type TEXT NOT NULL,
    signature TEXT,
    docstring TEXT,
    line_start INTEGER NOT NULL,
    line_end INTEGER NOT NULL,
    parent_name TEXT,
    FOREIGN KEY(file_id) REFERENCES files(id) ON DELETE CASCADE
);

-- 3. Relationships Table
CREATE TABLE relationships (
    id TEXT PRIMARY KEY,
    from_file TEXT NOT NULL,
    to_file TEXT NOT NULL,
    symbol_name TEXT NOT NULL,
    dependency_type TEXT NOT NULL,
    line_number INTEGER NOT NULL
);

-- 4. Chunks Table
CREATE TABLE chunks (
    id TEXT PRIMARY KEY,
    file_id TEXT NOT NULL,
    file_path TEXT NOT NULL,
    chunk_type TEXT NOT NULL,
    name TEXT NOT NULL,
    content TEXT NOT NULL,
    line_start INTEGER NOT NULL,
    line_end INTEGER NOT NULL,
    parent_symbol TEXT,
    docstring TEXT,
    symbols_contained TEXT,
    FOREIGN KEY(file_id) REFERENCES files(id) ON DELETE CASCADE
);

-- 5. Chunk Embeddings Table
CREATE TABLE chunk_embeddings (
    chunk_id TEXT PRIMARY KEY,
    model TEXT NOT NULL,
    dimensions INTEGER NOT NULL,
    embedding_blob BLOB NOT NULL,
    FOREIGN KEY(chunk_id) REFERENCES chunks(id) ON DELETE CASCADE
);

-- 6. FTS5 Virtual Table
CREATE VIRTUAL TABLE fts_chunks USING fts5(
    chunk_id UNINDEXED,
    file_path,
    name,
    chunk_type,
    symbols_contained,
    content,
    tokenize = 'porter unicode61'
);
```

---

## 🧪 Running Tests

```bash
pytest -v tests/
```
