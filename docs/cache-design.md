# Cache Design

The cache module is the core of the pattern and should be **completely API-agnostic** -- it works with any `list[dict]` regardless of what API produced it.

## Core Operations

| Method | Purpose |
|---|---|
| `store(items, source_label, ttl)` | Cache a list of dicts, return a `cache_id` |
| `profile(cache_id)` | Generate schema profile from cached items |
| `query(cache_id, fields, filter, limit, offset)` | Retrieve projected/filtered items |
| `list_caches()` | Show all active cache entries |

## Schema Profiling

Profiling samples the first N items (e.g., 20) and for each field:
- Infers type (string, number, boolean, array, object, null)
- Measures average serialized size in bytes
- Calculates presence rate (what % of items have this field)
- For arrays: measures average length

This is enough for the agent to make informed decisions about which fields to request.

### Sample Generation

The profile includes a sample built from the first cached item. Fields with an average size over ~500 bytes are replaced with size descriptions rather than actual values:

- Arrays become `"[5 items, ~48200 bytes avg total]"`
- Objects become `"{object, ~1800 bytes}"`

This lets the agent see the *shape* of heavy fields without the weight, keeping the full profile under ~3KB.

## Memory Management

- **TTL-based expiry** -- entries expire after a configurable duration (e.g., 10 minutes). Expired entries are cleaned up on the next `store()` call.
- **Size-based eviction** -- when total cache exceeds a cap (e.g., 100MB), oldest entries are evicted first.
- **Defensive copy** -- `store()` copies the input list to prevent caller mutation from corrupting the cache.

## Expired Cache Handling

When the agent calls `query_cached` with an expired or invalid `cache_id`, the tool should return a clear error message:

```json
{
  "error": "cache_expired",
  "message": "Cache entry 'items_abc12345' has expired. Re-call the original list tool to refresh."
}
```

This tells the agent exactly what happened and how to recover -- re-call the list tool that originally populated the cache. No ambiguity, no silent failures.

## Filter Support

The `query` method supports filtering cached items:
- **Exact match**: `{"status": "active"}` -- simple equality check. For dict values, checks if the criteria matches any value in the dict.
- **Substring search**: `{"name": {"contains": "keyword"}}` -- case-insensitive substring match on string fields or serialized objects.

## `list_caches`

Returns a summary of all active cache entries:

```json
[
  {
    "cache_id": "items_abc12345",
    "source_label": "items:instance1",
    "item_count": 200,
    "cached_size_bytes": 4200000,
    "expires_in_seconds": 312
  }
]
```

This is useful when the agent needs to check whether data is already cached before making another API call, or when working across multiple cached datasets in a single conversation.

## Limitations and Considerations

- **Single-process, in-memory by default.** The cache lives in the MCP server process's memory. If the server restarts between Phase 1 and Phase 2, or runs behind a load balancer with multiple instances, the cache will be lost. Multi-instance deployments need a shared cache layer (e.g., Redis) implementing the same interface.
- **Sampling bias.** Profiling samples the first N items, not a random selection. If the API returns items sorted by date or status, the profile may not represent the full dataset. For APIs with non-uniform data, consider random sampling or increasing the sample size.
- **Query results can still be large.** The pattern prevents context overflow on list operations, but a `query_cached` call requesting many items or wide fields can still produce a large response. Agents should use reasonable `limit` values and paginate with `offset` when working through large result sets.
- **Read-only assumption.** The cache has no invalidation mechanism beyond TTL expiry. If your MCP server has write tools (e.g., `update_item`), the cache may become stale after a mutation. Consider invalidating relevant cache entries after write operations, or rely on the TTL to keep staleness bounded.
