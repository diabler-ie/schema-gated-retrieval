# Implementation Guide

How to integrate informed retrieval into an MCP server.

## Step 1: Add the Cache Module

The cache module is API-agnostic. Copy it into your project or extract it to a shared package. It needs no configuration for specific entity types or APIs. See [Cache Design](cache-design.md) for the full interface and profiling algorithm.

## Step 2: Register the `query_cached` Tool

This tool is API-agnostic -- one registration covers all cached data regardless of source.

### Tool Schema

```json
{
  "name": "query_cached",
  "description": "Retrieve specific fields from cached data. Use this after receiving a schema profile from a list tool. Pass the cache_id from the profile and specify which fields you need.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "cache_id": {
        "type": "string",
        "description": "The cache_id returned in the schema profile"
      },
      "fields": {
        "type": "string",
        "description": "Comma-separated field names to retrieve (e.g., 'title,status,priority')"
      },
      "filter": {
        "type": "string",
        "description": "Optional JSON filter object. Exact match: {\"status\": \"active\"}. Substring: {\"name\": {\"contains\": \"keyword\"}}"
      },
      "limit": {
        "type": "integer",
        "description": "Maximum number of items to return (default: 50)"
      },
      "offset": {
        "type": "integer",
        "description": "Number of items to skip for pagination (default: 0)"
      }
    },
    "required": ["cache_id", "fields"]
  }
}
```

## Step 3: Register the `list_caches` Tool

```json
{
  "name": "list_caches",
  "description": "Show all active cache entries with their cache_id, source, item count, size, and remaining TTL. Use this to check if data is already cached before re-fetching.",
  "inputSchema": {
    "type": "object",
    "properties": {}
  }
}
```

## Step 4: Update List Tools

Change each list tool from returning raw results to caching + profiling:

```python
# Before
async def list_items(instance: str) -> str:
    items = await fetch_items_from_api(instance)
    return json.dumps({"items": items})

# After
async def list_items(instance: str) -> str:
    items = await fetch_items_from_api(instance)
    cache_id = cache.store(items, source_label=f"items:{instance}")
    return json.dumps(cache.profile(cache_id))
```

## Step 5: Update Server Instructions

Add instructions to your MCP server so the agent knows how to use the pattern:

```
List tools return a schema profile and cache_id, not raw items.
Use query_cached(cache_id, fields, filter) to selectively retrieve
only the fields you need. This keeps context usage minimal.
Use list_caches to see active cache entries before re-fetching data.
If query_cached returns a cache_expired error, re-call the original
list tool to refresh the cache.
```

## Step 6 (Optional): Conditional Caching for Variable-Size Results

List tools always cache because they inherently return collections. But search or query tools can return anything from a few rows to thousands of records. For these, use **conditional caching** based on response size:

```python
result_json = json.dumps(result)
items = result.get("results", [])
if len(result_json) > SIZE_THRESHOLD and items:
    # Large result -- cache and return profile
    cache_id = cache.store(items, source_label="search:instance1")
    profile = cache.profile(cache_id)
    return json.dumps({
        "total": len(items),
        "cached": True,
        "cache_id": cache_id,
        "schema": profile["schema"],
    })
else:
    # Small result -- return inline as usual
    return result_json
```

A threshold of ~50KB works well in practice. Small tabular results fall below this and are returned inline. Large result sets exceed it and get cached automatically.

Note: The key to check for (`"results"` above) will depend on your API's response structure. The cache module itself is API-agnostic, but extracting the item list from the response requires knowing the response format.
