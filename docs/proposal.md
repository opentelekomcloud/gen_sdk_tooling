# Gen SDK Tooling - Proposal

**Python SDK/(CLI?) Generator using RAG + MCP**

> Source: Anton Sidelnikov  
> Status: Approved  
> Date: January 2026

---

## Background

This project aims to build an automated system for generating and maintaining Python SDKs and CLIs based on API documentation from Open Telekom Cloud.

The system combines:
- **RAG (Retrieval-Augmented Generation)** — for grounded explanations and traceability to documentation
- **MCP (Model Context Protocol)** — as a standardized interface between an LLM agent and tools such as parsers, diff engines, and GitHub automation

### Core Principle

> **Deterministic data extraction and diffing first, LLM-assisted reasoning second.**

---

## Phase 1 — MCP Server Skeleton

**Goal:** Create an MCP server that defines state, tool boundaries, and execution model.

### MCP Server Responsibilities (global)

- Maintain workspace state:
  - selected documentation repo
  - selected working SDK/CLI repo
  - current API IR snapshot
- Execute deterministic tools
- Provide structured outputs

### Tool: `list_doc_services`

**Purpose:** Discover available services from opentelekomcloud-docs.

```json
{
  "name": "list_doc_services",
  "input": {},
  "output": {
    "services": [
      {
        "service": "cloud-container-engine",
        "repo": "opentelekomcloud-docs/cloud-container-engine",
        "has_api_ref": true
      }
    ]
  }
}
```

**Internal Logic:**
- Call GitHub API
- Check for `api-ref/source`

### Tool: `select_service`

**Purpose:** Set active service for all further operations.

```json
{
  "name": "select_service",
  "input": {
    "service": "cloud-container-engine"
  },
  "output": {
    "status": "ok",
    "api_ref_path": "api-ref/source"
  }
}
```

**State Change:** MCP server stores selected service in session context.

---

## Phase 2 — MCP-Driven Documentation Ingestion

### Tool: `ingest_api_docs`

**Purpose:** Fetch and version raw API documentation.

```json
{
  "name": "ingest_api_docs",
  "input": {
    "ref": "main"
  },
  "output": {
    "commit": "a1b2c3d",
    "files_ingested": 42
  }
}
```

**Internal Logic:**
- Clone / fetch doc repo
- Read all files under `api-ref/source`
- Store:
  - raw text
  - metadata (file, section, commit hash)

### Tool: `list_ingested_docs`

```json
{
  "name": "list_ingested_docs",
  "input": {},
  "output": {
    "documents": [
      {
        "file": "clusters.rst",
        "sections": ["List clusters", "Create cluster"]
      }
    ]
  }
}
```

---

## Phase 3 — MCP-Driven API IR Construction

**Goal:** Convert docs → Intermediate Representation via MCP tools.

### Tool: `build_api_ir`

**Purpose:** Parse raw docs into a structured API model.

```json
{
  "name": "build_api_ir",
  "input": {},
  "output": {
    "service": "CCE",
    "version": "v3",
    "endpoints": 27,
    "models": 14
  }
}
```

**Internal Logic:**
- Deterministic parsers:
  - RST table parser
  - Code block extractor
- Build canonical IR
- Assign stable `operation_id`

**Optional LLM Usage:**
- Only if parser marks a section as ambiguous
- LLM output must be validated against schema

### Tool: `get_api_ir`

```json
{
  "name": "get_api_ir",
  "input": {},
  "output": {
    "api_ir": { ... }
  }
}
```

---

## Phase 4 — MCP-Driven Diff & Change Classification

**Goal:** Detect changes inside MCP, not in external scripts.

### Tool: `diff_api_ir`

**Purpose:** Compare current IR with previous snapshot.

```json
{
  "name": "diff_api_ir",
  "input": {},
  "output": {
    "breaking": false,
    "summary": {
      "endpoints_added": 1,
      "params_added": 2
    }
  }
}
```

- Version-aware
- Output is machine-readable

### Tool: `explain_diff` (RAG-enabled)

**Purpose:** Explain why changes were detected.

```json
{
  "name": "explain_diff",
  "input": {
    "endpoint": "create_cluster"
  },
  "output": {
    "explanation": "A new optional parameter `proxy_mode` was added.",
    "evidence": [
      {
        "file": "clusters.rst",
        "section": "Create cluster"
      }
    ]
  }
}
```

**This is where RAG is used:**
- Retrieve doc chunks
- Cite evidence
- No logic decisions

---

## Phase 5 — MCP as Code Generator Orchestrator

### Tool: `select_working_repo`

```json
{
  "name": "select_working_repo",
  "input": {
    "repo": "opentelekomcloud/python-sdk"
  },
  "output": {
    "status": "ok"
  }
}
```

### Tool: `generate_sdk_skeleton`

```json
{
  "name": "generate_sdk_skeleton",
  "input": {},
  "output": {
    "files_created": [
      "otc/cce/client.py",
      "otc/cce/models.py"
    ]
  }
}
```

**Includes:**
- service module
- client class
- auth/session integration
- base error handling

### Tool: `create_pull_request`

```json
{
  "name": "create_pull_request",
  "input": {
    "draft": false
  },
  "output": {
    "pr_url": "https://github.com/..."
  }
}
```

**PR Body is assembled from:**
- diff summary
- breaking-change classification
- RAG explanations + evidence

---

## Tech Stack

| Layer | Tools |
|-------|-------|
| MCP | modelcontextprotocol |
| GitHub | PyGithub, GitHub API |
| Parsing | docutils, sphinx |
| IR | pydantic, jsonschema |
| Diff | custom engine, deepdiff |
| RAG | FAISS/Qdrant + embeddings |
| LLM | Llama 3.3 or Qwen2.5 |