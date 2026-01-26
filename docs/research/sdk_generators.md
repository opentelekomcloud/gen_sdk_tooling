# Research: SDK Generators

## Key Finding

> **Neither OpenAPI Generator nor AutoRest are directly applicable** — both tools expect an OpenAPI specification, not RST documentation. A **hybrid code-based approach** on Python is recommended.

---

## OpenAPI Generator

Template-based approach using **Mustache templates**.

### Architecture

```
OpenAPI Spec (JSON/YAML)
        ↓
   io.swagger.v3 Parser
        ↓
   DefaultCodegen (Java)
   - fromModel() → CodegenModel
   - fromOperation() → CodegenOperation
        ↓
   PythonClientCodegen
        ↓
   Mustache Template Engine
        ↓
   Generated Python SDK
```

### Key Files

- Repository: https://github.com/OpenAPITools/openapi-generator
- Python templates: `modules/openapi-generator/src/main/resources/python/`
- Generator class: `languages/PythonClientCodegen.java`

---

## AutoRest

Pipeline architecture with RPC protocol between components.

### Architecture

```
OpenAPI Spec + readme.md
        ↓
   AutoRest Core (@autorest/core)
        ↓
   Modelerfour (preprocessing)
   - Schema deduplication
   - Naming normalization
   - code-model-v4.yaml output
        ↓
   JSON-RPC Protocol (stdin/stdout)
        ↓
   Python Extension (@autorest/python)
   - Generator written in Python!
   - m2r → namer → codegen pipeline
        ↓
   Generated Python SDK
```

### Key Repositories

- Core: https://github.com/Azure/autorest
- Python generator: https://github.com/Azure/autorest.python

---

## Comparison

| Aspect | OpenAPI Generator | AutoRest |
|--------|-------------------|----------|
| **Input** | OpenAPI spec | OpenAPI spec |
| **Generation logic** | Mustache templates | Native code (Python → Python) |
| **Learning curve** | Lower | Higher |
| **Flexibility** | Limited by Mustache | Full language power |
| **Debugging** | Difficult | Standard tools |
| **Customization** | Template overrides + Java | Plugin architecture |
| **Complex logic** | Requires Java extension | Native in code |

---

## Other SDK Generators

| Tool | Language Support | Customization | CI/CD Integration |
|------|------------------|---------------|-------------------|
| **OpenAPI Generator** | 50+ languages | Flexible Mustache templates, open-source | CLI (Java jar, Docker), Maven/Gradle plugins |
| **AutoRest** | 5+ languages (C#, Java, Python, TS, Go) | Pipeline architecture, plugins | CLI (Node.js, npx), Azure DevOps |
| **APIMatic** (SaaS) | Java, C#, PHP, Python, TS, Ruby | Web platform, GUI settings | Via API/CLI, external step |
| **Speakeasy** | TypeScript, Python, Go, Java, C#, PHP (beta) | "Idiomatic" SDKs + telemetry | CLI via brew, creates PRs with changes |
| **Kiota** | C#, Go, Java, PHP, Python, TypeScript | Strongly typed clients | Open-source from Microsoft |

**Kiota:** Open generator from Microsoft, targeting Microsoft Graph but applicable to any OpenAPI. Goal — strictly typed, safe client code.

- Repository: https://github.com/microsoft/kiota

---

## Recommendation for RST → Python SDK

Since no existing tool supports RST as input format, we recommend a **hybrid code-based approach**:

```
RST Documentation
        ↓
   MCP + RAG Layer
   - docutils/sphinx parsing
   - LLM extraction
   - Vector search
        ↓
   Custom API Model (dataclasses)
        ↓
   Code-Based Generator (Python)
   - Jinja2 templates
   - Direct code generation
        ↓
   Python SDK (otcextensions-compatible)
```

### Why Code-Based is Better for This Task

1. **Custom input parsing** — RST requires specialized parsing
2. **Flexible intermediate model** — not constrained by OpenAPI schema
3. **Python for Python** — generator in target language simplifies development
4. **MCP integration** — easy to integrate LLM for resolving ambiguities

### Technical Recommendations

1. **Use Jinja2 instead of Mustache** — Python-native, more powerful, better IDE support

2. **Borrow patterns from both tools:**
   - From OpenAPI Generator: template organization, supporting files
   - From AutoRest: pipeline stages, code model structure

3. **Create intermediate model** specific to RST structure, not based on OpenAPI