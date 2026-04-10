# Prior Art

Existing approaches to handling large responses in MCP servers and agent frameworks.

## Cache-Then-Query Approaches

- **[Data Query MCP Server](https://medium.com/@alieladi/core-components-of-a-data-query-analysis-mcp-server-ec2cd6cde34d)** — The closest full-pattern match. Exposes schema metadata as a resource, auto-caches large query results server-side (returning a preview + `cache_key`), and lets the agent run processing on the cached data. Shares the cache-then-selectively-query flow. The main difference is that it targets data/analytics workloads and uses a processing tool (pandas/numpy) rather than field projection from a schema profile with size metadata.

- **[mcp-cache](https://github.com/swapnilsurdi/mcp-cache)** — A transparent MCP proxy that detects large responses, caches them locally, and exposes `query_response`/`get_chunk` tools for keyword search, JSONPath, and regex. The agent queries cached data iteratively, but without a schema profile step — it searches reactively rather than choosing fields from a data map.

- **[Universal JSON Agent MCP](https://dev.to/gautamvhavle/my-json-was-too-big-for-my-ai-so-i-built-an-mcp-server-to-fix-it-43m2)** — Loads local JSON files into server memory and exposes 26 composable tools (structure inspection, JSONPath queries, filtering, aggregation) so the agent explores metadata before requesting data. Shares the core insight of "don't dump everything into context, let the agent decide what it needs." Differs in targeting local files, using many small tools rather than a single generic query tool, and lacking size metadata for context-budget-aware field selection.

## Server-Side Approaches

- **[Axiom's wide-schema design](https://axiom.co/blog/designing-mcp-servers-for-wide-events)** — A server-side approach: heuristic column scoring based on importance, fill rate, and uniqueness, plus CSV over JSON (29% fewer tokens) and global cell budgets. Interesting alternative philosophy where the *server* intelligently selects fields rather than letting the agent choose.

## Protocol-Level

- **[MCP spec pagination](https://modelcontextprotocol.io/specification/2025-03-26/server/utilities/pagination)** — The official spec supports cursor-based pagination for list operations, but this addresses result count, not the context cost of individual items.

## Agent Frameworks

- **LangChain, LlamaIndex, CrewAI** — Handle large tool outputs via truncation, summarization, or streaming. None implement schema-profiled caching with agent-driven field projection.

## What's Different About Profile-Guided Retrieval

What distinguishes profile-guided retrieval is the schema profile step: giving the agent a structured map of the data (field types, sizes, presence rates) so it can make informed, task-specific decisions about what to retrieve — rather than searching blindly, relying on developer-chosen projections, or using server-side heuristics.
