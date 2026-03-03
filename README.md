# PANW Assessment – Hybrid RAG for Network IP Intelligence

A **hybrid exact + vector** Retrieval-Augmented Generation (RAG) system for querying structured network IP data. The system combines deterministic structured retrieval with FAISS-based semantic search and a local LLM to answer natural language questions about network records.

---

## Overview

Network infrastructure data contains highly structured identifiers such as:

- **Service names**
- **CIDR prefixes**
- **Regions**
- **Network border groups**

Pure semantic retrieval is unreliable for these identifiers. This system therefore **prioritizes deterministic matching first**, restricts semantic retrieval to queries that do not contain structured constraints.

The result is a hybrid architecture that provides both:

- **Precision** for structured queries  
- **Flexibility** for natural language queries  

---

## System Architecture

The pipeline consists of the following stages:

1. **JSON ingestion**
2. **Ghost record injection** (assignment requirement)
3. **Canonical chunk construction**
4. **Embedding generation**
5. **FAISS vector indexing**
6. **Hybrid retrieval** (Exact → Controlled Vector fallback)
7. **Grounded LLM answer generation**
8. **Zero-shot vs RAG evaluation**

---

## Data Ingestion

- The AWS IP ranges dataset is retrieved using **HTTP** with error handling.
- A synthetic record representing **Project Omega** is injected at index `0` to satisfy the assignment specification.

---

## Canonical Chunk Format

Each network record is converted into a normalized textual representation:

```
service: <value> | ip_prefix: <value> | region: <value> | network_border_group: <value>
```

This format preserves structured fields while remaining compatible with embedding models.

**Benefits:**

- Deterministic token lookup
- Semantic representation for embeddings
- Consistent retrieval inputs

---

## Layer 1 – Deterministic Structured Indexes

This layer builds **inverted indexes** for structured fields:

| Field | Purpose |
|-------|---------|
| `service` | Service name lookup |
| `ip_prefix` | CIDR / prefix lookup |
| `region` | Region filter |
| `network_border_group` | NBG filter |

Each field value maps to the set of records containing that value. These indexes enable fast deterministic retrieval before any semantic search is attempted.

Structured identifiers are **canonicalized** to avoid mismatches caused by:

- Casing differences
- Whitespace variations
- Punctuation differences

This ensures reliable exact matching for network identifiers.

---

## Layer 2 – Semantic Retrieval (Embeddings + FAISS)

Each canonical chunk is embedded using:

| Property | Value |
|----------|--------|
| **Model** | `all-MiniLM-L6-v2` |
| **Library** | `sentence-transformers` |
| **Embedding dimension** | 384 |

Embeddings are:

- **L2 normalized**
- Stored as **float32**
- Indexed in **FAISS IndexFlatIP** (inner product ≡ cosine similarity when normalized)

FAISS provides efficient nearest-neighbor search for semantic queries.

---

## Layer 3 – Hybrid Retrieval Logic

This layer extracts structured constraints from the query and determines the retrieval strategy.

### Constraint Extraction

Detects:

- **CIDR prefixes**
- **IPv4 addresses**
- **Regions**
- **Network border group** tokens
- **Service identifiers**

Service tokens are normalized to tolerate variations such as:

- `route 53` / `ROUTE-53` / `route_53` → canonical form

Optional **fuzzy matching** is applied with a high similarity threshold to correct minor typos without mapping unrelated services.

### Deterministic Retrieval

If structured constraints are detected:

1. Candidate sets are retrieved from Layer 1 indexes.
2. Sets are intersected using **AND** logic.
3. A deterministic result is returned.

**Example retrieval mode:**

```
EXACT_AND(service=EC2, region=us-east-1)
```

This path returns **score = 1.0** because it represents a deterministic match.

### IP Address Resolution

For raw IP queries, the system performs **CIDR containment** checks using Python’s `ipaddress` module. It applies **Longest Prefix Match (LPM)** to identify the most specific prefix containing the queried IP, mirroring real network routing behavior.

### Controlled Semantic Fallback

Vector search is **only** used when no structured constraints exist in the query. This prevents semantic search from incorrectly substituting structured identifiers.

**Example semantic queries:**

- *"What is the IP address for Project Omega?"*
- *"Which network record corresponds to this service?"*

Vector retrieval returns:

- **mode:** `VECTOR`
- **score:** embedding similarity

---

## Retrieval Score Semantics

| Retrieval Path | Score | Meaning |
|----------------|-------|---------|
| **Exact match** | `1.0` | Deterministic structured match |
| **Vector match** | similarity value | Semantic closeness |

> Scores between paths should **not** be directly compared.

---

## Grounded LLM Answer Generation

The retrieved chunk is passed to a **local language model**. The prompt instructs the model to:

- Answer **only** using the provided context
- Return **"I don't know"** when the answer is absent

Generation is **deterministic**. The prompt enforces grounded answers and deterministic generation (do_sample=False) to minimize hallucination and ensure consistent evaluation.

---

## Evaluation

Two systems are evaluated:

| System | Description |
|--------|-------------|
| **System A** | Zero-shot LLM answering without retrieval |
| **System B** | Hybrid RAG pipeline |

**Evaluation steps:**

1. Run retrieval inspection (`debug_retrieval`)
2. Generate answers
3. Compare outputs using a **judge** function

The judge checks whether the expected IP **`99.99.99.99`** appears in the answer.

---

## Observability Utilities

The notebook includes debugging tools:

### `debug_retrieval(question)`

Displays:

- Top retrieved chunk
- Similarity score
- Retrieval mode

### `debug_retrieval_verbose(question)`

Displays:

- Extracted constraints
- Candidate sets
- Retrieval path
- Retrieved chunk
- Score

These utilities help validate retrieval behavior independently of the LLM.

---

## Production Considerations

### Scalable Vector Search

For larger datasets consider:

- **FAISS IVF / HNSW** indexes
- Managed vector databases
- Hybrid **SQL + vector** retrieval

### Ranking Improvements

Possible extensions:

- Cross-encoder reranking
- Multi-result retrieval
- Confidence calibration

### Caching

Frequent deterministic queries can be **cached** to reduce latency.

### Observability

Production systems should log:

- Retrieval mode
- Constraint extraction statistics
- Latency metrics

---

## Setup

1. **Install dependencies:**

   ```bash
   pip install -r requirements.txt
   ```

2. **Run the notebook sequentially.**

---

## Design Summary

The system demonstrates a **hybrid retrieval architecture** tailored for structured network data.

**Key design principles:**

- **Deterministic retrieval** for structured identifiers  
- **Semantic retrieval** only when appropriate  
- **Grounded LLM** generation  
- **Modular** debugging and evaluation tools  

This architecture balances **precision**, **robustness**, and **extensibility** for network intelligence applications.
