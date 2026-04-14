# Prior Art

Existing approaches to handling large tool responses in MCP servers and agent frameworks. The core problem — MCP tool returns are force-fed into the LLM's context with no mechanism for the LLM to control what it receives — is [widely recognized](https://github.com/modelcontextprotocol/modelcontextprotocol/discussions/2211) and actively discussed.

## Proxy-Based Sandboxing

- **[Context Mode](https://github.com/mksglu/context-mode)** — An MCP server that intercepts tool output via hooks and redirects it to files and SQLite, keeping raw data out of context entirely. Claims 98% reduction. The LLM retrieves stored data via BM25 search and indexing tools. The key difference: Context Mode sandboxes *everything* blindly — the LLM doesn't get a profile of what was captured, so it searches reactively rather than making informed, upfront decisions about what to load. It also operates as a proxy layer across all tools rather than a pattern integrated into individual MCP servers.

- **[mcp-cache](https://github.com/swapnilsurdi/mcp-cache)** — A transparent MCP proxy that detects large responses, caches them locally, and exposes `query_response`/`get_chunk` tools for keyword search, JSONPath, and regex. Similar reactive retrieval — the agent queries cached data iteratively, but without a schema profile step.

## Cache-Then-Query Approaches

- **[Data Query MCP Server](https://medium.com/@alieladi/core-components-of-a-data-query-analysis-mcp-server-ec2cd6cde34d)** — The closest full-pattern match. Exposes schema metadata as a resource, auto-caches large query results server-side (returning a preview + `cache_key`), and lets the agent run processing on the cached data. Shares the cache-then-selectively-query flow. The main difference is that it targets data/analytics workloads and uses a processing tool (pandas/numpy) rather than field projection from a schema profile with size metadata.

- **[Universal JSON Agent MCP](https://dev.to/gautamvhavle/my-json-was-too-big-for-my-ai-so-i-built-an-mcp-server-to-fix-it-43m2)** — Loads local JSON files into server memory and exposes 26 composable tools (structure inspection, JSONPath queries, filtering, aggregation) so the agent explores metadata before requesting data. Shares the core insight of "don't dump everything into context, let the agent decide what it needs." Differs in targeting local files, using many small tools rather than a single generic query tool, and lacking size metadata for context-budget-aware field selection.

## Academic

- **[Solving Context Window Overflow in AI Agents](https://arxiv.org/abs/2511.22729)** (arXiv, Nov 2025) — Proposes "memory pointers" so LLMs interact with references to external data rather than raw data. Shares the core idea of keeping data outside context and using identifiers (like `cache_id`) to access it. The difference is that memory pointers are opaque — the LLM gets a reference but no profile of what it points to, so it can't make field-level or size-aware retrieval decisions.

## Protocol-Level Proposals

- **[MCP spec Discussion #2211](https://github.com/modelcontextprotocol/modelcontextprotocol/discussions/2211)** — Proposes protocol-level response size limits. The client would enforce a maximum allowed tool response size and reject oversized responses with a structured error. A blunt instrument — it prevents overflow but doesn't give the LLM a way to access the data it still needs.

- **[MCP spec Discussion #1780](https://github.com/modelcontextprotocol/modelcontextprotocol/discussions/1780)** — Proposes giving MCP tools two output channels: a metadata/instruction channel for the LLM's context, and a data channel for structured data meant to be read programmatically. Conceptually the closest to the gate idea — separating what the LLM sees from what's available. Still a proposal, not an implementation, and doesn't specify how the LLM decides what to pull from the data channel.

- **[MCP spec pagination](https://modelcontextprotocol.io/specification/2025-03-26/server/utilities/pagination)** — The official spec supports cursor-based pagination for list operations, but this addresses result count, not the per-item context cost.

## Server-Side Approaches

- **[Axiom's wide-schema design](https://axiom.co/blog/designing-mcp-servers-for-wide-events)** — A server-side approach: heuristic column scoring based on importance, fill rate, and uniqueness, plus CSV over JSON (29% fewer tokens) and global cell budgets. The server intelligently selects fields rather than letting the agent choose.

## Agent Frameworks

- **LangChain, LlamaIndex, CrewAI** — Handle large tool outputs via truncation, summarization, or streaming. None implement schema-profiled caching with agent-driven field projection.

## What's Different About Schema-Gated Retrieval

The fundamental issue is that MCP tool returns are force-fed into the LLM's context — the LLM has no agency over what enters its context window. Existing solutions either sandbox everything (Context Mode), reject oversized responses (size limits), give the LLM opaque pointers (memory pointers), or have the server guess what's relevant (Axiom). None give the LLM a structured map of the data so it can make its own context-budget-aware decisions.

Schema-gated retrieval's schema profile is the differentiator: field types, average sizes, presence rates, and a representative sample — all in ~3KB. The LLM sees the shape and cost of the data before committing any of it to context. It's the difference between searching blind and choosing informed.
