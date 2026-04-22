# Geo-Agent Ecosystem Slide Deck — Design Spec

**Date:** 2025-04-15
**Format:** Reveal.js (loaded via CDN, single HTML file)
**Duration:** ~60 minutes (standard talk with Q&A)
**Audience:** Data science team — technical, familiar with geospatial data, SQL, APIs
**Narrative arc:** Outside-in — start with what users see, peel back architecture layers
**Key takeaway:** The architecture is elegant (STAC as truth, config over code, MCP as universal interface, virtuous training loop)
**Working directory:** /home/cboettig/Documents/github/boettiger-lab/slides/

## Slide Outline (~34 slides)

### Opening (2 slides)

1. **Title slide**
   - "geo-agent: An LLM-Powered Geospatial Analysis Ecosystem"
   - Speaker name, affiliation, date

2. **The demo**
   - Full-screen screenshot placeholder: map with layers + chat panel + active conversation
   - "What if exploring geospatial data was a conversation?"

### The User Experience (3 slides)

3. **Three files is all you need**
   - Show the three files side by side: `index.html`, `layers-input.json`, `system-prompt.md`
   - Emphasize: zero custom JavaScript

4. **Config-driven, not code-driven**
   - `layers-input.json` anatomy: collections, assets (simple string / object / versioned)
   - Visual: annotated JSON snippet

5. **Screenshot: the app in action**
   - Layer panel, chat panel, settings panel, tool approval flow
   - Multiple screenshot placeholders showing different interaction states

### The Data Pipeline — cng-datasets (7 slides)

6. **The problem**
   - Raw geospatial data: shapefiles, GeoJSON, GeoTIFF — messy, huge, not query-friendly
   - Need: cloud-native formats optimized for both visualization and analytics

7. **Cloud-native target formats**
   - Three outputs per dataset:
     - GeoParquet (DuckDB queries, spatial joins)
     - PMTiles (web map vector tiles, MapLibre GL JS)
     - H3 hex parquet (spatial aggregation, analytics)
   - Rasters: COG + H3 hex parquet

8. **K8s parallel job strategy**
   - The DAG: setup-bucket → convert → (hex ∥ pmtiles) → repartition
   - Indexed batch jobs: `completionMode: Indexed`, 50-pod parallelism
   - `cng-datasets` CLI generates YAML, `kubectl apply` runs it
   - "Your laptop generates YAML. The cluster does the work."

9. **A dataset's journey (concrete example)**
   - Raw shapefile → `cng-convert-to-parquet` → GeoParquet
   - GeoParquet → PMTiles (single pod)
   - GeoParquet → H3 hex (50 parallel pods, chunked) → repartition by h0
   - Final S3 layout diagram

10. **Hive partitioning and why it matters**
    - `s3://bucket/dataset/hex/h0={cell}/data_0.parquet`
    - DuckDB reads only the partitions it needs (partition pruning)
    - h0 = coarsest H3 cell = geographic region → queries skip irrelevant continents

11. **STAC as the registry**
    - Every dataset gets a STAC collection with:
      - Asset links (parquet, PMTiles, COG, hex)
      - `table:columns` (types + descriptions + categorical values)
      - `vector:layers` (for PMTiles/vector sources)
    - Hierarchical: root catalog → sub-catalogs → collections

12. **STAC metadata anatomy**
    - Concrete example: what a `table:columns` entry looks like
    - How schema descriptions flow into the LLM's system prompt
    - "The better the metadata, the smarter the agent"

### The MCP Data Server (5 slides)

13. **MCP: tool-use for LLMs**
    - Model Context Protocol — standardized way for LLMs to call external tools
    - Not tied to geo-agent: any MCP client can connect
    - Diagram: LLM ↔ MCP client ↔ MCP server ↔ DuckDB ↔ S3

14. **The four MCP tools**
    - `browse_stac_catalog` — discover available datasets
    - `get_stac_details` — get S3 paths + column schemas
    - `query` — execute DuckDB SQL against S3 parquet
    - `register_hex_tiles` — materialize H3 pyramids for MapLibre
    - Example: a SQL query with H3 joins and h0 partition pruning

15. **DuckDB on H3-indexed parquet**
    - Per-request `:memory:` connection (stateless)
    - Extensions: spatial, h3, httpfs
    - Key pattern: always include h0 in joins for partition pruning
    - Result: sub-second queries over billions of hex cells

16. **One server, many clients**
    - Diagram showing the same MCP server consumed from:
      - geo-agent (browser)
      - Jupyter notebooks (jupyter-geoagent)
      - VSCode (mcp.json config)
      - Claude Desktop
      - LangChain agent
      - Python scripts (streamablehttp_client)

17. **The GPU experiment**
    - mcp-gpu-data-server: RAPIDS cuDF + kvikio S3 reads
    - kvikio: 6.25 Gbps parallel HTTP range requests (vs 0.97 Gbps Polars)
    - Benchmark table: GPU vs CPU across 5 query types
    - Punchline: DuckDB wins 2-4x because S3 bandwidth, not compute, is the bottleneck
    - "We built a faster pipe but the faucet is the same size"

### The Agent Architecture — geo-agent (6 slides)

18. **Architecture overview**
    - Module diagram: main → DatasetCatalog → MapManager → ToolRegistry → Agent → ChatUI
    - Plus: MCPClient, layout-manager
    - Bootstrap sequence: config → STAC → map → MCP → tools → agent → UI

19. **System prompt construction**
    - How STAC metadata gets injected:
      - `catalog.generatePromptCatalog()` → lists all datasets, columns, parquet paths
      - MCP `geospatial-analyst` prompt adds persona + query rules
      - App-provided `system-prompt.md` adds personality
    - "The agent knows what columns exist because STAC told it"

20. **Two tool tracks**
    - Local tools (10): map control, navigation, dataset knowledge
      - Instant, auto-approved, run in browser
    - Remote tools (MCP): SQL queries via DuckDB
      - Require user approval (unless auto_approve)
      - Execute via HTTP to MCP server

21. **The agentic loop**
    - Diagram: user message → LLM → tool_calls → execute → append results → LLM → ... → final response
    - Max 20 iterations, last 12 messages in context
    - Tool results capped at 4K chars

22. **Tool proposal UX**
    - What the user sees when the agent wants to run SQL:
      - Model's reasoning text
      - Collapsible details block with SQL
      - [Approve] / [Cancel] buttons
    - Screenshot placeholder of the approval flow

23. **Config-driven everything**
    - No hardcoded S3 paths — agent calls `get_dataset_details()` first
    - No hardcoded styles — defaults from `layers-input.json`, overridable by agent
    - No hardcoded filters — MapLibre expressions applied dynamically
    - Layers-input supports versioned assets (dropdown switching)

### The LLM Proxy + Training Loop (6 slides)

24. **open-llm-proxy: more than a proxy**
    - OpenAI-compatible gateway: `/v1/chat/completions`
    - Multi-provider routing: NRP (primary), OpenRouter, Nimbus
    - Client auth: PROXY_KEY in, provider API keys hidden
    - CORS handling for browser-based apps

25. **Structured logging**
    - Every request/response pair logged to S3 as JSONL
    - Request: user_question, message_count, tool_results_this_turn, model, origin
    - Response: latency_ms, tokens, tool_calls (with SQL), content_preview
    - S3 layout: `logs-<ns>/YYYY-MM-DD/HH-MM-SS-<pid>.jsonl`

26. **A log entry up close**
    - Concrete example: one request + response pair
    - Show the fields that matter for training analysis
    - "Every agentic turn is observable"

27. **The training workflow**
    - Query logs with DuckDB (it's DuckDB all the way down)
    - Reconstruct sessions: group by user_question + timestamp
    - Classify failures: STAC metadata issue? MCP tool issue? Model behavior?

28. **Before/after: the feedback loop in action**
    - Example: agent guesses wrong column name → logs show repeated failures
    - Fix: add column descriptions to STAC `table:columns`
    - Result: agent uses correct column names
    - "Logs → diagnosis → metadata fix → better agent"

29. **The virtuous cycle**
    - Diagram: geo-agent → proxy logs → training analysis → improve {STAC metadata, MCP tool descriptions, system prompts} → better geo-agent
    - This is continuous, not one-shot

### Deployment & Reuse (3 slides)

30. **CDN architecture**
    - geo-agent library served via jsDelivr (`@main` or pinned tag)
    - Downstream apps: static HTML that imports from CDN
    - Merge to main → cache purge → all apps updated

31. **Deployment patterns**
    - GitHub Pages: static HTML, user provides API key in browser
    - HF Spaces: mount config.json as secret
    - Kubernetes: init container injects config from ConfigMap
    - All patterns: zero JS to write

32. **Spin up a new app in 5 minutes**
    - geo-agent-template repo
    - Fork → edit 3 files → deploy
    - Each app: different datasets, branding, system prompt, same core

### Close (2 slides)

33. **Design principles**
    - STAC as single source of truth
    - Config over code (zero JS for app builders)
    - MCP as universal data interface (not just for chatbots)
    - The virtuous training loop (logs → analysis → improvement)
    - H3 + hive partitioning = fast queries at any scale

34. **Links & questions**
    - GitHub repos: geo-agent, mcp-data-server, data-workflows, open-llm-proxy
    - Live demos
    - Documentation sites

## Technical Notes

- **Framework:** Reveal.js 5.x loaded from CDN (`<script src="https://cdn.jsdelivr.net/npm/reveal.js@5/..."`)
- **Single file:** One `index.html` with all slides inline
- **Code highlighting:** Reveal.js highlight plugin for SQL, JSON, JS snippets
- **Diagrams:** Inline SVG or CSS-based (no external diagram tool dependency)
- **Screenshots:** Placeholder `<div>` blocks with dashed borders and labels; user swaps in real images later
- **Theme:** Dark theme (good for projectors and code readability)
- **No build step:** Open the HTML file directly or `python -m http.server`

## Screenshot Placeholders Needed

The following slides need real screenshots (user will provide):

- Slide 2: Full app — map + chat + active conversation
- Slide 5: Layer panel, chat panel, settings panel (multiple states)
- Slide 22: Tool proposal / approval flow in chat
- Optional: live demo recordings / GIFs for the opening
