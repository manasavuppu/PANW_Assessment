# PANW_Assessment

Hybrid exact + vector RAG system for structured network IP intelligence, combining deterministic lookup with FAISS-based semantic retrieval and grounded LLM answering.
---

## Hybrid RAG System for Network IP Intelligence

### Overview

This project implements a **hybrid retrieval system** over structured network IP data. It combines deterministic structured lookup with embedding-based semantic search to provide reliable and robust question answering over network records.

The system **prioritizes exact matching** for structured identifiers (e.g., service names, CIDR prefixes) and **falls back to vector similarity** when needed.

---

## System Architecture

The pipeline consists of:

1. **JSON ingestion**
2. **Ghost record injection** (per assignment requirement)
3. **Canonical chunk construction**
4. **Embedding generation**
5. **FAISS vector indexing**
6. **Hybrid retrieval** (Exact → Vector fallback)
7. **LLM-based answer generation**
8. **Zero-shot vs RAG evaluation comparison**

This design balances **deterministic precision** with **semantic flexibility**.

---

## Ingestion & Indexing Flow

### 1. JSON Download

The network dataset is retrieved via HTTP using `requests` with proper error handling.

### 2. Ghost Record Injection

A synthetic **"Project Omega"** record is inserted at index `0` as required.

### 3. Canonical Chunk Format

Each record is normalized into a structured text format:

```
service: <value> | ip_prefix: <value> | region: <value> | network_border_group: <value>
```

This ensures:

- Structured tokens remain searchable
- Embeddings retain semantic signal
- Retrieval remains consistent

### 4. Embedding Model

| Property   | Value                    |
|-----------|---------------------------|
| **Model** | all-MiniLM-L6-v2 `sentence-transformers`  |
| **Purpose** | Enable semantic matching for natural language queries |
| **Why**   | Structured identifiers alone are insufficient for fuzzy queries |

### 5. Vector Index

- **FAISS** `IndexFlatIP`
- Inner product similarity search
- Lightweight and sufficient for assignment scale

---

## Hybrid Retrieval Strategy

The system uses a **two-stage retrieval** process:

### Stage 1: Exact Structured Match (Deterministic)

If structured constraints are detected (e.g., service name, IP prefix, region):

- Pre-built inverted indexes are queried
- Regex-based constraint extraction is performed.
- Matching candidates are intersected
- Result returned with **score = 1.0**
- Retrieval method labeled "EXACT_AND"

**Why:** Network identifiers require deterministic handling. Embeddings alone are unreliable for structured tokens such as service names or CIDR prefixes.

### Stage 2: Vector Fallback (Semantic)

If no exact match is found:

- Query is embedded
- FAISS returns top-1 by similarity
- Embedding score is returned
- Retrieval method labeled "VECTOR"

**Why:** Handles schema mismatch and natural language variation, e.g.:

- *"Project Omega"* vs `PROJECT_OMEGA_UPLINK`
- Loose phrasing
- Indirect references

---

## Retrieval Score Semantics

| Path    | Score              | Meaning                          |
|---------|--------------------|----------------------------------|
| **Exact** | `1.0`            | Deterministic structured match   |
| **Vector** | Embedding similarity (inner product) | Semantic closeness |

> Scores between the two paths are **not directly comparable**.

---

## QA Generation

The top-1 retrieved chunk is passed to a **local LLM**.

**Prompt constraints** enforce:

- Output must be either:
  - An **IP address** exactly as shown in context
  - **"I don't know"**
- No extra text
- Regex extraction is used only as fallback normalization

**Why:** Strict formatting reduces hallucination risk and improves evaluation consistency.

---

## Evaluation

Two systems are compared:

| System | Description |
|--------|-------------|
| **System A** | Zero-shot — direct LLM answering without retrieval |
| **System B** | Hybrid RAG — exact-first + vector fallback + grounded generation |

Evaluation includes:

- Retrieval inspection via `debug_retrieval(question)`
- Top-1 chunk visibility
- Score reporting
- Judge-based correctness comparison

**Hybrid RAG** improves precision on structured queries.

---

## Production Considerations

### Scaling Retrieval

Current implementation uses:

- `IndexFlatIP` (O(N · d))

For production-scale datasets consider:

- FAISS **IVF** / **HNSW** indexes
- Managed vector databases
- Hybrid SQL + vector search
- **Longest-prefix matching** for CIDR containment
- Re-ranking layer (cross-encoder)
- Query result caching

### Deterministic Network Logic

In production systems, CIDR containment checks should use:

- Longest-prefix match
- `ipaddress` module for subnet inclusion
- Support hierarchical prefix resolution

> This assignment version keeps string-based matching for simplicity.

### Ambiguity Handling

Currently:

Exact match returns the first intersected candidate.

In production:

- Multiple matches should trigger clarification or ranking logic.
- Confidence scoring should combine constraint strength + semantic score.

### IPv6 Support

The current dataset primarily uses **IPv4** prefixes.

The architecture supports **IPv6** without structural changes:

- Canonical chunk format is generic
- Vector retrieval is schema-agnostic
- Prompting does not restrict IP version

---

## Setup

1. **Install dependencies:**

   ```bash
   pip install -r requirements.txt
   ```

2. **Run notebook cells sequentially.**

---

## Design Rationale Summary

- **Exact-first retrieval** ensures deterministic correctness for structured identifiers.
- **Vector fallback** handles natural language variation.
- **Strict output formatting** minimizes hallucination.
- **Hybrid design** balances precision and recall.
- Architecture is **extensible** to production-scale improvements.
