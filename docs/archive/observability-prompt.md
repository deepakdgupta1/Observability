# Role: Senior AI Observability Architect
# Task: Research and Design of a Comprehensive Analytics & Telemetry System for AI Agents and LLMs

Goal: Perform deep research and work with me to define the Functional Requirement Specifications (FRS) for a system that provides granular insights into AI Agent behavior, LLM performance, cost, and efficiency.

---

## Part 1: Operational Context & Hard Constraints

Before any design work, internalize these non-negotiable constraints. Every architectural decision must be validated against them.

### 1.1 Deployment Environment
- **Target**: Single developer's local macOS workstation. This is NOT a cloud or team deployment.
- **Hardware**: MacBook with **16 GB total RAM**. The entire observability stack (proxy, storage, UI, background workers) must operate within a **≤4 GB RAM** budget.
- **Cost**: **Zero incremental cost**. No pay-per-use API keys. All LLM-dependent features (e.g., evaluation pipelines) must operate via existing OAuth-authenticated provider subscriptions.

### 1.2 Agent Inventory (7 Agents)

| Agent | Primary Provider | Auth Mechanism | Runtime |
|:---|:---|:---|:---|
| Claude Code | Anthropic | OAuth | Node.js |
| Antigravity | Google (Gemini) | OAuth | Node.js |
| Codex | OpenAI | OAuth / API Key | Rust |
| Gemini CLI | Google | OAuth | Node.js |
| AMP Code CLI | AMP (Sourcegraph) | OAuth | Node.js |
| Kilo Code CLI | Kilo Code | OAuth / API Key | Node.js (VS Code) |
| Hermes | Nous Research | API Key | Python |

### 1.3 LLM Provider Scope

**MVP Scope (3 Providers)**:
- **Anthropic** (`api.anthropic.com`): Token types — input, output, cache_read, cache_creation.
- **OpenAI** (`api.openai.com`): Token types — input, output, reasoning, cached_input.
- **Google Gemini** (`generativelanguage.googleapis.com`): Token types — input, output, thinking, cached.

**Deferred**: AMP, Kilo Code, Nous Research provider API schemas to be reverse-engineered in a later phase.

### 1.4 Scale
- Current: ~20 developer sessions/day.
- Target: Hundreds of automated agent runs/day.
- Tools: 70+ tools across all agents.
- Data Retention: 3 months.

### 1.5 Priority Matrix (Ranked)

| Priority | Domain | Goal |
|:---|:---|:---|
| **P0** | Quality Assurance | Detect hallucinations, bad tool calls, parameter errors |
| **P1** | Efficiency Tuning | Minimize context window utilization, minimize LLM request payload size per turn |
| **P2** | Cost Optimization | Track token spend, maximize cache hit rate |
| **P3** | Debugging | Trace failures, inspect decision paths (lowest priority) |

---

## Part 2: Collaborative Requirement Gathering Process

Before proposing any technical architecture or toolset, you must:

1.  **Interview Me**: Ask targeted questions to understand the scale of operations, the complexity of the agentic loops, and the specific outcomes desired from this observability. Cover:
    - Agent inventory and provider mix.
    - Authentication mechanisms (OAuth vs. API Key).
    - Hardware and cost constraints.
    - Priority ranking of observability domains.
    - Data retention and storage preferences.
    - Existing monitoring infrastructure (if any).
    - Sub-agent and multi-agent orchestration patterns.
    - Skill and Knowledge Item (KI) discovery patterns.

2.  **Research Current State-of-the-Art**: Perform thorough research into current observability paradigms for agentic workflows, focusing on:
    - **OpenTelemetry GenAI Semantic Conventions** (v1.40.0+): Agent Spans, Tool Spans, MCP conventions.
    - **White-Box telemetry** and tracing standards for non-deterministic, multi-step agentic workflows.
    - **Open-source observability platforms** (Langfuse, Arize Phoenix, OpenLLMetry, Opik, OpenObserve) — compare features, resource requirements, and suitability for solo-developer self-hosted deployments.
    - **Proxy-based interception** for agents using provider-managed OAuth authentication.

3.  **Define the FRS**: Work with me to draft a detailed Functional Requirement Specification (FRS) document that includes data schemas, trace hierarchies, success metrics, and an architectural review for resiliency.

---

## Part 3: Behavioral & Performance Research Areas

Your research and subsequent design must address the following key domains:

### 1. Interception Architecture & OAuth Preservation

The most critical architectural challenge: all agents use **provider-managed OAuth 2.0 flows**, not user-supplied API keys. The system must intercept traffic transparently.

**Research and Design Requirements**:
- Evaluate **mitmproxy** (or equivalent) as a transparent HTTPS forward proxy. The proxy must observe and log traffic without interfering with OAuth token flows.
- Confirm that all agents in the inventory support `HTTPS_PROXY` environment variables. Research each agent's source code (all are open-source) to verify proxy compatibility.
- Design **agent-specific CA certificate trust** (e.g., `NODE_EXTRA_CA_CERTS`, `REQUESTS_CA_BUNDLE`, `SSL_CERT_FILE`) — **NO system-wide CA trust installation**.
- Evaluate and explicitly reject **LiteLLM** if its `master_key` authentication middleware conflicts with transparent OAuth JWT passthrough.
- Handle **Server-Sent Events (SSE)** streaming responses: buffer and reassemble SSE streams without blocking the data flow to the agent.
- Handle **WebSocket traffic** (e.g., Codex CLI uses `WSS_PROXY`).
- Design for **certificate pinning risk**: identify a fallback strategy (e.g., SDK-level instrumentation) for any agent that pins certificates.
- Avoid using port `8080` (common dev server conflict). Use a dedicated high port (e.g., `18080` or `3128`).

### 2. Agent & Sub-Agent Identification

With 7 agents routing through a single proxy, the system must reliably identify which agent made each request.

**Design Requirements**:
- **Primary method**: Custom `X-Agent-Name` header injection via wrapper launch scripts. Provide per-agent configuration guidance.
- **Sub-agent tracing**: When a parent agent spawns child sub-agents (e.g., Antigravity's `browser_subagent`), the system must support:
    - Dotted naming convention: `antigravity.browser_subagent`.
    - Session correlation via inherited `X-Session-ID`.
    - Process tree inspection fallback (`pgrep -P <parent_pid>`) but **only in a background worker** — never inline in the proxy's critical path.
- **Fallback**: `User-Agent` header parsing (e.g., Claude Code sends `anthropic-sdk/...`).
- Consider encoding Session ID into the proxy URL itself (`http://agentname:sessionid@127.0.0.1:18080`) to survive environment variable scrubbing by sandbox environments.

### 3. Trace Hierarchy & Context Composition

How should the system capture and categorize the "Context Window" composition?

**Design Requirements**:
- **Four-level span hierarchy**: Session → Turn → LLM Request → Tool Execution/Skill Discovery.
- **Granular context decomposition** of each LLM request into:
    - System prompts (and token count).
    - Tool definitions (count and token count — alert when >30% of context window).
    - Few-shot examples.
    - Conversation history (turn count and token count).
    - Dynamic agent state (KIs, rules, injected context).
    - Current user message.
- Track **both aggregate token counts AND full content** for each category. Store full content in a JSONL archive for offline analysis and version diffing.
- **Content versioning**: SHA-256 hash system prompts and tool definitions across requests. Store new versions and compute diffs when content changes. Correlate prompt version changes with quality/efficiency metric changes.
- **Skill Discovery tracking**: Detect when agents read SKILL.md files or reference Knowledge Items. Track candidates available, candidates evaluated, skill selected, discovery time, and relevance (via offline eval).
- Turn-level vs. Session-level aggregation requirements.

### 4. Advanced Token & Cost Economics

Design the logic for tracking multi-tiered pricing and token types.

**Design Requirements**:
- **Multi-type token tracking per provider**:
    - Anthropic: input, output, cache_read, cache_creation (no separate thinking tokens).
    - OpenAI: input, output, reasoning_tokens, cached_tokens.
    - Google Gemini: input, output, thinking_tokens, cached_tokens.
- **Provider-specific response parsing**: Define exact field mappings for each provider's API response schema into a unified `llm.*` attribute namespace (not `gen_ai.*`).
- **YAML-based pricing configuration** (`pricing.yaml`): Version-controlled, updated monthly. Include alert for unknown models with no pricing entry.
- **Dual tokenizer strategy for per-component estimates**: Since providers report only aggregate input token counts (not per-component), use two local tokenizers with a reconciliation algorithm:
    - Use purely local tokenizers only (no remote API calls like Gemini's `count_tokens` endpoint — that would add latency in the critical path).
    - If both tokenizer counts are within 5% of each other, use the average.
    - If they diverge (smaller < 95% of larger), use the larger value (conservative estimate).
- **Cost attribution dimensions**: Per-session, per-turn, per-agent, per-provider, per-model.
- Track **cache hit rate**, **cache savings (USD)**, **thinking/reasoning token overhead (%)**, **input/output token ratio**.

### 5. Agentic Behavior & Tool Analytics

How do we monitor the "Agentic" reasoning process?

**Design Requirements**:
- **Tool Use Quality**: Track accuracy, parameter validation, execution success rate, error recovery rate, and tool fallback/retry rate.
- **Tool Discovery**: Track whether the correct tool was selected on the first attempt, number of candidates considered, and time-to-first-tool-invocation.
- **Tool Definition Budget**: Alert when tool definitions exceed a configurable percentage of the context window (default: 30%).
- **Loop & Redundancy Detection** (future roadmap — not MVP): Action fingerprinting (hash tool name + normalized params). Detect when the same action repeats in a sliding window.
- **Decision Path Visualization** (future roadmap): Trace DAG rendering of logic branches taken during complex tasks.

### 6. Quality, Evaluation & Efficiency

**Design Requirements**:
- **Offline Evaluation Pipeline** (not real-time — too expensive):
    - **Zero-cost constraint**: The evaluation LLM judge must use the **same OAuth-authenticated provider subscriptions** that the agents use. Route judge calls through the same proxy. Tag them with `X-Agent-Name: eval-pipeline` to exclude from agent metrics.
    - Use the cheapest/fastest model available through the active subscription (e.g., Claude Haiku, Gemini Flash).
    - Define sampling rates and evaluation rubrics for: Trajectory Relevancy, Faithfulness, Output Completeness, Hallucination Detection, Tool Discovery Accuracy, and Token-to-Insight Ratio.
    - Handle laptop sleep/shutdown: the scheduler must be resilient (e.g., "run if 24 hours have elapsed since last eval"), not dependent on fixed cron windows.
    - Maintain state of which JSONL lines have been evaluated to prevent re-evaluation after interruptions.
- **Efficiency Metrics**: Context Window Utilization, Context Bloat Factor, Payload Size per Turn (p50/p95), Turns per Task, Context Compression Ratio.
- **Latency Profiling**: Decompose latency into Time-to-First-Token (TTFT), total request duration, tool execution time, and network overhead.
- **Alerts & Thresholds**: Tool definition budget exceeded (>30%), context window critical (>80%), cost anomaly (>3× rolling average), high tool failure rate (>20%).

---

## Part 4: Architectural Resiliency Requirements

These are critical architectural concerns that the FRS must explicitly address. Failure to handle them will result in a system that degrades the very agents it's trying to observe.

### 1. Proxy Critical Path Performance
The interception proxy sits in the critical path of every LLM request. **It must be "dumb and fast."**
- **Async Processing Architecture**: The proxy must NOT perform tokenization, content hashing, cost calculation, or OTel span emission synchronously in the request/response path. It should only capture the raw payload and drop it into a local async queue (in-memory queue or on-disk staging).
- A **background worker process** picks up the payloads and performs all heavy computation (dual tokenizer, content hashing, Phoenix export, JSONL writing) asynchronously.
- **No Remote API Calls in Proxy**: Do not call provider APIs (e.g., Gemini `count_tokens`) from within the proxy addon. Use local tokenizers only.
- **No Shell-Outs in Proxy**: Process identification via `lsof` or `pgrep` must happen in the background worker, not inline.

### 2. Memory Safety for Streaming
- SSE stream reassembly must **stream to disk** (temp files), not accumulate in RAM buffers. With multiple concurrent agent sessions generating large responses, in-memory buffering risks exceeding the proxy's ~200 MB RAM budget.

### 3. JSONL Log Management
- **Daily log rotation**: `agent_logs_YYYY-MM-DD.jsonl` (not a single monolithic file).
- **Async file writing**: Use a writer queue (e.g., Python `QueueHandler` or `aiofiles`) to prevent disk I/O from blocking the proxy's event loop.
- **Filelock-free concurrency**: Rotate to new files daily; concurrent writes within a day should use an async queue to serialize access.

### 4. Credential & Secret Scrubbing
- Scrub `Authorization` headers (OAuth tokens, API keys) from all logged data.
- Apply a **regex-based PII/secret scanner** to request/response bodies before writing to JSONL. Agents often pass codebase secrets, `.env` file contents, or proprietary API keys within prompt bodies.

### 5. Port Conflict Avoidance
- Do not use port `8080` (common local dev server port). Default to a dedicated high port such as `18080` or `3128`.

### 6. Evaluation Scheduler Resilience
- The offline evaluation pipeline must handle laptop sleep, shutdown, and timezone changes gracefully. Use an "elapsed time since last run" trigger, not fixed cron schedules.
- Maintain a persistent evaluation cursor (SQLite or marker file) tracking which JSONL records have been processed.

---

## Part 5: Open-Source Solutions Landscape

As of March 2026, the ecosystem has moved beyond simple logging to complex **Agentic Tracing**. Evaluate these against the constraints in Part 1:

### 1. Arize Phoenix
Part of the Arize ecosystem, optimized for RAG and Agentic workflows.
- **Why it fits this project**: Single Docker container, ~1 GB RAM with SQLite backend, 100% open-source. Excellent for debugging context windows. It includes built-in evaluation metrics (faithfulness, relevancy) and specialized visualizers for vector embedding drifts.
- **Resource Profile**: 0.5 vCPU / 1 GB RAM minimum, ~5-10 GB disk for 3 months retention.
- **Strengths**: OTel-native (OpenInference), built-in eval UI, agent graph visualization.
- **Weaknesses**: Less mature cost analytics than Langfuse; custom attributes needed for cost tracking.

### 2. Langfuse
The most popular open-source platform for production-grade tracing.
- **Why it may NOT fit this project**: Requires ClickHouse + PostgreSQL. Minimum 16 GB RAM. Exceeds the 4 GB budget.
- **Agent Focus**: Excellent nested traces, prompt management, cost analytics.
- **Consider if**: Hardware constraints are relaxed in the future.

### 3. OpenLLMetry (by Traceloop)
A framework-agnostic, OpenTelemetry-native instrumentation library.
- **Why it fits**: Standardized OTel instrumentation. "Instrument once, export anywhere."
- **Limitation**: SDK-based instrumentation — requires modifying agent code or its dependencies, which may not be feasible for all 7 agents.

### 4. Opik (by Comet)
A newer player focusing on the "Build -> Eval -> Monitor" lifecycle.
- **Why it fits**: Tight integration between experiment tracking and production monitoring.

### 5. OpenObserve
An all-in-one observability platform (Logs, Metrics, Traces) that competes with Datadog.
- **Why it fits**: Specific "LLM Observability" module. Performant and cost-effective for high-volume agent logs.

---

## Part 6: Metrics & Analytics Design

When designing your system, ground your selection of analytical metrics into a deep research into the current state-of-the-art in AI agent observability. Below are some indicative metrics presented as a suggestion. Feel free to use them or discard them or modify them or derive better metrics based on your research:

### P0: Quality Assurance (8 Metrics)

| Metric | Formula / Description | Unit | Collection |
|:---|:---|:---|:---|
| Tool Call Success Rate | `successful / total × 100` | % | Real-time |
| Tool Parameter Validity Rate | `valid_params / total × 100` | % | Real-time |
| Tool Fallback Rate | `retried / total × 100` | % | Real-time |
| Trajectory Relevancy Score | LLM-judge score of full session decision path | 1-5 | Offline eval |
| Faithfulness Score | LLM-judge: is output grounded in context? | 1-5 | Offline eval |
| Output Completeness | Did the agent satisfy the user's objective? | bool | Offline eval |
| Hallucination Incidents | Responses with fabricated paths/APIs/facts | count | Offline eval |
| Error Recovery Rate | `recovered / total_errors × 100` | % | Real-time |

### P1: Efficiency Tuning (11 Metrics)

| Metric | Formula / Description | Unit | Collection |
|:---|:---|:---|:---|
| Context Window Utilization | `total_input / model_max_context × 100` | % | Real-time |
| Context Bloat Factor | `(history + redundant_state) / user_message` | ratio | Real-time |
| System Prompt Overhead | `system_prompt / total_input × 100` | % | Real-time |
| Tool Definition Overhead | `tool_defs / total_input × 100` | % | Real-time |
| Tool Discovery Accuracy | `correct_first_attempt / total × 100` | % | Offline eval |
| Tool Discovery Latency (p50/p95) | Time to first tool invocation | ms | Real-time |
| Token-to-Insight Ratio | `output_complexity / total_tokens` | ratio | Offline eval |
| Payload Size per Turn (p50/p95) | Input tokens per LLM request | tokens | Real-time |
| Turns per Task | LLM round-trips per user objective | count | Real-time |
| Skill/KI Discovery Rate | `relevant_used / relevant_available × 100` | % | Offline eval |
| Context Compression Ratio | Tokens saved by summarization vs. raw | ratio | Real-time |

### P2: Cost Optimization (10 Metrics)

| Metric | Formula / Description | Unit | Collection |
|:---|:---|:---|:---|
| Total Cost per Session | Sum of all LLM request costs | USD | Real-time |
| Cost per Turn | Average single-turn cost | USD | Real-time |
| Cache Hit Rate | `cache_read / total_input × 100` | % | Real-time |
| Cache Savings | Cost avoided via cached pricing | USD | Real-time |
| Thinking Token Overhead | `reasoning / output × 100` | % | Real-time |
| Cost per Provider | Aggregated by provider | USD | Real-time |
| Cost per Agent | Aggregated by agent | USD | Real-time |
| Cost per Model | Aggregated by model variant | USD | Real-time |
| Input/Output Token Ratio | `input / output` per request | ratio | Real-time |
| Daily Spend Trend | Rolling daily cost | USD/day | Derived |

### P3: Debugging (6 Metrics)

| Metric | Formula / Description | Unit | Collection |
|:---|:---|:---|:---|
| Request Latency (p50/p95/p99) | Total LLM request duration | ms | Real-time |
| Time-to-First-Token (p50/p95) | Delay before streaming begins | ms | Real-time |
| Tool Execution Latency (p50/p95) | Duration of tool execution | ms | Real-time |
| Error Rate per Agent | `errors / total × 100` per agent | % | Real-time |
| Error Rate per Provider | `errors / total × 100` per provider | % | Real-time |
| Finish Reason Distribution | Breakdown of stop reasons | histogram | Real-time |

### Configurable Alerts

| Alert | Trigger Condition | Default Threshold |
|:---|:---|:---|
| Tool Definition Budget Exceeded | `tool_defs / total_input` | > 30% |
| Context Window Critical | `window_utilization` | > 80% |
| Cost Anomaly | Session cost vs. 7-day rolling average | > 3× |
| High Tool Failure Rate | Tool fallback rate over 1-hour window | > 20% |

---

## Part 7: Future Roadmap (Post-MVP)

After the FRS is finalized and MVP is delivered, the following capabilities should be planned:

### Advanced Analytics

| Capability | Description |
|:---|:---|
| **Loop & Redundancy Detection** | Action fingerprinting (hash tool name + normalized params). Detect repeated actions in sliding windows. |
| **Decision Path DAG** | Visualize full decision tree as a directed acyclic graph with causal edges. |
| **Comparative Agent Benchmarking** | Same task → multiple agents. Compare quality, cost, latency side-by-side. |
| **Prompt A/B Testing** | Version-controlled prompt variants with automated eval scoring and statistical significance testing. |
| **Embedding Drift Detection** | Track vector embedding distributions over time. Alert on retrieval quality degradation. |
| **Multi-Agent Orchestration Tracing** | Full trace visibility across orchestrated multi-agent workflows (CrewAI, AutoGen). |

### Dashboard Specifications

1.  **Agent Intelligence Scorecard**: Per-agent quality trend, efficiency index, cost rank, top 5 failure modes.
2.  **Context Economy Monitor**: Stacked area chart of context composition, alert thresholds, historical trends per agent.
3.  **Provider Economics**: Sankey diagram (Provider → Model → Agent), cache efficiency scatter plot, monthly burn rate projection.
4.  **Decision Path Explorer**: Interactive DAG rendering with loop/cycle highlighting and side-by-side run comparison.
5.  **Skill & Tool Discovery Analytics**: Tool selection accuracy trends, skill discovery funnel, tool category distribution, discovery time correlation matrix.

---

## Part 8: Solution Recommendation Phase

**Only after the FRS is finalized** should you proceed to suggest a specific technology stack. The recommendation must:

1.  Validate every component against the **4 GB RAM** and **zero-cost** constraints.
2.  Provide a complete Docker Compose configuration with resource limits.
3.  Include per-agent proxy configuration instructions (env vars and CA trust setup).
4.  Address all architectural resiliency requirements from Part 4.
5.  Define a phased implementation timeline (MVP → Quality → Scale).