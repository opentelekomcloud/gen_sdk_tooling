# Architecture Variants

Four approaches to building the SDK generation pipeline. All share the same core: RST parser + IR model + code generator. The difference is in **who triggers the process**, **how much documentation is processed**, and **who controls the output**.

---

## Variant A — MCP as Chat Interface (Anton)

> Source: Original proposal

The developer connects an MCP server to an LLM client (Claude, Kimi) and interacts via chat. The LLM calls tools sequentially, the developer sees each step.

```
Developer: "Generate SDK for CCE"
        ↓
LLM Client (Claude / Kimi)
        ↓
┌─────────────────────────────────┐
│         MCP Server              │
│                                 │
│  list_doc_services              │
│  select_service("CCE")          │
│  ingest_api_docs(ref="main")    │  ← full clone of docs repo
│  build_api_ir()                 │  ← parse ALL RST files
│  get_api_ir()                   │
│  diff_api_ir()                  │  ← compare with previous snapshot
│  explain_diff()                 │  ← RAG-powered explanation
│  select_working_repo()          │
│  generate_sdk_skeleton()        │
│  create_pull_request()          │
└─────────────────────────────────┘
        ↓
Pull Request
```

### Tools (11)

| Tool | Purpose |
|------|---------|
| `list_doc_services` | Discover services from opentelekomcloud-docs |
| `select_service` | Set active service for session |
| `ingest_api_docs` | Fetch and store all RST files |
| `list_ingested_docs` | List downloaded files and sections |
| `build_api_ir` | Parse RST → Intermediate Representation |
| `get_api_ir` | Return current IR |
| `diff_api_ir` | Compare IR versions |
| `explain_diff` | RAG-powered change explanation |
| `select_working_repo` | Set target SDK repository |
| `generate_sdk_skeleton` | Generate code from IR |
| `create_pull_request` | Create PR with generated code |

### Pros

- Full developer control at every step
- Good for initial SDK generation from scratch (new service)
- Interactive — developer can ask LLM to clarify, retry, adjust
- Natural fit for MCP (this is what the protocol was designed for)

### Cons

- Manual process — someone has to initiate and monitor
- Doesn't scale to 100+ repositories
- Full ingestion every time is slow
- No automatic reaction to documentation changes

### Best for

First-time SDK generation for a new service.

---

## Variant B — Event-Driven Pipeline (Sergey)

> Source: Anton's message after discussion with Sergey

Merge into opentelekomcloud-docs triggers an automated pipeline. Agents process the diff and generate code without human intervention.

```
Merge into opentelekomcloud-docs
        ↓
Trigger (GitHub webhook / Actions)
        ↓
Git diff of last merge
        ↓
Filter: did api-ref/ change?
  ├── No  → stop
  └── Yes ↓
        ↓
Agent 1: Diff → change spec
  "Added parameter proxy_mode to POST /v3/clusters"
  "Removed field legacy_flag from GET /v2/nodes"
        ↓
Agent 2..N: Spec → generated code
  (using Kimi-K2.5 or similar model)
        ↓
Pull Request (automatic)
```

### Tools (not MCP — regular scripts/agents)

| Component | Purpose |
|-----------|---------|
| GitHub webhook listener | Detect merges |
| Git diff analyzer | Extract changed RST files |
| RST parser | Parse changed files into structured data |
| Spec agent (LLM) | Formulate change specification |
| Code agent (LLM) | Generate SDK code from spec |
| PR creator | Push branch and open PR |

### Pros

- Fully automated — no manual intervention
- Reacts to changes immediately
- Processes only the diff (fast, focused)
- Scales to 100+ repositories

### Cons

- **No human-in-the-loop** — errors accumulate silently
- Parser mistakes → bad spec → bad code → merged PR → drift
- Reviewers develop "approval fatigue" over time
- Cannot generate SDK from scratch (only incremental changes)
- Debugging is hard — "why does this code look like this?"
- Heavy LLM dependency — agents make decisions without oversight

### Best for

Incremental updates after initial SDK exists — **but only with review safeguards**.

### Risk

> Without active oversight, the pipeline becomes a machine that produces plausible-looking but subtly broken code. Each error is small. Accumulated over months, the codebase becomes unmaintainable.

---

## Variant C — Hybrid (Proposed)

Combines automatic detection from Variant B with human control from Variant A. Supports two flows: first-time SDK generation for new services, and incremental updates for existing ones.

### Flow 1 — New Service Detected

Trigger monitors not only changes in existing repos, but also new repos or first-time `api-ref/` appearance.

```
New repo in opentelekomcloud-docs
  OR existing repo gets api-ref/source/ for the first time
        ↓
Trigger (GitHub Actions / scheduled scan)
        ↓
Create GitHub Issue (automatic):
  "🆕 New service detected: document-database-service
   - Repository: opentelekomcloud-docs/document-database-service
   - api-ref/source/ contains 24 RST files
   - No matching SDK module in python-otcextensions
   - Action required: initial SDK generation"
        ↓                              ↓
  [Developer reviews issue]    OR   [Ignored / closed]
        ↓
  Developer opens MCP chat:
  "Generate SDK for new service from issue #38"
        ↓
┌─────────────────────────────────┐
│         MCP Server              │
│                                 │
│  ingest_api_docs(service)       │  ← full clone, all RST files
│  build_api_ir()                 │  ← parse entire service
│  get_api_ir()                   │  ← developer reviews IR
│  generate_sdk_skeleton()        │  ← full: Resource, Proxy, entry points
│  create_pull_request(draft)     │  ← draft PR for review
└─────────────────────────────────┘
        ↓
Developer reviews draft PR
  - checks generated Resource classes
  - checks Proxy methods
  - checks entry points in setup.cfg
        ↓
Merge / request changes
```

### Flow 2 — Existing Service Updated

Same as before — trigger catches merges that change `api-ref/` in known services.

```
Merge into opentelekomcloud-docs
        ↓
Trigger (GitHub Actions)
        ↓
Git diff of last merge
        ↓
Filter: did api-ref/ change?
  ├── No  → stop
  └── Yes ↓
        ↓
Create GitHub Issue (automatic):
  "📋 CCE docs updated — 3 files changed in api-ref/
   - clusters.rst: new parameter proxy_mode
   - nodes.rst: removed field legacy_flag
   - diff summary attached"
        ↓                              ↓
  [Developer reviews issue]    OR   [Ignored / closed]
        ↓
  Developer opens MCP chat:
  "Process issue #42"
        ↓
┌─────────────────────────────────┐
│         MCP Server              │
│                                 │
│  fetch_merge_diff(issue=#42)    │  ← only changed files
│  build_api_ir()                 │  ← parse changed RST
│  diff_api_ir()                  │  ← compare with existing IR
│  explain_diff()                 │  ← RAG explanation
│  generate_code()                │  ← patch: only affected files
│  create_pull_request(draft)     │  ← draft PR for review
└─────────────────────────────────┘
        ↓
Developer reviews draft PR → merge / request changes
```

### Trigger Detection Logic

```
For each repo in opentelekomcloud-docs:
  │
  ├── New repo appeared?
  │     └── Has api-ref/source/?
  │           └── Yes → Issue: "🆕 New service detected" (Flow 1)
  │
  ├── Merge in existing repo?
  │     └── Diff touches api-ref/?
  │           └── Yes → Issue: "📋 Service updated" (Flow 2)
  │
  └── No relevant changes → skip
```

### Tools (MCP)

| Tool | Flow | Purpose |
|------|------|---------|
| `list_doc_services` | 1 | Discover services from opentelekomcloud-docs |
| `ingest_api_docs` | 1 | Full ingestion of all RST files for a service |
| `fetch_merge_diff` | 2 | Get only changed RST files from a merge/issue |
| `build_api_ir` | 1, 2 | Parse RST → IR (full or partial) |
| `get_api_ir` | 1, 2 | Return current IR for review |
| `diff_api_ir` | 2 | Compare IR versions |
| `explain_diff` | 2 | RAG-powered change explanation |
| `generate_sdk_skeleton` | 1 | Generate full SDK: Resource, Proxy, entry points |
| `generate_code` | 2 | Generate incremental code changes |
| `create_pull_request` | 1, 2 | Create draft PR |

### Pros

- Automatic detection — no need to monitor 100+ repos manually
- Catches both new services and updates to existing ones
- Human reviews before code generation — catches parser errors early
- Developer sees intermediate results, can intervene
- Draft PRs — additional review gate before merge
- MCP-native — fits the protocol design
- Two clear flows — no ambiguity about what happens when

### Cons

- Most complex to build (trigger + MCP server + issue integration + two flows)
- Still requires developer attention (not fully hands-off)
- Slower than Variant B (human in the loop adds latency)

### Best for

Production use — covers the full lifecycle from new service discovery to ongoing maintenance, with quality control at every step.

---

## Variant C2 — ChatOps in GitHub (Proposed)

> Evolution of Variant C: replace the MCP chat with GitHub issue commands

Same automatic detection as Variant C, but instead of switching to an MCP chat, the developer controls the pipeline directly from GitHub issue comments. MCP server is still used under the hood — but the client is a GitHub Actions script, not a human in a chat window.

### Flow 1 — New Service

```
New repo / new api-ref/ detected
        ↓
GitHub Issue (automatic):
  "🆕 New service: document-database-service
   - 24 RST files in api-ref/source/
   - No matching SDK module"
        ↓
Developer writes comment:
  /generate --full
        ↓
GitHub Actions picks up command
        ↓
MCP Client (script) calls MCP Server:
  ingest_api_docs → build_api_ir → generate_sdk_skeleton
        ↓
Bot comments on issue (intermediate results):
  "🔍 Parsed 24 files
   ✅ 22 endpoints extracted
   ⚠️ 2 files skipped (no URI section)
   📝 IR: 8 resources, 3 sub-resources"
        ↓
Bot comments:
  "✅ Draft PR #87 created — 12 files generated"
        ↓
Developer reviews PR
```

### Flow 2 — Incremental Update

```
Merge into opentelekomcloud-docs
        ↓
GitHub Issue (automatic):
  "📋 CCE updated — 3 files changed
   - clusters.rst: +1 parameter
   - nodes.rst: -1 field"
        ↓
Developer writes comment:
  /generate              ← incremental (default)
        ↓
GitHub Actions → MCP Client → MCP Server:
  fetch_merge_diff → build_api_ir → diff_api_ir → generate_code
        ↓
Bot comments:
  "🔍 Parsed 3 changed files
   📋 Changes:
     + POST /v3/clusters: new param proxy_mode (string, optional)
     - GET /v2/nodes: removed field legacy_flag
   ✅ Draft PR #92 created"
        ↓
Developer reviews PR
```

### Available Commands

| Command | Effect |
|---------|--------|
| `/generate` | Incremental — process diff only |
| `/generate --full` | Full — ingest all docs, generate complete SDK |
| `/generate --dry` | Dry run — parse and show IR, no code generation |
| `/cancel` | Cancel running generation |
| `/status` | Show current pipeline state |

### Under the Hood

```
GitHub Issue Comment ("/generate")
        ↓
GitHub Actions workflow
        ↓
┌──────────────────────────────────────────────────┐
│  MCP Client (Python script in Actions runner)    │
│                                                  │
│  async with mcp_client.connect(server) as ctx:   │
│      result = await ctx.call("build_api_ir")     │
│      post_github_comment(result.summary)         │
│      code = await ctx.call("generate_code")      │
│      pr = await ctx.call("create_pull_request")  │
│      post_github_comment(f"PR {pr.url}")         │
└──────────────────────────────────────────────────┘
        ↓
Same MCP Server, same tools as Variant C
```

The MCP server does not know or care whether it's called from a chat client or a GitHub Actions script. The tools are identical.

### MCP Chat as Fallback

The MCP chat is not removed — it remains available for:

- **Complex cases** — when `/generate` fails and you need to debug interactively
- **Exploration** — "show me the IR for this service", "what would change if I re-parse?"
- **Edge cases** — OpenStack API sections, unusual RST structures

```
Normal flow:     Issue → /generate → bot comment → PR → review
Exception flow:  Issue → /generate → ⚠️ "Failed: ambiguous types in 5 fields"
                        → developer opens MCP chat for interactive debug
```

### Tools (MCP — same as Variant C)

| Tool | Flow | Purpose |
|------|------|---------|
| `list_doc_services` | 1 | Discover services |
| `ingest_api_docs` | 1 | Full ingestion |
| `fetch_merge_diff` | 2 | Changed files from merge |
| `build_api_ir` | 1, 2 | Parse RST → IR |
| `get_api_ir` | 1, 2 | Return IR |
| `diff_api_ir` | 2 | Compare IR versions |
| `explain_diff` | 2 | RAG explanation |
| `generate_sdk_skeleton` | 1 | Full SDK generation |
| `generate_code` | 2 | Incremental code changes |
| `create_pull_request` | 1, 2 | Draft PR |

### Pros

- **Everything lives in GitHub** — no context switching to a chat client
- Issue = decision log (who triggered what, when, with which result)
- Intermediate results visible to the whole team, not just one developer
- MCP chat available as fallback for complex cases
- Same MCP server and tools — no duplication
- Familiar interface for developers (slash commands, like `/deploy`)

### Cons

- GitHub Actions adds latency and complexity (runner setup, secrets)
- Less interactive than chat — no back-and-forth mid-pipeline
- Slash command parsing needs to be robust
- Debugging through issue comments is clunkier than chat

### Best for

Team workflow — multiple developers can monitor, trigger, and review without sharing an MCP chat session.

---

## Comparison

| Aspect | A (Anton) | B (Sergey) | C (Hybrid) | C2 (ChatOps) |
|--------|-----------|------------|------------|---------------|
| **Trigger** | Manual | Merge webhook | Merge → issue → MCP chat | Merge → issue → `/generate` |
| **Interface** | LLM chat | None | LLM chat | GitHub issues |
| **Scope** | Full docs | Diff only | Both | Both |
| **Human control** | Full | None | At decision points | At decision points |
| **Scales to 100+ repos** | ❌ | ✅ | ✅ | ✅ |
| **Initial SDK generation** | ✅ | ❌ | ✅ | ✅ |
| **Incremental updates** | Slow | ✅ | ✅ | ✅ |
| **Error accumulation risk** | Low | High | Low | Low |
| **Team visibility** | One person | PR only | One person | Whole team |
| **MCP usage** | Core | Not needed | Core | Core (headless) |
| **LLM dependency** | Optional | Heavy | Moderate | Moderate |
| **Complexity to build** | Medium | Medium | High | High |

---

## Shared Core (all variants)

Regardless of the chosen architecture, these components are identical:

1. **RST Parser** — docutils-based, extracts endpoints from RST tables
2. **IR Model** — ServiceModel → ResourceModel → PropertyModel
3. **Code Generator** — Jinja2 templates for otcextensions-compatible code

The architectural decision affects orchestration, not the core engine.

---

## Recommendation

Start with the **RST parser prototype** (shared by all variants). Parse 3-5 real services, identify edge cases and limitations. Then make the architecture decision with concrete data.

The parser prototype does not depend on the chosen variant — it is the foundation for all four.

A possible evolution path:

```
RST Parser prototype
        ↓
Variant A (MCP chat) — validate the full pipeline works interactively
        ↓
Variant C2 (ChatOps) — wrap the working pipeline in GitHub automation
        ↓
Variant B tendencies — if confidence in the parser is high enough,
                       some low-risk updates could be auto-approved
```
