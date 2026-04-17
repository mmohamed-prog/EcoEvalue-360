# Review: datagouv/datagouv-mcp

**Repository**: https://github.com/datagouv/datagouv-mcp  
**Reviewed**: 2026-04-17  
**Version reviewed**: 0.2.23

---

## Summary

`datagouv-mcp` is a well-constructed MCP (Model Context Protocol) server that exposes France's open data platform (data.gouv.fr) to AI chatbots. The codebase is clean, consistently structured, and production-hardened. It is a solid reference implementation for MCP servers that wrap external APIs.

---

## Architecture

The project follows a clear separation of concerns:

```
main.py                  ← server bootstrap, security, health endpoint
tools/                   ← one file per MCP tool (10 tools)
helpers/                 ← API clients, env config, logging, monitoring
tests/                   ← unit + integration tests per module
```

Each tool lives in its own file and follows a consistent `register_<tool>_tool(mcp: FastMCP)` factory pattern. This makes tools easy to discover, add, and test in isolation. The helpers layer cleanly abstracts five external APIs (datagouv, tabular, metrics, crawler, site) behind typed async functions.

---

## Strengths

### 1. MCP Tool Annotations
All tools are decorated with `READ_ONLY_EXTERNAL_API_TOOL` (`ToolAnnotations(readOnlyHint=True, idempotentHint=True, ...)`), which is excellent practice. This lets MCP clients reason about safety and retry behavior without inspecting tool logic.

### 2. Security Hardening
`main.py` configures DNS rebinding protection and strict origin/host validation, allowing only `mcp.data.gouv.fr`, `mcp.preprod.data.gouv.fr`, and localhost. This is a production-grade concern often overlooked in MCP server implementations.

### 3. Graceful Search Fallback
`search_datasets.py` implements a two-pass search strategy: it strips French stop words first (addressing the API's strict AND logic), then falls back to the original query. This meaningfully improves result quality without any API-side change.

### 4. Layered Error Handling in Tabular Client
`tabular_api_client.py` distinguishes retriable server errors (5xx, 408, 429) from client errors, and provides column-specific hints when a sort/filter references a non-existent column. This reduces LLM hallucination loops caused by opaque API errors.

### 5. Session Reuse Pattern
All API client functions accept an optional `httpx.AsyncClient` parameter, allowing callers to share a session across requests. Functions create and close their own sessions when none is provided, preventing connection leaks.

### 6. Observability
Matomo analytics tracks tool usage, Sentry captures exceptions, and structured logging decorates each tool call via `@log_tool`. The Matomo integration correctly skips tracking when the base URL env var is unset, making local dev friction-free.

### 7. Test Coverage
11 test files covering API clients, tool logging, health endpoint, Matomo middleware, and error cases. A dedicated `test_stress.py` (excluded from default runs) enables load testing against a live instance.

---

## Issues and Suggestions

### 1. `get_metrics.py` — Verbose with Duplication (Medium)
The `get_metrics` function is ~120 lines handling dataset and resource metrics through parallel if-branches that duplicate structure. The dataset and resource metric-fetching blocks share identical patterns (fetch metadata → catch → append; fetch metrics → catch → format table). Extracting a helper reduces this to ~50 lines with no behavior change.

```python
# Current: duplicated structure ~60 lines each
if dataset_id:
    # fetch metadata + metrics, build table
if resource_id:
    # same pattern

# Suggested: extract helper
async def _fetch_and_format_metrics(entity_type, entity_id, limit, ...): ...
```

### 2. Python `>=3.13` Requirement (Low)
The `pyproject.toml` requires `python>=3.13,<3.15`. Python 3.13 was released October 2024, but many deployment environments (GitHub Actions runners, OS packages) still default to 3.12. This will break `pip install` silently in those environments. The Dockerfile correctly pins `python3.14`, but teams deploying without Docker may hit this unexpectedly. Consider documenting this prominently or providing a 3.12-compatible fallback.

### 3. `get_metrics.py` — Demo Environment Check at Runtime (Low)
The demo-environment guard is implemented inside the tool function body via `os.getenv("DATAGOUV_API_ENV")`. This means the tool is always registered and advertised to clients, but fails at call time with a text error string rather than a proper MCP error. A cleaner approach is to skip registering the metrics tool entirely when `DATAGOUV_API_ENV=demo`, or to raise an `McpError` instead of returning an error string, so clients can distinguish tool errors from tool results.

### 4. Broad Exception Catch with `# noqa: BLE001` (Low)
The codebase uses `except Exception as e: # noqa: BLE001` extensively to suppress the "blind exception" lint warning. While the intent (returning user-facing strings instead of raising) is correct for MCP tools, suppressing the lint rule globally masks cases where unexpected exception types (e.g., `TypeError` from malformed API responses) could instead indicate a programming error. Prefer catching specific exception types where possible, reserving the broad catch only for the outermost handler.

### 5. No Input Sanitization on IDs (Low)
Dataset/resource IDs passed to API clients are lightly sanitized (`.strip()`) but not validated against an expected format (e.g., UUID or slug pattern). Malformed IDs will produce 404s from the upstream API, which are handled, but a short regex pre-check would provide a faster, clearer user-facing error and would prevent any potential path-traversal in URL construction.

### 6. `uv.lock` Committed without `--frozen` Documentation (Informational)
The Dockerfile runs `uv sync --frozen`, which is correct. However, the README doesn't mention that contributors must regenerate the lock file with `uv lock` (not `pip install`) when changing dependencies. A one-line note in the contributing section would prevent confusion.

---

## Dependency Notes

| Package | Pinned Range | Notes |
|---------|-------------|-------|
| `mcp` | `>=1.25.0,<2` | Good — avoids major-version breakage |
| `httpx` | `>=0.28.0` | Could benefit from an upper bound |
| `sentry-sdk` | `>=2.54.0` | No upper bound; MAJOR updates could break |
| `ruff` | `>=0.14.0` | Linting version unpinned; may cause CI churn |

---

## Relevance to EcoEvalue-360

EcoEvalue-360 currently hardcodes all regulatory data (28 rules, CEE catalog, 13 funding programs). The datagouv-mcp server provides access to data.gouv.fr datasets that include:

- **Décret Tertiaire** tracking data (OPERAT platform exports)
- **DPE (Diagnostic de Performance Énergétique)** database (ADEME)
- **CEE (Certificats d'Économie d'Énergie)** program references
- **ADEME open datasets** on energy consumption by sector

Integrating the datagouv-mcp public endpoint (`https://mcp.data.gouv.fr/mcp`) into an AI assistant layer on top of EcoEvalue-360 would allow dynamic enrichment of regulatory thresholds and funding amounts rather than relying on hardcoded values that require manual updates. The server requires no authentication and supports streamable HTTP, making it straightforward to call from a server-side enrichment layer.

---

## Verdict

**Recommended for integration.** The codebase is production-quality, well-tested, and actively maintained (0.2.23 as of April 2026). The issues identified are minor and do not affect correctness or security. The public instance at `https://mcp.data.gouv.fr/mcp` makes adoption zero-friction.
