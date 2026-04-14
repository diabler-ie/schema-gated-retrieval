# Informed Retrieval for MCP Servers

A mechanism for MCP servers that return large responses. Instead of dumping everything into the LLM's context window, return a schema profile and let the LLM choose what to load.

#### Usage
```
This is a spec, not a library. Point your coding agent to this repo and
ask it to implement informed retrieval in your MCP server.
```

## The Problem

MCP tool returns are force-fed into the LLM's context. Whatever a tool returns goes directly into the conversation -- the LLM doesn't get to choose. A search returning 10,000 log entries means 500K tokens loaded whether the LLM wants them or not. The LLM has no agency over what enters its context window.

This is a fundamental issue with how MCP tools interact with agents. The context window is the scarce resource, and the LLM -- the thing best positioned to know what it needs -- has zero control over what fills it.

Naive mitigations don't solve this:
- **Truncation** cuts the response to fit, but the LLM still didn't choose what survived.
- **Static field projections** require the developer to predict what the agent will need, which varies by task.
- **Pagination** controls how many items, but each page still dumps everything into context.

## The Solution

Put a gate between the data and the context window. The cache isn't storage -- it's a mechanism that gives the LLM control over what enters its context.

1. **Fetch + Profile**: The tool fetches the full API response, stores it in a local cache, and returns only a lightweight schema profile (~1-3KB) describing the data's structure, field types, and sizes. This is what enters context -- metadata, not data.

2. **Decide**: The LLM sees the profile -- field names, types, average sizes, presence rates -- and makes an informed decision about what it actually needs. It can see that `attachments` averages 48KB per item and skip it entirely, or that `status` is 60 bytes and cheap to pull.

3. **Selective Query**: The LLM calls `query_cached` with specific fields, filters, and a limit. Only the requested slice enters context.

The key insight is that **the LLM -- not the tool -- controls what enters the context window**. The profile gives it enough information to make context-budget-aware decisions before committing to loading any data. An agent summarizing items needs different fields than one doing deep analysis, even when querying the same API endpoint.

## How It Works

### Phase 1: Schema Profile

When a list tool is called, instead of returning the full API response:

```python
# In your MCP tool
items = await fetch_from_api(...)          # Full API call
cache_id = cache.store(items)              # Store in memory
profile = cache.profile(cache_id)          # Generate schema
return json.dumps(profile)                 # Return ~3KB, not 3MB
```

The profile describes what's in the cache without including the actual data:

```json
{
  "cache_id": "items_abc12345",
  "item_count": 200,
  "cached_size_bytes": 4200000,
  "expires_in_seconds": 600,
  "schema": [
    {"field": "attachments", "type": "array",  "avg_size": 48200, "avg_length": 5, "presence": "100%"},
    {"field": "details",     "type": "object", "avg_size": 1800,  "presence": "100%"},
    {"field": "title",       "type": "string", "avg_size": 55,    "presence": "100%"},
    {"field": "status",      "type": "object", "avg_size": 60,    "presence": "100%"},
    {"field": "priority",    "type": "string", "avg_size": 16,    "presence": "85%"}
  ],
  "sample": {
    "id": "6d952dc4-...",
    "title": "Example Item",
    "status": {"name": "closed", "displayName": "Closed"},
    "attachments": "[5 items, ~48200 bytes avg total]",
    "details": "{object, ~1800 bytes}"
  }
}
```

Key design choices:
- **Schema sorted by `avg_size` descending** -- the agent immediately sees the heaviest fields and can avoid them.
- **Heavy fields replaced in sample** -- fields averaging >500 bytes are shown as size descriptions (e.g., `"[5 items, ~48200 bytes avg total]"`) so the agent understands the structure without the weight.
- **Presence percentage** -- tells the agent which fields are sparse (useful for filtering decisions).

### Phase 2: Selective Query

The agent reads the profile and calls a generic `query_cached` tool:

```
query_cached(
  cache_id="items_abc12345",
  fields="title,status,priority,created",
  filter='{"status": "Open"}',
  limit=50
)
```

This returns only the requested fields for matching items -- maybe 15KB instead of 4.2MB.

### Flow Diagram

```
Agent                          MCP Server                    Cache
  |                               |                            |
  |-- list_items() -------------->|                            |
  |                               |-- GET /api/items --------->|
  |                               |<-- 200 (4.2MB) -----------|
  |                               |-- cache.store() --------->|
  |                               |-- cache.profile()          |
  |<-- schema profile (3KB) -----|                            |
  |                               |                            |
  |  (agent reads schema,         |                            |
  |   decides what it needs)      |                            |
  |                               |                            |
  |-- query_cached(fields=...) -->|                            |
  |                               |-- cache.query() --------->|
  |                               |-- project + filter         |
  |<-- projected items (15KB) ---|                            |
```

## How Is This Different?

| Approach | How it handles large responses | LLM controls what enters context? |
|---|---|---|
| **Truncation / summarization** | Cut or compress the response to fit | No -- tool decides what survives |
| **Cursor-based pagination** | Return fewer items per page | No -- each page still dumps everything |
| **Static field projections** | Developer hardcodes which fields to return | No -- developer decides up front |
| **Server-side heuristics** | Server scores and selects columns | No -- server guesses what's relevant |
| **Proxy sandboxing** ([Context Mode](https://github.com/mksglu/context-mode)) | Intercept all tool output, store externally, retrieve via search | Partially -- retrieves via search, but without a data map |
| **Memory pointers** ([arXiv](https://arxiv.org/abs/2511.22729)) | Store externally, give LLM opaque references | Partially -- can access data, but doesn't know its shape or cost |
| **Reactive caching** ([mcp-cache](https://github.com/swapnilsurdi/mcp-cache)) | Cache + keyword/JSONPath search | Partially -- searches, but without a data map |
| **Informed retrieval** | Cache + schema profile with field types, sizes, and samples | **Yes -- LLM sees the data's shape and cost before loading any of it** |

Every other approach still force-feeds data into context -- they just argue about how much. The profile is the differentiator because it shifts control to the LLM. Instead of the tool deciding what the LLM sees, the LLM gets a structured map of the data -- field types, average sizes, presence rates -- and decides for itself what's worth the context budget. See [Prior Art](docs/prior-art.md) for detailed comparisons.

## Observed Results

In production use:

| Metric | Value |
|---|---|
| Raw API response (200 items) | 4.2 MB |
| Schema profile returned to agent | ~3 KB |
| Selective query (50 items, 4 fields) | ~15 KB |
| Total context consumed (both phases) | ~18 KB |
| End-to-end context reduction | **99.6%** |

The gate turns a 4.2MB context dump into an 18KB informed retrieval. The full dataset stays available in the cache -- the LLM can always request more fields or different filters without another API call. The LLM sampled before committing context budget, which is the constraint that actually matters.

## Implementation

For details on integrating this pattern into your MCP server:

- **[Cache Design](docs/cache-design.md)** -- cache module interface, profiling algorithm, memory management, and filter support
- **[Implementation Guide](docs/implementation.md)** -- step-by-step integration, tool schemas, server instructions, and conditional caching
- **[Prior Art](docs/prior-art.md)** -- detailed comparisons with existing approaches



##### If you implement this method and it works well for you, consider leaving a star on the repo! :)
