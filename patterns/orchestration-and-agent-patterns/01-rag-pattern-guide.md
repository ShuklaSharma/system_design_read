# RAG (Retrieval-Augmented Generation) Pattern

> AI Engineering Series | March 2026

---

## Table of Contents

1. [Introduction](#introduction)
2. [How RAG Works](#how-rag-works)
3. [Core Components](#core-components)
4. [Building a RAG Pipeline — Step by Step](#building-a-rag-pipeline)
5. [Chunking Strategies](#chunking-strategies)
6. [Embedding Models](#embedding-models)
7. [Vector Databases](#vector-databases)
8. [Retrieval Strategies](#retrieval-strategies)
9. [Prompt Construction](#prompt-construction)
10. [Advanced RAG Patterns](#advanced-rag-patterns)
11. [Evaluation](#evaluation)
12. [Production Best Practices](#production-best-practices)
13. [Common Mistakes](#common-mistakes)
14. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## Introduction

### What is RAG?

**Retrieval-Augmented Generation (RAG)** is an architecture pattern that connects an LLM to an external knowledge base. Before generating an answer, the system *retrieves* relevant documents from that knowledge base and injects them into the LLM prompt as context — grounding the response in real, up-to-date information rather than relying solely on the model's training data.

**Origin:** Introduced by Lewis et al. (Facebook AI, 2020) in the paper *"Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks"*.

### The Problem RAG Solves

```
LLMs have two fundamental limitations:

1. Knowledge Cutoff
   ─────────────────────────────────────────
   Training data has an end date.
   User: "What are the new features in Python 3.13?"
   LLM:  "I don't have information beyond my training cutoff."
   ← Useless for anything recent

2. No Private Knowledge
   ─────────────────────────────────────────
   LLMs know public internet data, not YOUR data.
   User: "Summarise our Q4 2025 board meeting notes."
   LLM:  "I don't have access to your internal documents."
   ← Useless for enterprise use cases

3. Hallucination
   ─────────────────────────────────────────
   When the model doesn't know, it may invent plausible-sounding answers.
   User: "What is our refund policy?"
   LLM:  "Your refund policy allows 30 days..." ← INVENTED
   ← Dangerous for customer-facing applications
```

### The Solution With RAG

```
RAG Pipeline:

User Question
     ↓
[Embed question into vector]
     ↓
[Search vector DB for similar documents]
     ↓
[Retrieve top-k most relevant chunks]
     ↓
[Inject chunks into LLM prompt as context]
     ↓
[LLM generates answer grounded in retrieved context]
     ↓
Answer (cited, accurate, up-to-date)
```

### When to Use RAG vs Alternatives

| Approach | Use When | Avoid When |
|---|---|---|
| **RAG** | Large/dynamic knowledge base, need citations | Knowledge is tiny and static |
| **Fine-tuning** | Consistent style/format, stable knowledge | Frequently changing data |
| **Long context** | Small docs that fit in context window | Millions of documents |
| **Pure LLM** | General knowledge, reasoning tasks | Any domain-specific/private data |

---

## How RAG Works

### The Full Pipeline

```
INDEXING (one-time / periodic)          QUERYING (per request)
══════════════════════════════          ══════════════════════

Raw Documents                           User Question
     │                                       │
     ▼                                       ▼
[1. Load & Clean]                      [4. Embed Question]
     │                                       │
     ▼                                       ▼
[2. Chunk into segments]               [5. Vector Search]
     │                                       │
     ▼                                       ▼
[3. Embed each chunk]                  [6. Retrieve top-k chunks]
     │                                       │
     ▼                                       ▼
[Store in Vector DB] ──────────────→   [7. Build prompt with context]
                                            │
                                            ▼
                                       [8. LLM generates answer]
                                            │
                                            ▼
                                       Grounded Answer + Sources
```

### Step-by-Step Walkthrough

```
Example: Internal HR chatbot

Documents: Employee Handbook (200 pages), Benefits Guide, PTO Policy

─── INDEXING ───────────────────────────────────────────────────────

Step 1 — Load
  employee_handbook.pdf → raw text

Step 2 — Chunk
  "Section 4.2: Annual Leave
   Employees are entitled to 20 days of annual leave per calendar year.
   Leave must be approved by your line manager at least 2 weeks in advance..."
  
  Chunk 1: "Employees are entitled to 20 days of annual leave..."  (300 tokens)
  Chunk 2: "Leave must be approved by your line manager..."        (280 tokens)
  Chunk 3: "Unused leave can be carried over up to 5 days..."     (310 tokens)

Step 3 — Embed
  Chunk 1 → [0.23, -0.41, 0.87, ..., 0.12]  (1536-dim vector)
  Chunk 2 → [0.19, -0.38, 0.91, ..., 0.08]
  Chunk 3 → [0.31, -0.22, 0.79, ..., 0.17]

Step 4 — Store in Pinecone/Weaviate/pgvector
  {id: "handbook_4_2_chunk1", vector: [...], metadata: {text: "...", page: 42}}

─── QUERYING ───────────────────────────────────────────────────────

User: "How many vacation days do I get?"

Step 5 — Embed query
  "How many vacation days do I get?" → [0.21, -0.39, 0.88, ..., 0.11]

Step 6 — Vector search (cosine similarity)
  Chunk 1 score: 0.94  ← very similar
  Chunk 2 score: 0.87
  Chunk 3 score: 0.81
  ... retrieve top 3

Step 7 — Build prompt
  "Answer using only this context:
   [Chunk 1 text]
   [Chunk 2 text]
   [Chunk 3 text]
   
   Question: How many vacation days do I get?"

Step 8 — LLM answers
  "According to the employee handbook, you are entitled to 20 days of annual
   leave per calendar year (Section 4.2). Leave must be approved by your line
   manager at least 2 weeks in advance."
```

---

## Core Components

### Component Overview

```
RAG System
├── Document Loader      — reads PDFs, Word docs, web pages, databases
├── Text Splitter        — chunks documents into LLM-friendly segments
├── Embedding Model      — converts text to dense vectors
├── Vector Database      — stores and searches vectors by similarity
├── Retriever            — queries the vector DB for relevant chunks
├── Prompt Builder       — assembles context + question into LLM prompt
├── LLM                  — generates the final answer
└── Output Parser        — extracts structured data from LLM response
```

---

## Building a RAG Pipeline

### Complete End-to-End Implementation

```python
# ── Dependencies ──────────────────────────────────────────────────
# pip install anthropic pinecone-client sentence-transformers pypdf

import os
import re
from pathlib import Path
from typing import List, Dict, Optional
import anthropic
from pinecone import Pinecone, ServerlessSpec
from sentence_transformers import SentenceTransformer

# ── Configuration ─────────────────────────────────────────────────
EMBEDDING_MODEL  = "all-MiniLM-L6-v2"   # 384-dim, fast, free
PINECONE_INDEX   = "rag-tutorial"
CHUNK_SIZE       = 400    # tokens (approx characters / 4)
CHUNK_OVERLAP    = 80     # overlap between consecutive chunks
TOP_K            = 5      # chunks to retrieve per query


# ═══════════════════════════════════════════════════════════════════
# STEP 1 — DOCUMENT LOADING
# ═══════════════════════════════════════════════════════════════════

class DocumentLoader:
    """Load text from various sources."""

    def load_text_file(self, path: str) -> str:
        with open(path, "r", encoding="utf-8") as f:
            return f.read()

    def load_pdf(self, path: str) -> str:
        from pypdf import PdfReader
        reader = PdfReader(path)
        pages = [page.extract_text() or "" for page in reader.pages]
        return "\n\n".join(pages)

    def load_directory(self, directory: str, extensions: List[str] = [".txt", ".pdf"]) -> List[Dict]:
        """Load all documents from a directory."""
        docs = []
        for path in Path(directory).rglob("*"):
            if path.suffix in extensions:
                try:
                    if path.suffix == ".pdf":
                        text = self.load_pdf(str(path))
                    else:
                        text = self.load_text_file(str(path))
                    docs.append({"source": str(path), "text": text})
                    print(f"  Loaded: {path.name} ({len(text)} chars)")
                except Exception as e:
                    print(f"  Failed to load {path.name}: {e}")
        return docs


# ═══════════════════════════════════════════════════════════════════
# STEP 2 — TEXT SPLITTING (CHUNKING)
# ═══════════════════════════════════════════════════════════════════

class TextSplitter:
    """Split documents into overlapping chunks."""

    def __init__(self, chunk_size: int = CHUNK_SIZE, overlap: int = CHUNK_OVERLAP):
        self.chunk_size = chunk_size   # in approximate tokens (chars / 4)
        self.overlap    = overlap

    def split(self, text: str, source: str = "") -> List[Dict]:
        """Split text into overlapping chunks with metadata."""

        # Clean text
        text = re.sub(r'\n{3,}', '\n\n', text)   # collapse excess newlines
        text = re.sub(r' {2,}', ' ', text)         # collapse excess spaces

        chunks = []
        char_size    = self.chunk_size * 4    # approximate chars per chunk
        char_overlap = self.overlap * 4

        start = 0
        chunk_idx = 0

        while start < len(text):
            end = start + char_size

            # Try to break at a sentence boundary
            if end < len(text):
                boundary = text.rfind('. ', start, end)
                if boundary != -1 and boundary > start + char_size // 2:
                    end = boundary + 1   # include the period

            chunk_text = text[start:end].strip()

            if chunk_text:
                chunks.append({
                    "id":       f"{source}_chunk_{chunk_idx}",
                    "text":     chunk_text,
                    "source":   source,
                    "chunk_idx": chunk_idx,
                    "char_start": start,
                    "char_end":  end,
                })
                chunk_idx += 1

            start = end - char_overlap   # overlap with next chunk
            if start >= len(text):
                break

        return chunks

    def split_by_paragraph(self, text: str, source: str = "") -> List[Dict]:
        """Alternative: split on paragraph boundaries (good for structured docs)."""
        paragraphs = [p.strip() for p in text.split('\n\n') if p.strip()]
        chunks = []
        current_chunk = ""
        chunk_idx = 0

        for para in paragraphs:
            if len(current_chunk) + len(para) > self.chunk_size * 4:
                if current_chunk:
                    chunks.append({
                        "id":        f"{source}_para_chunk_{chunk_idx}",
                        "text":      current_chunk.strip(),
                        "source":    source,
                        "chunk_idx": chunk_idx,
                    })
                    chunk_idx += 1
                current_chunk = para
            else:
                current_chunk += "\n\n" + para if current_chunk else para

        if current_chunk:
            chunks.append({
                "id":        f"{source}_para_chunk_{chunk_idx}",
                "text":      current_chunk.strip(),
                "source":    source,
                "chunk_idx": chunk_idx,
            })

        return chunks


# ═══════════════════════════════════════════════════════════════════
# STEP 3 — EMBEDDING
# ═══════════════════════════════════════════════════════════════════

class EmbeddingEngine:
    """Convert text chunks into dense vectors."""

    def __init__(self, model_name: str = EMBEDDING_MODEL):
        print(f"Loading embedding model: {model_name}")
        self.model = SentenceTransformer(model_name)
        self.dimension = self.model.get_sentence_embedding_dimension()
        print(f"Embedding dimension: {self.dimension}")

    def embed(self, text: str) -> List[float]:
        """Embed a single text string."""
        vector = self.model.encode(text, normalize_embeddings=True)
        return vector.tolist()

    def embed_batch(self, texts: List[str], batch_size: int = 64) -> List[List[float]]:
        """Embed a list of texts in batches for efficiency."""
        all_vectors = []
        for i in range(0, len(texts), batch_size):
            batch = texts[i:i + batch_size]
            vectors = self.model.encode(batch, normalize_embeddings=True)
            all_vectors.extend(vectors.tolist())
            print(f"  Embedded {min(i + batch_size, len(texts))}/{len(texts)} chunks")
        return all_vectors


# ═══════════════════════════════════════════════════════════════════
# STEP 4 — VECTOR DATABASE (PINECONE)
# ═══════════════════════════════════════════════════════════════════

class VectorStore:
    """Store and retrieve embeddings from Pinecone."""

    def __init__(self, api_key: str, index_name: str, dimension: int):
        self.pc         = Pinecone(api_key=api_key)
        self.index_name = index_name
        self.dimension  = dimension
        self.index      = self._get_or_create_index()

    def _get_or_create_index(self):
        existing = [i.name for i in self.pc.list_indexes()]

        if self.index_name not in existing:
            print(f"Creating Pinecone index: {self.index_name}")
            self.pc.create_index(
                name=self.index_name,
                dimension=self.dimension,
                metric="cosine",
                spec=ServerlessSpec(cloud="aws", region="us-east-1"),
            )

        return self.pc.Index(self.index_name)

    def upsert_chunks(self, chunks: List[Dict], vectors: List[List[float]]) -> int:
        """Store chunks and their vectors in Pinecone."""
        records = []
        for chunk, vector in zip(chunks, vectors):
            records.append({
                "id":     chunk["id"],
                "values": vector,
                "metadata": {
                    "text":      chunk["text"][:1000],  # Pinecone metadata limit
                    "source":    chunk.get("source", ""),
                    "chunk_idx": chunk.get("chunk_idx", 0),
                }
            })

        # Upsert in batches of 100 (Pinecone limit)
        batch_size = 100
        total_upserted = 0
        for i in range(0, len(records), batch_size):
            batch = records[i:i + batch_size]
            self.index.upsert(vectors=batch)
            total_upserted += len(batch)
            print(f"  Upserted {total_upserted}/{len(records)} records")

        return total_upserted

    def search(self, query_vector: List[float], top_k: int = TOP_K) -> List[Dict]:
        """Find the top-k most similar chunks to the query vector."""
        results = self.index.query(
            vector=query_vector,
            top_k=top_k,
            include_metadata=True,
        )

        return [
            {
                "id":     match["id"],
                "score":  round(match["score"], 4),
                "text":   match["metadata"]["text"],
                "source": match["metadata"].get("source", ""),
            }
            for match in results["matches"]
        ]

    def get_stats(self) -> Dict:
        return self.index.describe_index_stats()


# ═══════════════════════════════════════════════════════════════════
# STEP 5 — PROMPT BUILDER
# ═══════════════════════════════════════════════════════════════════

def build_rag_prompt(question: str, retrieved_chunks: List[Dict]) -> str:
    """Assemble the retrieved context and question into an LLM prompt."""

    if not retrieved_chunks:
        context_section = "No relevant documents were found."
    else:
        context_parts = []
        for i, chunk in enumerate(retrieved_chunks, 1):
            source = chunk.get("source", "Unknown")
            context_parts.append(
                f"[Source {i}: {source}]\n{chunk['text']}"
            )
        context_section = "\n\n---\n\n".join(context_parts)

    return f"""You are a helpful assistant. Answer the question using ONLY the provided context.
If the answer is not found in the context, say "I don't have enough information to answer this."
Do not use any knowledge outside of the provided context.
Always mention which source(s) you used.

CONTEXT:
{context_section}

QUESTION:
{question}

ANSWER:"""


# ═══════════════════════════════════════════════════════════════════
# STEP 6 — RAG PIPELINE (PUTTING IT ALL TOGETHER)
# ═══════════════════════════════════════════════════════════════════

class RAGPipeline:
    """Full RAG pipeline: index documents, answer questions."""

    def __init__(self, pinecone_api_key: str, anthropic_api_key: str):
        self.loader    = DocumentLoader()
        self.splitter  = TextSplitter(chunk_size=CHUNK_SIZE, overlap=CHUNK_OVERLAP)
        self.embedder  = EmbeddingEngine(EMBEDDING_MODEL)
        self.store     = VectorStore(pinecone_api_key, PINECONE_INDEX, self.embedder.dimension)
        self.llm       = anthropic.Anthropic(api_key=anthropic_api_key)

    # ── Indexing ─────────────────────────────────────────────────

    def index_file(self, file_path: str) -> int:
        """Load, chunk, embed, and store a single file."""
        print(f"\n📄 Indexing: {file_path}")

        # 1. Load
        if file_path.endswith(".pdf"):
            text = self.loader.load_pdf(file_path)
        else:
            text = self.loader.load_text_file(file_path)
        print(f"  Loaded {len(text)} characters")

        # 2. Chunk
        chunks = self.splitter.split(text, source=file_path)
        print(f"  Split into {len(chunks)} chunks")

        # 3. Embed
        texts   = [c["text"] for c in chunks]
        vectors = self.embedder.embed_batch(texts)

        # 4. Store
        count = self.store.upsert_chunks(chunks, vectors)
        print(f"  ✅ Indexed {count} chunks from {file_path}")
        return count

    def index_directory(self, directory: str) -> int:
        """Index all documents in a directory."""
        docs = self.loader.load_directory(directory)
        total = 0
        for doc in docs:
            chunks  = self.splitter.split(doc["text"], source=doc["source"])
            texts   = [c["text"] for c in chunks]
            vectors = self.embedder.embed_batch(texts)
            total  += self.store.upsert_chunks(chunks, vectors)
        print(f"\n✅ Total indexed: {total} chunks from {len(docs)} documents")
        return total

    # ── Querying ─────────────────────────────────────────────────

    def query(self, question: str, top_k: int = TOP_K) -> Dict:
        """Answer a question using RAG."""
        print(f"\n🔍 Query: {question}")

        # 1. Embed the question
        query_vector = self.embedder.embed(question)

        # 2. Retrieve relevant chunks
        chunks = self.store.search(query_vector, top_k=top_k)
        print(f"  Retrieved {len(chunks)} chunks")
        for c in chunks:
            print(f"    Score {c['score']:.3f} | {c['source'][:50]}")

        # 3. Build prompt
        prompt = build_rag_prompt(question, chunks)

        # 4. Generate answer
        response = self.llm.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}]
        )
        answer = response.content[0].text

        return {
            "question":  question,
            "answer":    answer,
            "sources":   [c["source"] for c in chunks],
            "chunks":    chunks,
            "tokens_used": {
                "input":  response.usage.input_tokens,
                "output": response.usage.output_tokens,
            }
        }


# ── Usage Example ─────────────────────────────────────────────────

if __name__ == "__main__":
    rag = RAGPipeline(
        pinecone_api_key  = os.environ["PINECONE_API_KEY"],
        anthropic_api_key = os.environ["ANTHROPIC_API_KEY"],
    )

    # Index documents once
    rag.index_file("docs/employee_handbook.pdf")
    rag.index_file("docs/benefits_guide.txt")

    # Query anytime
    result = rag.query("How many vacation days do I get per year?")
    print(f"\nAnswer: {result['answer']}")
    print(f"Sources: {result['sources']}")
```

---

## Chunking Strategies

Chunking is one of the most impactful decisions in RAG. Bad chunking = bad retrieval = bad answers.

### Strategy Comparison

| Strategy | Best For | Chunk Size | Pros | Cons |
|---|---|---|---|---|
| **Fixed size** | Simple text, prototyping | 200–500 tokens | Simple, predictable | May cut mid-sentence |
| **Sentence** | Q&A, factual docs | 1–5 sentences | Semantically complete | Variable size |
| **Paragraph** | Articles, reports | 1–3 paragraphs | Preserves context | Can be too long |
| **Recursive** | General purpose | 200–500 tokens | Smart boundary detection | Slightly complex |
| **Semantic** | Any structured content | Variable | Highest quality | Requires embedding |
| **Document structure** | PDFs with headers | Sections | Respects hierarchy | Parsing complexity |

### Recursive Text Splitter (Recommended Default)

```python
class RecursiveTextSplitter:
    """
    Tries to split on natural boundaries in order:
    paragraphs → sentences → words → characters
    """

    SEPARATORS = ["\n\n", "\n", ". ", "! ", "? ", " ", ""]

    def __init__(self, chunk_size: int = 400, overlap: int = 80):
        self.chunk_size = chunk_size * 4   # chars
        self.overlap    = overlap * 4

    def split(self, text: str, source: str = "") -> List[Dict]:
        raw_chunks = self._recursive_split(text, self.SEPARATORS)
        chunks = []
        for i, chunk_text in enumerate(raw_chunks):
            if chunk_text.strip():
                chunks.append({
                    "id":        f"{source}_chunk_{i}",
                    "text":      chunk_text.strip(),
                    "source":    source,
                    "chunk_idx": i,
                })
        return chunks

    def _recursive_split(self, text: str, separators: List[str]) -> List[str]:
        if len(text) <= self.chunk_size:
            return [text]

        separator = separators[0] if separators else ""

        parts = text.split(separator) if separator else list(text)
        chunks = []
        current = ""

        for part in parts:
            candidate = current + separator + part if current else part

            if len(candidate) <= self.chunk_size:
                current = candidate
            else:
                if current:
                    chunks.append(current)
                    # Start new chunk with overlap
                    overlap_text = current[-self.overlap:] if len(current) > self.overlap else current
                    current = overlap_text + separator + part if overlap_text else part
                else:
                    # Part itself is too long — recurse with finer separator
                    if len(separators) > 1:
                        sub_chunks = self._recursive_split(part, separators[1:])
                        chunks.extend(sub_chunks[:-1])
                        current = sub_chunks[-1] if sub_chunks else ""
                    else:
                        chunks.append(part)
                        current = ""

        if current:
            chunks.append(current)

        return chunks
```

### Semantic Chunking (Highest Quality)

```python
def semantic_chunking(text: str, embedder, threshold: float = 0.85) -> List[str]:
    """
    Split at points where sentence similarity drops below the threshold.
    Groups semantically similar sentences into the same chunk.
    """
    sentences = re.split(r'(?<=[.!?])\s+', text)
    if len(sentences) <= 1:
        return sentences

    # Embed all sentences
    embeddings = embedder.embed_batch(sentences)

    # Calculate cosine similarity between adjacent sentences
    from numpy import dot
    from numpy.linalg import norm

    def cosine(a, b):
        return dot(a, b) / (norm(a) * norm(b))

    chunks = []
    current_chunk = [sentences[0]]

    for i in range(1, len(sentences)):
        similarity = cosine(embeddings[i - 1], embeddings[i])

        if similarity >= threshold:
            current_chunk.append(sentences[i])   # same topic — keep together
        else:
            chunks.append(" ".join(current_chunk))
            current_chunk = [sentences[i]]        # topic shift — new chunk

    if current_chunk:
        chunks.append(" ".join(current_chunk))

    return chunks
```

### Chunk Size Guidelines

```
Too small (< 100 tokens):
  ✗ Missing context — retrieving a sentence tells the LLM too little
  ✗ More chunks to store and search
  ✗ Higher embedding costs

Too large (> 800 tokens):
  ✗ Irrelevant content dilutes the signal
  ✗ LLM context window fills up quickly
  ✗ Precision suffers — "found the right document, wrong part"

Sweet spot (200–500 tokens):
  ✓ Enough context for the LLM to reason
  ✓ Specific enough to retrieve the right passage
  ✓ Efficient use of context window
  
Rule of thumb: 
  Short factual docs  → 150–250 tokens
  Long narrative docs → 300–500 tokens
  Technical docs      → 400–600 tokens (denser concepts)
```

---

## Embedding Models

### Model Comparison

| Model | Provider | Dimensions | Speed | Quality | Cost |
|---|---|---|---|---|---|
| `all-MiniLM-L6-v2` | HuggingFace (local) | 384 | Very fast | Good | Free |
| `all-mpnet-base-v2` | HuggingFace (local) | 768 | Fast | Better | Free |
| `text-embedding-3-small` | OpenAI API | 1536 | Fast | Very good | $0.02/M tokens |
| `text-embedding-3-large` | OpenAI API | 3072 | Medium | Best | $0.13/M tokens |
| `embed-english-v3.0` | Cohere API | 1024 | Fast | Very good | $0.10/M tokens |

### Using OpenAI Embeddings

```python
import openai

class OpenAIEmbedder:
    def __init__(self, api_key: str, model: str = "text-embedding-3-small"):
        self.client = openai.OpenAI(api_key=api_key)
        self.model  = model

    def embed(self, text: str) -> List[float]:
        text = text.replace("\n", " ")
        response = self.client.embeddings.create(input=[text], model=self.model)
        return response.data[0].embedding

    def embed_batch(self, texts: List[str], batch_size: int = 100) -> List[List[float]]:
        texts = [t.replace("\n", " ") for t in texts]
        all_embeddings = []

        for i in range(0, len(texts), batch_size):
            batch    = texts[i:i + batch_size]
            response = self.client.embeddings.create(input=batch, model=self.model)
            embeddings = [item.embedding for item in sorted(response.data, key=lambda x: x.index)]
            all_embeddings.extend(embeddings)

        return all_embeddings
```

### Key Embedding Principle

```
The embedding model used at INDEX time MUST be the same model
used at QUERY time. Mixing models produces nonsensical similarity scores.

Index:  document → model_A → vector_A stored in DB
Query:  question → model_B → vector_B searched in DB
Result: garbage similarity scores ← NEVER do this
```

---

## Vector Databases

### Comparison

| DB | Type | Best For | Hosting | Standout Feature |
|---|---|---|---|---|
| **Pinecone** | Managed cloud | Production, large scale | Cloud only | Simplest managed option |
| **Weaviate** | Open source + cloud | Multi-modal RAG | Self-host or cloud | Built-in hybrid search |
| **Qdrant** | Open source + cloud | High performance | Self-host or cloud | Payload filtering |
| **pgvector** | PostgreSQL extension | Existing Postgres users | Self-host | No new infrastructure |
| **Chroma** | Open source | Local dev, prototyping | Local or cloud | Easiest to start with |
| **FAISS** | Library (Facebook) | Offline, no server needed | In-memory/local | Fastest for batch search |

### Chroma (Best for Local Development)

```python
import chromadb

class ChromaVectorStore:
    def __init__(self, persist_directory: str = "./chroma_db"):
        self.client     = chromadb.PersistentClient(path=persist_directory)
        self.collection = self.client.get_or_create_collection(
            name="documents",
            metadata={"hnsw:space": "cosine"}
        )

    def upsert_chunks(self, chunks: List[Dict], vectors: List[List[float]]) -> None:
        self.collection.upsert(
            ids        = [c["id"] for c in chunks],
            embeddings = vectors,
            documents  = [c["text"] for c in chunks],
            metadatas  = [{"source": c.get("source", "")} for c in chunks],
        )

    def search(self, query_vector: List[float], top_k: int = 5) -> List[Dict]:
        results = self.collection.query(
            query_embeddings = [query_vector],
            n_results        = top_k,
        )
        return [
            {
                "id":     results["ids"][0][i],
                "score":  1 - results["distances"][0][i],  # convert distance to similarity
                "text":   results["documents"][0][i],
                "source": results["metadatas"][0][i].get("source", ""),
            }
            for i in range(len(results["ids"][0]))
        ]
```

### pgvector (Best for PostgreSQL Shops)

```python
import psycopg2
from pgvector.psycopg2 import register_vector
import numpy as np

class PGVectorStore:
    def __init__(self, connection_string: str, dimension: int):
        self.conn      = psycopg2.connect(connection_string)
        self.dimension = dimension
        register_vector(self.conn)
        self._setup_table()

    def _setup_table(self):
        with self.conn.cursor() as cur:
            cur.execute("CREATE EXTENSION IF NOT EXISTS vector;")
            cur.execute(f"""
                CREATE TABLE IF NOT EXISTS document_chunks (
                    id          TEXT PRIMARY KEY,
                    text        TEXT NOT NULL,
                    source      TEXT,
                    embedding   vector({self.dimension})
                );
            """)
            cur.execute("""
                CREATE INDEX IF NOT EXISTS chunks_embedding_idx
                ON document_chunks
                USING ivfflat (embedding vector_cosine_ops)
                WITH (lists = 100);
            """)
            self.conn.commit()

    def upsert_chunks(self, chunks: List[Dict], vectors: List[List[float]]) -> None:
        with self.conn.cursor() as cur:
            for chunk, vector in zip(chunks, vectors):
                cur.execute("""
                    INSERT INTO document_chunks (id, text, source, embedding)
                    VALUES (%s, %s, %s, %s)
                    ON CONFLICT (id) DO UPDATE SET
                        text = EXCLUDED.text,
                        embedding = EXCLUDED.embedding;
                """, (chunk["id"], chunk["text"], chunk.get("source", ""), np.array(vector)))
        self.conn.commit()

    def search(self, query_vector: List[float], top_k: int = 5) -> List[Dict]:
        with self.conn.cursor() as cur:
            cur.execute("""
                SELECT id, text, source,
                       1 - (embedding <=> %s) AS similarity
                FROM document_chunks
                ORDER BY embedding <=> %s
                LIMIT %s;
            """, (np.array(query_vector), np.array(query_vector), top_k))

            return [
                {"id": row[0], "text": row[1], "source": row[2], "score": float(row[3])}
                for row in cur.fetchall()
            ]
```

---

## Retrieval Strategies

### 1. Dense Retrieval (Default)

Pure vector similarity search. Good for semantic matches.

```python
def dense_retrieve(query: str, embedder, store, top_k: int = 5) -> List[Dict]:
    query_vector = embedder.embed(query)
    return store.search(query_vector, top_k=top_k)
```

---

### 2. Hybrid Retrieval (Recommended for Production)

Combine dense (semantic) + sparse (keyword/BM25) search. Catches both conceptual and exact matches.

```python
from rank_bm25 import BM25Okapi

class HybridRetriever:
    def __init__(self, chunks: List[Dict], embedder, vector_store, alpha: float = 0.5):
        """
        alpha = weight for dense search (0.0 = pure BM25, 1.0 = pure dense)
        """
        self.chunks       = chunks
        self.embedder     = embedder
        self.vector_store = vector_store
        self.alpha        = alpha

        # Build BM25 index
        tokenised  = [c["text"].lower().split() for c in chunks]
        self.bm25  = BM25Okapi(tokenised)
        self.chunk_ids = {c["id"]: i for i, c in enumerate(chunks)}

    def retrieve(self, query: str, top_k: int = 5) -> List[Dict]:
        # Dense search
        query_vector  = self.embedder.embed(query)
        dense_results = self.vector_store.search(query_vector, top_k=top_k * 2)

        # BM25 search
        tokenised_query = query.lower().split()
        bm25_scores     = self.bm25.get_scores(tokenised_query)
        top_bm25_idx    = bm25_scores.argsort()[-top_k * 2:][::-1]

        # Build score maps
        dense_scores = {r["id"]: r["score"] for r in dense_results}
        bm25_max     = bm25_scores.max() or 1

        # Combine with weighted sum
        combined_scores = {}

        for result in dense_results:
            combined_scores[result["id"]] = self.alpha * result["score"]

        for idx in top_bm25_idx:
            chunk_id    = self.chunks[idx]["id"]
            norm_bm25   = bm25_scores[idx] / bm25_max
            if chunk_id in combined_scores:
                combined_scores[chunk_id] += (1 - self.alpha) * norm_bm25
            else:
                combined_scores[chunk_id]  = (1 - self.alpha) * norm_bm25

        # Sort and return top-k
        sorted_ids = sorted(combined_scores, key=combined_scores.get, reverse=True)[:top_k]

        return [
            {**self.chunks[self.chunk_ids[cid]], "score": combined_scores[cid]}
            for cid in sorted_ids
            if cid in self.chunk_ids
        ]
```

---

### 3. MMR — Maximum Marginal Relevance

Retrieve diverse results, avoiding redundant chunks about the same topic.

```python
import numpy as np
from numpy.linalg import norm

def mmr_retrieve(
    query_vector: List[float],
    candidate_chunks: List[Dict],
    candidate_vectors: List[List[float]],
    top_k: int = 5,
    lambda_param: float = 0.5,
) -> List[Dict]:
    """
    Balance relevance (to query) and diversity (from already selected chunks).
    lambda_param: 1.0 = pure relevance, 0.0 = pure diversity
    """
    q = np.array(query_vector)
    C = [np.array(v) for v in candidate_vectors]

    def cosine(a, b):
        return float(np.dot(a, b) / (norm(a) * norm(b) + 1e-9))

    selected_idx = []
    remaining    = list(range(len(candidate_chunks)))

    for _ in range(min(top_k, len(candidate_chunks))):
        mmr_scores = []
        for i in remaining:
            relevance = cosine(C[i], q)
            if selected_idx:
                redundancy = max(cosine(C[i], C[j]) for j in selected_idx)
            else:
                redundancy = 0
            mmr_scores.append(lambda_param * relevance - (1 - lambda_param) * redundancy)

        best = remaining[np.argmax(mmr_scores)]
        selected_idx.append(best)
        remaining.remove(best)

    return [candidate_chunks[i] for i in selected_idx]
```

---

### 4. Contextual Compression

Compress retrieved chunks to only the sentences directly relevant to the query:

```python
def compress_chunk(query: str, chunk_text: str, llm_client) -> str:
    """Extract only the sentences from a chunk that are relevant to the query."""

    prompt = f"""Extract ONLY the sentences from the text that are directly relevant
to answering the question. Return just those sentences, nothing else.
If no sentences are relevant, return "NOT_RELEVANT".

Question: {query}

Text: {chunk_text}

Relevant sentences:"""

    response = llm_client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=256,
        messages=[{"role": "user", "content": prompt}]
    )
    compressed = response.content[0].text.strip()
    return None if compressed == "NOT_RELEVANT" else compressed
```

---

## Prompt Construction

### The RAG Prompt Formula

```
A great RAG prompt has 5 elements:

1. Role definition       — who the LLM is
2. Grounding instruction — use ONLY provided context
3. Uncertainty handling  — what to do when context lacks the answer
4. Context block         — the retrieved chunks with source labels
5. Question              — the actual user question
```

### Production RAG Prompt Templates

```python
# Template 1: Strict grounding (customer support, legal, medical)
STRICT_RAG_PROMPT = """You are a {role}. Your answers must be based EXCLUSIVELY on
the provided documents. Do not use any external knowledge.

Rules:
- If the answer is in the documents, cite the source (e.g., "According to [Source 1]...")
- If the answer is NOT in the documents, say exactly: "I don't have information about this in the provided documents."
- Never guess or infer beyond what the documents explicitly state.

DOCUMENTS:
{context}

QUESTION: {question}

ANSWER:"""

# Template 2: Balanced (general knowledge assistant)
BALANCED_RAG_PROMPT = """You are a helpful assistant. Use the provided context as
your primary source. You may supplement with general knowledge when clearly indicated,
but always prioritise the context.

CONTEXT:
{context}

QUESTION: {question}

ANSWER (cite sources where used):"""

# Template 3: Analytical (research, synthesis tasks)
ANALYTICAL_RAG_PROMPT = """You are a research analyst. Synthesise information
from the provided sources to answer the question comprehensively.

Instructions:
- Integrate information from multiple sources when relevant
- Note any contradictions between sources
- Indicate your confidence level
- List all sources used

SOURCES:
{context}

RESEARCH QUESTION: {question}

ANALYSIS:"""


def format_context(chunks: List[Dict]) -> str:
    """Format retrieved chunks into a readable context block."""
    parts = []
    for i, chunk in enumerate(chunks, 1):
        source = Path(chunk.get("source", "Unknown")).stem
        score  = chunk.get("score", 0)
        parts.append(f"[Source {i}: {source} | Relevance: {score:.2f}]\n{chunk['text']}")
    return "\n\n" + "─" * 40 + "\n\n".join(parts)
```

---

## Advanced RAG Patterns

### 1. Query Expansion

Rewrite the query before retrieval to improve recall:

```python
def expand_query(original_query: str, llm_client) -> List[str]:
    """Generate multiple search queries from one user question."""

    prompt = f"""Generate 3 different search queries to retrieve information
that would help answer this question. Output one query per line, nothing else.

Original question: {original_query}

Search queries:"""

    response = llm_client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=200,
        messages=[{"role": "user", "content": prompt}]
    )

    queries = [q.strip() for q in response.content[0].text.strip().split("\n") if q.strip()]
    return [original_query] + queries   # include original


def retrieve_with_expansion(query: str, embedder, store, top_k: int = 5) -> List[Dict]:
    """Retrieve using multiple query variants, deduplicate, and rank."""
    from collections import defaultdict

    expanded = expand_query(query, llm_client)
    all_results = defaultdict(lambda: {"score": 0})

    for q in expanded:
        vector  = embedder.embed(q)
        results = store.search(vector, top_k=top_k)
        for r in results:
            # Take the max score across all query variants
            if r["score"] > all_results[r["id"]]["score"]:
                all_results[r["id"]] = r

    # Sort by score and return top-k unique chunks
    sorted_results = sorted(all_results.values(), key=lambda x: x["score"], reverse=True)
    return sorted_results[:top_k]
```

---

### 2. HyDE — Hypothetical Document Embeddings

Generate a hypothetical answer first, then search for real documents similar to it:

```python
def hyde_retrieve(query: str, embedder, store, llm_client, top_k: int = 5) -> List[Dict]:
    """
    HyDE: Generate a hypothetical answer → embed it → search for similar real docs.
    Dramatically improves recall for complex questions.
    """

    # Step 1: Generate a hypothetical answer
    hypothetical_prompt = f"""Write a 2-3 sentence factual answer to this question,
even if you are uncertain. Focus on the key concepts and terminology.

Question: {query}

Hypothetical answer:"""

    response = llm_client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=200,
        messages=[{"role": "user", "content": hypothetical_prompt}]
    )
    hypothetical_answer = response.content[0].text.strip()

    print(f"  HyDE hypothetical: {hypothetical_answer[:80]}...")

    # Step 2: Search using the hypothetical answer's embedding
    # (This vector lives in the "answer space" not the "question space")
    hyp_vector = embedder.embed(hypothetical_answer)
    return store.search(hyp_vector, top_k=top_k)
```

---

### 3. Re-Ranking

After initial retrieval, re-rank chunks for higher precision:

```python
def rerank_chunks(query: str, chunks: List[Dict], llm_client, top_k: int = 3) -> List[Dict]:
    """
    Use LLM to re-score and re-rank retrieved chunks.
    More accurate than cosine similarity alone.
    """

    scores = []
    for chunk in chunks:
        prompt = f"""Rate how relevant this document is for answering the question.
Score from 0-10. Reply with just the number.

Question: {query}

Document: {chunk['text'][:500]}

Relevance score (0-10):"""

        response = llm_client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=5,
            messages=[{"role": "user", "content": prompt}]
        )

        try:
            score = float(response.content[0].text.strip())
        except:
            score = 0.0

        scores.append({**chunk, "rerank_score": score})

    # Sort by re-rank score and return top-k
    return sorted(scores, key=lambda x: x["rerank_score"], reverse=True)[:top_k]
```

---

### 4. Parent-Child Chunking

Store small chunks for retrieval precision, but pass larger parent chunks to the LLM for context:

```python
class ParentChildChunker:
    """
    Index small child chunks (precise retrieval).
    Return larger parent chunks (rich context for LLM).
    """

    def __init__(self, parent_size: int = 1000, child_size: int = 200):
        self.parent_splitter = TextSplitter(chunk_size=parent_size, overlap=50)
        self.child_splitter  = TextSplitter(chunk_size=child_size,  overlap=30)
        self.parent_store    = {}   # id → parent chunk text

    def create_chunks(self, text: str, source: str) -> tuple:
        parents  = self.parent_splitter.split(text, source)
        children = []

        for parent in parents:
            self.parent_store[parent["id"]] = parent

            child_chunks = self.child_splitter.split(parent["text"], source)
            for child in child_chunks:
                child["parent_id"] = parent["id"]   # link to parent
                children.append(child)

        return parents, children

    def get_parent_context(self, child_chunk: Dict) -> str:
        """Given a retrieved child chunk, return its full parent context."""
        parent_id = child_chunk.get("parent_id")
        if parent_id and parent_id in self.parent_store:
            return self.parent_store[parent_id]["text"]
        return child_chunk["text"]   # fallback to child text
```

---

## Evaluation

### Key RAG Metrics

| Metric | What It Measures | Target |
|---|---|---|
| **Retrieval Precision** | % of retrieved chunks that are relevant | > 80% |
| **Retrieval Recall** | % of relevant chunks that were retrieved | > 70% |
| **Answer Faithfulness** | Does the answer stay grounded in context? | > 90% |
| **Answer Relevance** | Does the answer address the question? | > 85% |
| **Context Utilisation** | How much of the retrieved context was used? | > 60% |

### Automated Evaluation with RAGAS

```python
# pip install ragas

from ragas import evaluate
from ragas.metrics import (
    faithfulness,          # answer grounded in context?
    answer_relevancy,      # answer relevant to question?
    context_precision,     # retrieved chunks relevant?
    context_recall,        # all relevant chunks retrieved?
)
from datasets import Dataset

def evaluate_rag_pipeline(pipeline: RAGPipeline, test_cases: List[Dict]) -> Dict:
    """
    test_cases: [{"question": str, "ground_truth": str}, ...]
    """

    rows = {"question": [], "answer": [], "contexts": [], "ground_truth": []}

    for tc in test_cases:
        result = pipeline.query(tc["question"])
        rows["question"].append(tc["question"])
        rows["answer"].append(result["answer"])
        rows["contexts"].append([c["text"] for c in result["chunks"]])
        rows["ground_truth"].append(tc.get("ground_truth", ""))

    dataset = Dataset.from_dict(rows)
    scores  = evaluate(dataset, metrics=[
        faithfulness,
        answer_relevancy,
        context_precision,
        context_recall,
    ])

    return scores


# Test cases
TEST_CASES = [
    {
        "question": "How many vacation days do employees get?",
        "ground_truth": "Employees are entitled to 20 days of annual leave per year."
    },
    {
        "question": "What is the remote work policy?",
        "ground_truth": "Employees may work remotely up to 3 days per week with manager approval."
    },
]
```

---

## Production Best Practices

### Metadata Filtering

Add structured metadata to chunks for filtered retrieval:

```python
# When indexing — add rich metadata
chunk_with_metadata = {
    "id":          "handbook_sec4_chunk_0",
    "text":        "Employees are entitled to 20 days...",
    "source":      "employee_handbook.pdf",
    "metadata": {
        "document_type": "policy",
        "department":    "HR",
        "last_updated":  "2026-01-15",
        "section":       "4.2",
        "tags":          ["leave", "vacation", "annual"],
    }
}

# When querying — filter by metadata
results = index.query(
    vector=query_vector,
    top_k=5,
    filter={
        "document_type": {"$eq": "policy"},
        "last_updated":  {"$gte": "2025-01-01"},
    }
)
```

---

### Caching

```python
import hashlib
import json

class CachedRAGPipeline(RAGPipeline):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._query_cache = {}

    def query(self, question: str, top_k: int = TOP_K) -> Dict:
        # Cache key based on question + top_k
        cache_key = hashlib.md5(f"{question}:{top_k}".encode()).hexdigest()

        if cache_key in self._query_cache:
            print("  Cache hit ✓")
            return self._query_cache[cache_key]

        result = super().query(question, top_k)
        self._query_cache[cache_key] = result
        return result
```

---

### Index Freshness Strategy

```
Document types and recommended re-index frequency:

Static docs (contracts, policies):   Re-index on change only
Slow-changing docs (wikis, manuals):  Weekly full re-index
Fast-changing docs (news, logs):      Streaming upsert on write
Real-time data (prices, metrics):     Don't use RAG — use tool calls

Re-index approach:
  1. Build new index in parallel
  2. Run quality checks on new index
  3. Blue-green swap (point traffic to new index)
  4. Keep old index for 24h rollback window
```

---

### Observability Checklist

```python
# Log everything for debugging and evaluation

def query_with_observability(pipeline, question: str) -> Dict:
    import time

    start = time.time()
    result = pipeline.query(question)
    elapsed = time.time() - start

    log = {
        "question":          question,
        "latency_ms":        round(elapsed * 1000),
        "chunks_retrieved":  len(result["chunks"]),
        "top_chunk_score":   result["chunks"][0]["score"] if result["chunks"] else 0,
        "sources_used":      list(set(result["sources"])),
        "tokens_input":      result["tokens_used"]["input"],
        "tokens_output":     result["tokens_used"]["output"],
        "cost_usd":          (result["tokens_used"]["input"] * 3 +
                              result["tokens_used"]["output"] * 15) / 1_000_000,
    }

    print(json.dumps(log, indent=2))
    return result
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Chunks too large | LLM ignores most of the context | Reduce to 200–400 tokens |
| Chunks too small | Missing context, incomplete answers | Increase to 200–400 tokens, use parent-child |
| No overlap between chunks | Answers cut off at chunk boundaries | Add 15–20% overlap |
| Mismatch between index/query embedder | Nonsense similarity scores | Always use the same embedding model |
| Retrieving too few chunks (k=1) | Missing relevant content | Start with k=5, tune based on evals |
| Retrieving too many chunks (k=20) | LLM ignores most, high cost | Keep k ≤ 8 for most use cases |
| No metadata filtering | Returning stale/irrelevant docs | Add date, type, department filters |
| Not re-indexing updated docs | Stale answers | Implement incremental upsert on doc change |
| No faithfulness checking | LLM answers from training data | Instruct LLM to cite sources, verify in output |
| Prompt allows hallucination | LLM invents answers | Use strict grounding prompt: "answer ONLY from context" |

---

## Interview Cheat Sheet

**Q: What is RAG and why is it used?**
A: Retrieval-Augmented Generation connects an LLM to an external knowledge base. Before generating, the system retrieves relevant documents and injects them as context. It solves three LLM limitations: knowledge cutoff, lack of private/domain knowledge, and hallucination on unknown facts.

**Q: Walk me through the RAG pipeline.**
A: Two phases — indexing and querying. Indexing: load documents → chunk into segments → embed each chunk → store vectors in a vector DB. Querying: embed the user question → similarity search in vector DB → retrieve top-k chunks → inject chunks into LLM prompt → generate grounded answer.

**Q: What is chunking and why does it matter?**
A: Chunking splits documents into segments for embedding and retrieval. Too small = missing context; too large = irrelevant content dilutes the signal. The sweet spot is 200–500 tokens with 15–20% overlap. The recursive text splitter is a good default: tries paragraph → sentence → word boundaries.

**Q: What is hybrid retrieval?**
A: Combining dense (vector similarity) and sparse (BM25 keyword) search. Dense excels at semantic/conceptual matches; sparse excels at exact keyword matches. Combining both with a weighted sum gives better recall than either alone.

**Q: What is HyDE?**
A: Hypothetical Document Embeddings. Generate a hypothetical answer to the question, embed that answer, and search for real documents similar to it. Works because a hypothetical answer lives in the same vector space as real answers, not in the "question space".

**Q: How do you evaluate a RAG system?**
A: Four key RAGAS metrics — faithfulness (answer grounded in context?), answer relevancy (addresses the question?), context precision (retrieved chunks relevant?), context recall (all relevant chunks found?). Build a test set of question-answer pairs from your domain and run automated evaluation.

**Q: What is parent-child chunking?**
A: Index small child chunks for high-precision retrieval, but return the larger parent chunk to the LLM for richer context. Balances retrieval accuracy with generation quality.

**Q: How do you keep a RAG index fresh?**
A: Depends on content velocity — static docs re-index on change; slow-changing docs weekly; fast-changing use streaming upsert. Use a blue-green swap: build new index in parallel, validate it, then atomically switch traffic.

**Q: When would you use fine-tuning instead of RAG?**
A: Fine-tuning when the task requires consistent style/format, knowledge is stable and won't change, and you have 1,000+ high-quality examples. RAG when the knowledge base is large, changes frequently, or needs citations and explainability.

---

**Document Version:** 1.0  
**Last Updated:** February 2026  
**Author:** Ensemble mountain Team
*See also: [Router pattern guide](./02-router-pattern-guide.md)*
