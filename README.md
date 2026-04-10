# Profile-Guided Retrieval for MCP Servers

Fetch large responses to a local cache and return a data profile, letting the agent selectively query the cache for only what it needs.

## The Problem

MCP servers that wrap REST APIs often return massive JSON responses. A single API object can be tens of kilobytes when it contains nested objects, embedded arrays, or rich metadata. Listing hundreds of such objects can mean megabytes of data loaded directly into the LLM's context window -- far too much.

Naive mitigations like truncating the item list until it fits a size cap can result in returning only a handful of items, defeating the purpose of a list operation. Static field projections (hardcoding which fields to return) require the developer to predict what the agent will need, which varies by task.

## The Solution

Separate data retrieval from data consumption using the profile-guided approach:

1. **Fetch + Profile**: The tool fetches the full API response, stores it in a local cache, and returns only a lightweight schema profile (~1-3KB) describing the data's structure, field types, and sizes.

2. **Selective Query**: The agent reads the profile, decides which fields it actually needs for the current task, and queries the cache for just those fields.

The key insight is that **the agent -- not the developer -- chooses which fields to retrieve**, and it makes that choice dynamically based on the task at hand. An agent summarizing items needs different fields than one doing deep analysis, even when querying the same API endpoint.

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

| Approach | How it handles large responses | Agent chooses fields? |
|---|---|---|
| **Truncation / summarization** | Cut or compress the response to fit | No |
| **Cursor-based pagination** | Return fewer items per page | No |
| **Static field projections** | Developer hardcodes which fields to return | No |
| **Server-side heuristics** | Server scores and selects columns | No |
| **Reactive caching** (e.g., mcp-cache) | Cache + keyword/JSONPath search | Partially -- searches, but without a data map |
| **Profile-guided retrieval** | Cache + auto-generated profile with field types, sizes, and samples | **Yes -- informed by the profile** |

The profile is the differentiator. It gives the agent a structured map of the data -- field types, average sizes, presence rates -- so it can make context-budget-aware decisions about what to retrieve. See [Prior Art](docs/prior-art.md) for detailed comparisons.

## Observed Results

In production use:

| Metric | Value |
|---|---|
| Raw API response (200 items) | 4.2 MB |
| Schema profile returned to agent | ~3 KB |
| Selective query (50 items, 4 fields) | ~15 KB |
| Total context consumed (both phases) | ~18 KB |
| End-to-end context reduction | **99.6%** |

The pattern turns multi-megabyte API responses into kilobyte-scale context consumption while keeping the full dataset available in the cache -- the agent can always request more fields or different filters without another API call.

## Implementation

For details on integrating this pattern into your MCP server:

- **[Cache Design](docs/cache-design.md)** -- cache module interface, profiling algorithm, memory management, and filter support
- **[Implementation Guide](docs/implementation.md)** -- step-by-step integration, tool schemas, server instructions, and conditional caching
- **[Prior Art](docs/prior-art.md)** -- detailed comparisons with existing approaches
