# Functional Requirement Specification: AI Agent Observability System

---

## Index of Contents

- [1. Executive Summary](#1-executive-summary)
  - [1.1 Phase 1 Provider Coverage](#11-phase-1-provider-coverage)
- [2. System Context](#2-system-context)
  - [2.1 Agent Inventory](#21-agent-inventory)
  - [2.2 Provider API Endpoints (MVP Scope)](#22-provider-api-endpoints-mvp-scope)
  - [2.3 Priority Matrix](#23-priority-matrix)
- [3. Architecture Overview](#3-architecture-overview)
  - [3.1 Component Architecture](#31-component-architecture)
  - [3.2 Request/Response Sequence ("Dumb and Fast" Proxy)](#32-requestresponse-sequence-dumb-and-fast-proxy)
  - [3.3 Sub-Agent Tracing Sequence](#33-sub-agent-tracing-sequence)
  - [3.4 Component Summary](#34-component-summary)
- [4. Interception Layer: The OAuth Proxy Solution](#4-interception-layer-the-oauth-proxy-solution)
  - [4.1 Problem Statement](#41-problem-statement)
  - [4.2 Solution: mitmproxy as Transparent HTTPS Proxy](#42-solution-mitmproxy-as-transparent-https-proxy)
  - [4.3 Certificate Trust: Agent-Specific Configuration](#43-certificate-trust-agent-specific-configuration-no-system-wide-trust)
  - [4.4 Agent Identification Strategy](#44-agent-identification-strategy)
  - [4.5 Streaming (SSE) Handling](#45-streaming-sse-handling)
- [5. Data Model & Trace Hierarchy](#5-data-model--trace-hierarchy)
  - [5.1 Trace Hierarchy (OTel-Aligned)](#51-trace-hierarchy-otel-aligned)
  - [5.2 Skill Discovery Tracking](#52-skill-discovery-tracking)
  - [5.3 Span Attributes (`llm.*` Namespace)](#53-span-attributes-llm-namespace)
  - [5.4 Raw Log Schema (JSONL Archive)](#54-raw-log-schema-jsonl-archive)
- [6. Metrics Definition](#6-metrics-definition)
  - [6.1 P0: Quality Assurance Metrics](#61-p0-quality-assurance-metrics)
  - [6.2 P1: Efficiency Tuning Metrics](#62-p1-efficiency-tuning-metrics)
  - [6.3 P2: Cost Optimization Metrics](#63-p2-cost-optimization-metrics)
  - [6.4 P3: Debugging Metrics](#64-p3-debugging-metrics)
  - [6.5 Alerts & Thresholds](#65-alerts--thresholds)
- [7. Pricing Configuration (Multi-Provider)](#7-pricing-configuration-multi-provider)
- [8. Context Composition Analysis Engine](#8-context-composition-analysis-engine)
  - [8.1 Input Decomposition Strategy](#81-input-decomposition-strategy)
  - [8.2 Token Counting: Dual Tokenizer Strategy](#82-token-counting-dual-tokenizer-strategy)
  - [8.3 Content Versioning](#83-content-versioning)
- [9. Observability Platform: Arize Phoenix (Self-Hosted)](#9-observability-platform-arize-phoenix-self-hosted)
  - [9.1 Justification: Why Phoenix Instead of Langfuse](#91-justification-why-phoenix-instead-of-langfuse)
  - [9.2 Phoenix Docker Setup](#92-phoenix-docker-setup)
  - [9.3 Resource Budget](#93-resource-budget)
- [10. Offline Evaluation Pipeline](#10-offline-evaluation-pipeline)
  - [10.1 Architecture](#101-architecture)
  - [10.2 Judge Model: OAuth-Authenticated Access](#102-judge-model-oauth-authenticated-access)
  - [10.3 Evaluation Scheduler & State Management](#103-evaluation-scheduler--state-management)
  - [10.4 Evaluation Rubrics](#104-evaluation-rubrics)
- [11. Provider-Specific Parsing Rules](#11-provider-specific-parsing-rules)
  - [11.1 Anthropic](#111-anthropic)
  - [11.2 OpenAI](#112-openai)
  - [11.3 Google (Gemini)](#113-google-gemini)
- [12. Functional Requirements: Proxy vs. Worker Split](#12-functional-requirements-proxy-vs-worker-split)
  - [12.1 Proxy Addon Functions (Thin — Critical Path)](#121-proxy-addon-functions-thin--critical-path)
  - [12.2 Background Worker Functions (Heavy — Async)](#122-background-worker-functions-heavy--async)
  - [12.3 Hot-Path Failure Model](#123-hot-path-failure-model)
- [13. Data Retention & Lifecycle](#13-data-retention--lifecycle)
- [14. Success Criteria](#14-success-criteria)
  - [14.1 Phase 1 (MVP) — Target: 4 weeks](#141-phase-1-mvp--target-4-weeks)
  - [14.2 Phase 2 (Quality) — Target: +3 weeks](#142-phase-2-quality--target-3-weeks)
  - [14.3 Phase 3 (Scale) — Target: +3 weeks](#143-phase-3-scale--target-3-weeks)
- [15. Future Roadmap: Analytics & Visualization](#15-future-roadmap-analytics--visualization)
  - [15.1 Advanced Analytics (Post-MVP)](#151-advanced-analytics-post-mvp)
  - [15.2 Visualization Capabilities Roadmap](#152-visualization-capabilities-roadmap)
  - [15.3 Future Dashboard Specifications](#153-future-dashboard-specifications)
- [16. Resolved Decisions](#16-resolved-decisions)
- [Verification Plan](#verification-plan)

---

## 1. Executive Summary

This FRS defines a **self-hosted, proxy-based observability system** for monitoring 7 AI coding agents across 6 LLM providers on a local developer workstation. The system provides **proxy-observable telemetry** — LLM request/response interception with derived analytics — covering quality assurance, efficiency tuning, cost optimization, and debugging, in that priority order.

> [!IMPORTANT]
> **Telemetry Scope**: The proxy intercepts all LLM API traffic and provides granular token, cost, and context composition data. Tool execution spans and skill discovery events are **inferred** from request/response content analysis (tool_use blocks, SKILL.md references in messages), not from true white-box agent instrumentation. True agent-internal spans (reasoning steps, tool execution timing, skill evaluation logic) would require SDK-level instrumentation within each agent's codebase, which is out of scope for the proxy-based architecture. The inferred spans are best-effort reconstructions suitable for trend analysis and offline evaluation, not real-time debugging of agent internals.

**Scope**: Local workstation only. 20+ sessions/day currently, scaling to hundreds of automated agent runs. 70+ tools across agents. 3-month data retention.

**Hardware Constraint**: MacBook with 16 GB total RAM. The entire observability stack must operate within **≤4 GB RAM**.

**Cost Constraint**: Zero incremental cost. All evaluation and processing must use existing OAuth-authenticated provider subscriptions or local computation.

### 1.1 Phase 1 Provider Coverage

| Provider | Agents | Phase 1 Telemetry Level |
|:---|:---|:---|
| Anthropic | Claude Code | **Full** — normalized parsing, cost, context decomposition |
| OpenAI | Codex | **Full** — normalized parsing, cost, context decomposition |
| Google (Gemini) | Antigravity, Gemini CLI | **Full** — normalized parsing, cost, context decomposition |
| AMP (Sourcegraph) | AMP Code CLI | **Raw pass-through** — JSONL logging of raw req/res only |
| Kilo Code | Kilo Code CLI | **Raw pass-through** — JSONL logging of raw req/res only |
| Nous Research | Hermes | **Raw pass-through** — JSONL logging of raw req/res only |

> [!NOTE]
> Agents targeting deferred providers still benefit from proxy routing, credential scrubbing, and raw archival. Normalized parsing rules for AMP, Kilo Code, and Nous Research will be reverse-engineered in Phase 3.

---

## 2. System Context

### 2.1 Agent Inventory

| Agent | Primary Provider | Auth Mechanism | Proxy Support | CA Trust Mechanism |
|:---|:---|:---|:---|:---|
| Claude Code | Anthropic | OAuth | ✅ `HTTPS_PROXY` env var | `NODE_EXTRA_CA_CERTS` |
| Antigravity | Google (Gemini) | OAuth | ✅ `HTTPS_PROXY` env var | `NODE_EXTRA_CA_CERTS` |
| Codex | OpenAI | OAuth / API Key | ✅ `HTTPS_PROXY` + `WSS_PROXY` | OS-level or `SSL_CERT_FILE` |
| Gemini CLI | Google | OAuth | ✅ `HTTPS_PROXY` or `settings.json` | OS-level or `NODE_EXTRA_CA_CERTS` |
| AMP Code CLI | AMP (Sourcegraph) | OAuth | ✅ `HTTPS_PROXY` env var | `NODE_EXTRA_CA_CERTS` |
| Kilo Code CLI | Kilo Code | OAuth / API Key | ✅ `HTTPS_PROXY` or VS Code settings | VS Code `http.proxyStrictSSL` |
| Hermes | Nous Research | API Key | ✅ `HTTPS_PROXY` env var | `REQUESTS_CA_BUNDLE` / `SSL_CERT_FILE` |

> [!NOTE]
> **All 7 agents support `HTTPS_PROXY`**. Research confirms each agent's underlying HTTP client (Node.js, Python `requests`, Rust) respects standard proxy environment variables. No hardcoded HTTP clients were identified. The Codex CLI additionally requires `WSS_PROXY` for WebSocket traffic (v0.104.0+).

### 2.2 Provider API Endpoints (MVP Scope)

| Provider | Base URL(s) | Token Types |
|:---|:---|:---|
| Anthropic | `api.anthropic.com` | input, output, cache_read, cache_creation |
| Google (Gemini) | `generativelanguage.googleapis.com` | input, output, thinking |
| OpenAI | `api.openai.com` | input, output, reasoning, cached_input |

> [!NOTE]
> **Deferred to Future Roadmap**: AMP, Kilo Code, and Nous Research provider API schemas will be reverse-engineered in a later phase once the core system is stable with the three primary providers.

### 2.3 Priority Matrix

| Priority | Domain | Goal |
|:---|:---|:---|
| P0 | Quality Assurance | Detect hallucinations, bad tool calls, parameter errors |
| P1 | Efficiency Tuning | Minimize context window utilization, minimize payload per turn |
| P2 | Cost Optimization | Track token spend, maximize cache hit rate |
| P3 | Debugging | Trace failures, inspect decision paths |

---

## 3. Architecture Overview

### 3.1 Component Architecture

```mermaid
graph TB
    subgraph Workstation["LOCAL WORKSTATION (16 GB RAM)"]
        subgraph Agents["AI Coding Agents"]
            CC["Claude Code"]
            AG["Antigravity"]
            CX["Codex"]
            GC["Gemini CLI"]
            AMP["AMP Code CLI"]
            KC["Kilo Code CLI"]
            HM["Hermes"]
        end

        subgraph Proxy["Interception Layer (~150 MB RAM)"]
            MP["mitmproxy<br/>+ Thin Addon<br/>(capture + queue only)"]
        end

        subgraph Worker["Background Worker (~200 MB RAM)"]
            BW["Async Processing Worker<br/>(Python process)"]
        end

        subgraph Storage["Storage & Analysis (~1 GB RAM)"]
            PX["Arize Phoenix<br/>(SQLite backend)"]
            JL["JSONL Archive<br/>(daily rotation)"]
            CV["Content Versions<br/>(SHA-256 indexed)"]
        end

        subgraph Eval["Offline Evaluation (on-demand)"]
            EP["Eval Pipeline<br/>(Python scripts)"]
            JM["LLM Judge<br/>(via OAuth)"]
        end

        subgraph Providers["LLM Provider APIs"]
            AN["Anthropic API"]
            OA["OpenAI API"]
            GO["Google Gemini API"]
        end

        CC -->|"HTTPS_PROXY<br/>+ NODE_EXTRA_CA_CERTS"| MP
        AG -->|"HTTPS_PROXY<br/>+ NODE_EXTRA_CA_CERTS"| MP
        CX -->|"HTTPS_PROXY<br/>+ WSS_PROXY"| MP
        GC -->|"HTTPS_PROXY"| MP
        AMP -->|"HTTPS_PROXY"| MP
        KC -->|"HTTPS_PROXY"| MP
        HM -->|"HTTPS_PROXY<br/>+ REQUESTS_CA_BUNDLE"| MP

        MP -->|"Forward request<br/>(OAuth preserved)"| AN
        MP -->|"Forward request<br/>(OAuth preserved)"| OA
        MP -->|"Forward request<br/>(OAuth preserved)"| GO

        AN -->|"Response<br/>(usage metadata)"| MP
        OA -->|"Response<br/>(usage metadata)"| MP
        GO -->|"Response<br/>(usage metadata)"| MP

        MP -->|"Raw payload<br/>(staging dir)"| BW
        BW -->|"Normalized spans<br/>(OTLP gRPC :4317)"| PX
        BW -->|"Sanitized records<br/>(async writer)"| JL
        BW -->|"Content hashes<br/>(deduplicated)"| CV

        JL -->|"Batch read"| EP
        EP -->|"OAuth API call"| JM
        EP -->|"Scores"| PX
    end

    style Workstation fill:#1a1a2e,stroke:#16213e,color:#e0e0e0
    style Agents fill:#0f3460,stroke:#533483,color:#e0e0e0
    style Proxy fill:#533483,stroke:#e94560,color:#e0e0e0
    style Worker fill:#4a1942,stroke:#e94560,color:#e0e0e0
    style Storage fill:#16213e,stroke:#0f3460,color:#e0e0e0
    style Eval fill:#1a1a2e,stroke:#533483,color:#e0e0e0
    style Providers fill:#0f3460,stroke:#16213e,color:#e0e0e0
```

### 3.2 Request/Response Sequence ("Dumb and Fast" Proxy)

```mermaid
sequenceDiagram
    participant Agent as AI Agent<br/>(e.g., Claude Code)
    participant Proxy as mitmproxy<br/>(thin addon)
    participant Provider as LLM Provider<br/>(e.g., Anthropic)
    participant Staging as Staging Directory<br/>(on disk)
    participant Worker as Background Worker
    participant Phoenix as Arize Phoenix
    participant JSONL as JSONL Archive

    Note over Agent,Provider: Phase 1: OAuth Authentication (one-time)
    Agent->>Provider: OAuth flow (browser-based)
    Provider-->>Agent: OAuth token issued
    Note over Agent,Provider: Proxy is NOT in the OAuth path

    Note over Agent,Provider: Phase 2: API Request Flow
    Agent->>Proxy: POST /v1/messages<br/>Authorization: Bearer <token><br/>Body: {system, messages, tools}
    
    Note over Proxy: Thin addon (fast path):<br/>1. Identify provider (URL match)<br/>2. Extract agent identity from<br/>   proxy-auth or User-Agent<br/>3. Record timestamps<br/>4. Stream request body to staging file

    Proxy->>Provider: Forward request UNMODIFIED<br/>(OAuth token preserved,<br/>proxy-auth credentials stripped)
    
    alt Non-Streaming Response
        Provider-->>Proxy: 200 OK<br/>Body: {content, usage, stop_reason}
        Note over Proxy: Write response body<br/>to staging file
    else Streaming Response (SSE)
        Provider-->>Proxy: 200 OK<br/>Content-Type: text/event-stream
        loop SSE Chunks
            Provider-->>Proxy: data: {delta, ...}
            Proxy-->>Agent: Forward chunk unmodified
            Note over Proxy: Append chunk to staging<br/>temp file on disk<br/>(NOT in-memory buffer)
        end
    end

    Proxy-->>Agent: Forward response UNMODIFIED
    Proxy->>Staging: Write completed staging file<br/>(raw req + res + metadata)

    Note over Worker: Background worker polls staging dir
    Worker->>Staging: Read staging file
    Note over Worker: Heavy processing:<br/>1. Parse provider-specific schema<br/>2. Dual tokenizer estimation<br/>3. Content hashing + versioning<br/>4. Cost calculation (pricing.yaml)<br/>5. Credential + secret scrubbing<br/>6. PID/process mapping (if needed)
    Worker->>Phoenix: OTLP gRPC on :4317<br/>(normalized span data)
    Worker->>JSONL: Append scrubbed record<br/>(daily-rotated file)
    Note over Phoenix: Trace visible in UI<br/>with cost, tokens, latency
```

> [!IMPORTANT]
> **"Dumb and Fast" Principle**: The proxy addon performs NO computation on the request/response content — no tokenization, no content hashing, no cost calculation, no secret scrubbing, no process tree inspection. It only (1) identifies the provider URL, (2) extracts agent identity from proxy-auth credentials, (3) streams the raw payload to a staging file on disk, and (4) forwards traffic unmodified. All heavy processing is performed by a separate background worker process.

### 3.3 Sub-Agent Tracing Sequence

Sub-agent correlation uses **proxy-auth URL encoding** as the primary mechanism because it survives environment variable scrubbing by sandbox environments and Docker containers.

```mermaid
sequenceDiagram
    participant User
    participant Parent as Parent Agent<br/>(e.g., Antigravity)
    participant Proxy as mitmproxy<br/>(:18080)
    participant Child as Child Sub-Agent<br/>(e.g., browser_subagent)
    participant Provider as LLM Provider

    User->>Parent: Complex task request
    Note over Parent: HTTPS_PROXY=<br/>http://antigravity:sess-001@127.0.0.1:18080
    Parent->>Proxy: LLM Request #1<br/>Proxy-Authorization: Basic(antigravity:sess-001)
    Note over Proxy: Extract agent=antigravity,<br/>session=sess-001 from proxy-auth<br/>Strip Proxy-Authorization header
    Proxy->>Provider: Forward (clean request,<br/>no proxy-auth headers)
    Provider-->>Proxy: Response (tool_use: spawn_subagent)
    Proxy-->>Parent: Response

    Note over Parent: Agent spawns sub-agent<br/>Child inherits HTTPS_PROXY env var<br/>(identity + session survive)

    Parent->>Child: Delegate sub-task
    Note over Child: Inherits HTTPS_PROXY=<br/>http://antigravity:sess-001@127.0.0.1:18080
    
    Child->>Proxy: LLM Request #2<br/>Proxy-Authorization: Basic(antigravity:sess-001)
    Note over Proxy: Same session ID → same trace<br/>Temporal ordering determines<br/>parent-child relationship
    Proxy->>Provider: Forward (clean)
    Provider-->>Proxy: Response
    Proxy-->>Child: Response

    Child-->>Parent: Sub-task result

    Note over Proxy: Both spans linked via<br/>session=sess-001 from proxy-auth URL.<br/>Worker later resolves hierarchy via<br/>temporal ordering + process tree.
```

> [!NOTE]
> **Inject-and-Strip Mechanism**: The proxy-auth credentials (`agentname:sessionid`) are embedded in the `HTTPS_PROXY` URL. When agents make requests through the proxy, mitmproxy receives these as `Proxy-Authorization` headers (HTTP CONNECT) or userinfo in the URL. The addon extracts the agent name and session ID, then **strips the `Proxy-Authorization` header** before forwarding to the provider. The provider never sees the observability metadata. Process tree inspection (`pgrep -P <parent_pid>`) runs only in the background worker as a fallback for disambiguating sub-agent parent relationships.

### 3.4 Component Summary

| Component | Role | Technology | RAM Budget |
|:---|:---|:---|:---|
| **Interception Layer** | Thin HTTPS proxy — capture and queue only | `mitmproxy` + thin Python addon | ~150 MB |
| **Background Worker** | Tokenization, hashing, cost calc, OTLP export | Python process (asyncio) | ~200 MB |
| **Trace Storage & UI** | Stores traces, visualizes spans | Arize Phoenix (SQLite) | ~1 GB |
| **Raw Log Archive** | Full request/response bodies | Daily-rotated JSONL files on local disk | ~0 MB (disk only) |
| **Content Versions** | Deduplicated prompt/tool tracking | SHA-256 indexed files on disk | ~0 MB (disk only) |
| **Evaluation Engine** | Offline quality scoring | Python scripts (on-demand) | ~500 MB (on-demand) |
| **Total** | | | **~1.85 GB** (≤4 GB budget ✅) |

---

## 4. Interception Layer: The OAuth Proxy Solution

### 4.1 Problem Statement

All major AI coding agents authenticate with their providers via **OAuth 2.0 flows** — not user-supplied API keys. This means:
- A traditional API gateway (LiteLLM) cannot be used, because it expects to *own* the credentials.
- The agents initiate browser-based OAuth, receive tokens, and embed them in `Authorization: Bearer <token>` headers.

### 4.2 Solution: mitmproxy as Transparent HTTPS Proxy

**Architecture**: `mitmproxy` operates as a forward HTTPS proxy. Agents are configured to route traffic through it via **agent-specific environment variables only** (no system-wide CA trust).

**How OAuth works through the proxy**:

1. Agent initiates OAuth flow → Browser opens provider's auth page → User authenticates → Provider issues OAuth token to the agent.
2. Agent sends API requests with `Authorization: Bearer <token>` through the proxy.
3. mitmproxy terminates TLS (using its own CA cert), reads the request/response, then re-encrypts and forwards to the provider.
4. The OAuth token passes through **unmodified** — the proxy is invisible to the authentication flow.

> [!IMPORTANT]
> **LiteLLM Verdict**: LiteLLM is **not suitable** for this use case. Its `forward_llm_provider_auth_headers` feature is designed for provider API keys, not OAuth JWTs. When `master_key` is enabled, LiteLLM's auth middleware intercepts and validates the `Authorization` header before it can reach pass-through logic, causing 401 errors. Additionally, LiteLLM requires ClickHouse or PostgreSQL for its analytics, which would exceed the 4 GB RAM budget.

### 4.3 Certificate Trust: Agent-Specific Configuration (No System-Wide Trust)

The mitmproxy CA certificate is trusted **only by the AI agent processes**, not system-wide. This is achieved by setting agent-specific environment variables:

```bash
# ~/.zshrc or agent launcher scripts — NOT system keychain

# === mitmproxy CA cert path ===
export MITMPROXY_CA="$HOME/.mitmproxy/mitmproxy-ca-cert.pem"

# === Proxy Settings (all agents) ===
# Agent identity and session ID are encoded in the proxy URL userinfo.
# Format: http://<agent-name>:<session-id>@127.0.0.1:18080
# The mitmproxy addon extracts these and strips the Proxy-Authorization
# header before forwarding to the provider.
export HTTPS_PROXY="http://default-agent:default-session@127.0.0.1:18080"
export HTTP_PROXY="http://127.0.0.1:18080"
export NO_PROXY="localhost,127.0.0.1,.local"

# === Node.js agents (Claude Code, Antigravity, AMP Code, Gemini CLI) ===
export NODE_EXTRA_CA_CERTS="$MITMPROXY_CA"

# === Python agents (Hermes) ===
export REQUESTS_CA_BUNDLE="$MITMPROXY_CA"
export SSL_CERT_FILE="$MITMPROXY_CA"

# === Rust agents (Codex) ===
# Codex uses the Rust `reqwest` library which respects SSL_CERT_FILE.
# WebSocket traffic also routes through the proxy.
export WSS_PROXY="http://codex:default-session@127.0.0.1:18080"
export SSL_CERT_FILE="$MITMPROXY_CA"

# === Kilo Code (VS Code extension) ===
# Configure in VS Code settings.json:
# "http.proxy": "http://kilo-code:default-session@127.0.0.1:18080"
# "http.proxyStrictSSL": false
```

> [!NOTE]
> **Security**: Regular browser traffic, system updates, and non-agent applications are NOT affected by this configuration. Only processes launched from the configured shell environment (or with these env vars) will use the proxy. The proxy binds to `127.0.0.1` only — it is not accessible from the network.

> [!WARNING]
> **Codex & Gemini CLI CA Trust**: These agents use different underlying HTTP libraries (Rust `reqwest` for Codex, Node.js for Gemini CLI). Both respect `SSL_CERT_FILE` and `NODE_EXTRA_CA_CERTS` respectively. If an agent is observed to pin certificates (rejecting the mitmproxy CA), the fallback strategy is SDK-level instrumentation via OpenLLMetry. No certificate-pinning agents have been identified in the current inventory, but this should be validated during Phase 1.

### 4.4 Agent Identification Strategy

#### Primary: Proxy-Auth URL Encoding

Agent identity and session ID are embedded in the `HTTPS_PROXY` URL as userinfo credentials. This is the most robust method because it survives environment variable scrubbing by sandbox environments and Docker containers.

| Agent | HTTPS_PROXY Value | Identity Extracted |
|:---|:---|:---|
| Claude Code | `http://claude-code:<session>@127.0.0.1:18080` | `agent=claude-code` |
| Antigravity | `http://antigravity:<session>@127.0.0.1:18080` | `agent=antigravity` |
| Codex | `http://codex:<session>@127.0.0.1:18080` | `agent=codex` |
| Gemini CLI | `http://gemini-cli:<session>@127.0.0.1:18080` | `agent=gemini-cli` |
| AMP Code CLI | `http://amp-code:<session>@127.0.0.1:18080` | `agent=amp-code` |
| Kilo Code CLI | `http://kilo-code:<session>@127.0.0.1:18080` | `agent=kilo-code` |
| Hermes | `http://hermes:<session>@127.0.0.1:18080` | `agent=hermes` |

**How it works**: Wrapper launch scripts (one per agent) set `HTTPS_PROXY` with the agent name as the username and a session UUID as the password. When the agent makes requests through the proxy, mitmproxy receives a `Proxy-Authorization: Basic <base64>` header during the HTTP CONNECT handshake. The thin addon decodes the credentials, extracts the agent name and session ID, then **strips the `Proxy-Authorization` header** before forwarding to the provider.

```bash
# Example wrapper script: launch-claude-code.sh
#!/bin/bash
SESSION_ID="sess-$(date +%s)-$(openssl rand -hex 4)"
export HTTPS_PROXY="http://claude-code:${SESSION_ID}@127.0.0.1:18080"
export HTTP_PROXY="http://127.0.0.1:18080"
export NODE_EXTRA_CA_CERTS="$HOME/.mitmproxy/mitmproxy-ca-cert.pem"
exec claude "$@"
```

#### Fallback: User-Agent Parsing

If proxy-auth is not feasible for an agent (e.g., it strips proxy credentials):
1. Parse `User-Agent` header (e.g., Claude Code sends `anthropic-sdk/...`).
2. Process tree inspection (`pgrep`, `lsof`) as a last resort — this runs **only in the background worker**, never in the proxy's hot path.

#### Sub-Agent Identification

When a parent agent spawns child sub-agents (e.g., Antigravity's `browser_subagent`):

1. **Environment Inheritance**: Child processes inherit the parent's `HTTPS_PROXY` env var, which contains both the agent name and session ID. This is the key advantage of using the proxy URL for identity — it survives all forms of subprocess spawning.
2. **Session Correlation**: Same session ID in the proxy-auth → same trace in Phoenix.
3. **Parent-Child Resolution**: The background worker determines parent-child relationships via:
   - Same session ID → same trace
   - Temporal ordering → parent span is the one that preceded the sub-agent spawn
   - Process tree inspection (background fallback): `pgrep -P <parent_pid>` to discover child processes
4. **Naming Convention**: Sub-agents are identified by dotted convention where possible. The eval pipeline can infer sub-agent identity from response content (e.g., `tool_use: browser_subagent`).

```
# Example trace hierarchy with sub-agent:
Session: sess-001 (antigravity)
  ├── Turn 1: User request
  │     ├── LLM Request (antigravity) → response includes "spawn browser_subagent"
  │     ├── [Inferred] Sub-Agent Requests (same session ID, temporal gap)
  │     │     ├── LLM Request (sub-agent — inferred from proxy-auth)
  │     │     └── LLM Request (sub-agent)
  │     └── LLM Request (antigravity) → processes sub-agent result
  └── Session Summary
```

### 4.5 Streaming (SSE) Handling

Most LLM APIs use Server-Sent Events (SSE) for streaming responses. The mitmproxy addon must stream to disk, NOT buffer in memory.

1. **Detect SSE streams** via `Content-Type: text/event-stream` in `responseheaders` hook.
2. **Stream chunks to a temporary file on disk** — each chunk is appended as it passes through, then forwarded unmodified to the agent.
3. After stream completion, the staging file (containing the full reassembled response) is picked up by the background worker.
4. **Parse `data:` lines** happens in the background worker, not the proxy.

```python
# Pseudocode for SSE handling in mitmproxy addon (disk-based streaming)
import tempfile
import os

STAGING_DIR = os.path.expanduser("~/.observability/staging")

def responseheaders(self, flow):
    if "text/event-stream" in flow.response.headers.get("content-type", ""):
        flow.response.stream = self._stream_to_disk(flow)

def _stream_to_disk(self, flow):
    """Returns a callable that streams chunks to disk while passing through."""
    staging_path = os.path.join(STAGING_DIR, f"{flow.id}_response.tmp")
    def stream_modifier(chunks):
        with open(staging_path, 'ab') as f:
            for chunk in chunks:
                f.write(chunk)
                yield chunk  # Pass through unmodified to agent
        # Signal completion — worker picks up from staging dir
        os.rename(staging_path, staging_path.replace('.tmp', '.ready'))
    return stream_modifier
```

> [!IMPORTANT]
> **Memory Safety**: With multiple concurrent agent sessions generating large responses (8K+ output tokens), in-memory buffering risks exceeding the proxy's ~150 MB RAM budget. Streaming to disk ensures the proxy's memory footprint remains constant regardless of response size or concurrency.

---

## 5. Data Model & Trace Hierarchy

### 5.1 Trace Hierarchy (OTel-Aligned)

The system uses a **four-level span hierarchy** aligned with OpenTelemetry GenAI Semantic Conventions (v1.40.0). Spans are categorized by observability source:

```
Session Span (top-level) ........................... [CONSTRUCTED — from proxy-auth session ID]
  ├── Turn Span (one per user↔agent interaction) ... [INFERRED — gap detection between requests]
  │     ├── LLM Request Span (per API call) ........ [DIRECTLY OBSERVED — proxy intercepts req/res]
  │     │     ├── Context Composition Event ......... [COMPUTED — worker parses request body]
  │     │     ├── Token Usage Event ................. [DIRECTLY OBSERVED — from provider response]
  │     │     └── Cost Calculation Event ............ [COMPUTED — worker applies pricing.yaml]
  │     ├── Tool Execution Span .................... [INFERRED — from tool_use blocks in response]
  │     │     ├── Tool Parameter Event .............. [INFERRED — from tool_use input params]
  │     │     └── Tool Result Event ................ [INFERRED — from subsequent tool_result messages]
  │     ├── Skill Discovery Span ................... [INFERRED — from SKILL.md refs in messages]
  │     │     └── Skill Relevance .................. [OFFLINE EVAL — LLM judge scoring]
  │     └── Reasoning Span ......................... [INFERRED — from thinking tokens in response]
  └── Session Summary Event ....................... [COMPUTED — aggregated by worker]
```

> [!WARNING]
> **Observability Limitations**: Tool execution timing, tool discovery latency, and skill evaluation logic are NOT directly observable from the proxy. These spans are reconstructed from request/response content (e.g., `tool_use` blocks in LLM responses, `tool_result` blocks in subsequent requests, `view_file` calls targeting `SKILL.md` files). This means:
> - **Tool execution duration** cannot be precisely measured (only the gap between the LLM response containing `tool_use` and the next LLM request containing `tool_result`).
> - **Skill discovery time** is approximated from the sequence of `view_file` tool calls targeting skill files.
> - **Reasoning spans** are inferred from thinking/reasoning token counts, not from actual reasoning step boundaries.

### 5.2 Skill Discovery Tracking

Skill discovery efficiency (metric EF-10) is tracked by analyzing the **request body content** at the LLM Request level. This analysis is performed by the **background worker**, not the proxy.

**How it works**:
1. The background worker inspects the `system` prompt and `messages` in each archived request for skill-related patterns:
   - References to SKILL.md files being read (via `view_file` tool calls targeting `*/SKILL.md`)
   - Skill names mentioned in system prompts or agent rules
   - Knowledge Item (KI) references in the system context
2. A **Skill Discovery Span** is created (by the worker) when the agent's response includes skill-related tool calls (reading SKILL.md files, checking KI summaries).
3. The span records:
   - `skill.candidates_available`: Total skills/KIs configured for the agent
   - `skill.candidates_evaluated`: Skills/KIs the agent actually considered (read/referenced)
   - `skill.selected`: The skill ultimately used (if any)
   - `skill.discovery_time_ms`: Approximate time from task start to skill invocation (inferred from request timestamps)
   - `skill.relevance_match`: Whether the chosen skill was relevant to the task (offline eval)
4. The **Skill Discovery Rate** (EF-10) is computed as: `relevant_skills_used / relevant_skills_available × 100`

This is primarily computed via **offline evaluation** — the eval pipeline reviews traces where skill-eligible tasks were performed and scores whether the agent found and used the appropriate skill.

### 5.3 Span Attributes (`llm.*` Namespace)

> [!NOTE]
> **Naming Convention**: All attributes use the `llm.*` namespace prefix instead of the OTel default `gen_ai.*`, per project convention. When exporting to external tools, a mapping layer can translate `llm.*` → `gen_ai.*` if needed.

#### Session Span Attributes
| Attribute | Type | Description |
|:---|:---|:---|
| `session.id` | string | Unique session identifier |
| `llm.agent.name` | string | Agent name (e.g., "claude-code", "antigravity") |
| `llm.agent.version` | string | Agent version |
| `llm.agent.parent` | string | Parent agent name (for sub-agents, e.g., "antigravity") |
| `session.start_time` | timestamp | Session start |
| `session.user_objective` | string | Initial user prompt (if capturable) |

#### LLM Request Span Attributes
| Attribute | Type | Description |
|:---|:---|:---|
| `llm.system` | string | Provider name (e.g., "anthropic", "openai") |
| `llm.request.model` | string | Model ID (e.g., "claude-sonnet-4-20250514") |
| `llm.request.temperature` | float | Sampling temperature |
| `llm.request.max_tokens` | int | Max output tokens requested |
| `llm.request.top_p` | float | Nucleus sampling parameter |
| `llm.response.model` | string | Actual model used (may differ from requested) |
| `llm.response.finish_reason` | string | `stop`, `max_tokens`, `tool_use`, etc. |
| `llm.usage.input_tokens` | int | Input token count |
| `llm.usage.output_tokens` | int | Output token count |
| `llm.usage.reasoning_tokens` | int | Thinking/reasoning token count |
| `llm.usage.cache_read_tokens` | int | Tokens served from cache |
| `llm.usage.cache_creation_tokens` | int | Tokens written to cache |
| `llm.latency.ttft_ms` | float | Time to first token (ms) |
| `llm.latency.total_ms` | float | Total request duration (ms) |

#### Context Composition Event Schema
| Field | Type | Description |
|:---|:---|:---|
| `context.system_prompt_tokens` | int | Tokens consumed by system prompt |
| `context.tool_definitions_tokens` | int | Tokens consumed by tool schemas |
| `context.tool_definitions_count` | int | Number of tool definitions sent |
| `context.conversation_history_tokens` | int | Tokens in conversation history |
| `context.conversation_history_turns` | int | Number of history turns included |
| `context.few_shot_tokens` | int | Tokens in few-shot examples |
| `context.dynamic_state_tokens` | int | Tokens for agent state / KI / rules |
| `context.user_message_tokens` | int | Current user message tokens |
| `context.total_input_tokens` | int | Sum of all input components |
| `context.window_utilization_pct` | float | `total_input / model_max_context × 100` |

#### Tool Execution Span Attributes
| Attribute | Type | Description |
|:---|:---|:---|
| `tool.name` | string | Tool function name |
| `tool.category` | string | Tool category (file_ops, search, browser, command, etc.) |
| `tool.parameters` | json | Serialized tool parameters |
| `tool.result.status` | string | `success`, `error`, `timeout` |
| `tool.result.error_message` | string | Error details if failed |
| `tool.execution_time_ms` | float | Tool execution duration |
| `tool.discovery.candidates_considered` | int | Tools evaluated before selection |
| `tool.discovery.time_ms` | float | Time spent selecting this tool |
| `tool.result.output_tokens` | int | Token count of tool output |

#### Skill Discovery Span Attributes
| Attribute | Type | Description |
|:---|:---|:---|
| `skill.name` | string | Skill name (e.g., "tdd-workflow") |
| `skill.candidates_available` | int | Total skills configured for the agent |
| `skill.candidates_evaluated` | int | Skills the agent considered/read |
| `skill.selected` | bool | Whether a skill was ultimately invoked |
| `skill.discovery_time_ms` | float | Time from task start to skill invocation |
| `skill.relevance_match` | bool | Was the chosen skill relevant? (offline eval) |

### 5.4 Raw Log Schema (JSONL Archive)

Every intercepted request/response pair is stored as a JSONL record. Records are written by the **background worker** (not the proxy) to **daily-rotated files** via an async writer queue.

#### File Naming & Rotation
```
~/.observability/logs/
  ├── agent_logs_2026-03-31.jsonl
  ├── agent_logs_2026-03-30.jsonl
  ├── agent_logs_2026-03-29.jsonl
  └── ...
```

- **Daily rotation**: New file at midnight local time. No single monolithic file.
- **Async writing**: The worker uses a `asyncio.Queue` + dedicated writer coroutine to serialize file writes without blocking processing.
- **Retention**: Files older than 90 days are deleted by a cleanup script.

#### Credential & Secret Scrubbing

Before writing to JSONL, the background worker applies a **two-layer scrubbing pipeline**:

1. **Header scrubbing**: Remove `Authorization`, `Proxy-Authorization`, `x-api-key`, and any header matching `*token*` or `*secret*` patterns. Replace values with `[REDACTED]`.
2. **Body secret scanning**: Apply a configurable regex-based scanner to request/response body text. Default patterns:
   ```python
   SECRET_PATTERNS = [
       r'(?i)(api[_-]?key|apikey)\s*[:=]\s*["\']?[\w-]{20,}',      # API keys
       r'(?i)(secret|password|passwd|pwd)\s*[:=]\s*["\']?[^\s"\']+', # Passwords/secrets
       r'sk-[a-zA-Z0-9]{20,}',                                       # OpenAI API keys
       r'key-[a-zA-Z0-9]{20,}',                                      # Anthropic API keys
       r'AIza[0-9A-Za-z_-]{35}',                                      # Google API keys
       r'ghp_[a-zA-Z0-9]{36}',                                        # GitHub tokens
       r'-----BEGIN (RSA |EC )?PRIVATE KEY-----',                     # Private keys
       r'(?i)(aws_access_key_id|aws_secret_access_key)\s*[:=]\s*\S+', # AWS credentials
   ]
   ```
3. Matched patterns are replaced with `[SECRET_REDACTED:<pattern_name>]` to preserve log structure.

> [!IMPORTANT]
> **Privacy**: Agents frequently pass codebase secrets, `.env` file contents, and proprietary API keys within prompt bodies. Header-only scrubbing is insufficient. The regex scanner runs on the full serialized body before JSONL write. Custom patterns can be added to `~/.observability/scrub_patterns.yaml`.

#### Record Schema

```json
{
  "timestamp": "2026-03-31T01:00:00.000Z",
  "trace_id": "abc123",
  "span_id": "def456",
  "agent": "claude-code",
  "session_id": "sess-1711843200-a1b2c3d4",
  "agent_parent": null,
  "provider": "anthropic",
  "model": "claude-sonnet-4-20250514",
  "scrubbing": {
    "headers_scrubbed": ["Authorization"],
    "body_secrets_found": 0
  },
  "request": {
    "method": "POST",
    "url": "https://api.anthropic.com/v1/messages",
    "headers": { "...sanitized — OAuth tokens REMOVED..." },
    "body": {
      "system": "You are an AI coding assistant...",
      "messages": [ "...full message array..." ],
      "tools": [ "...tool definitions..." ],
      "max_tokens": 8192,
      "temperature": 0.7
    }
  },
  "response": {
    "status_code": 200,
    "headers": { "..." },
    "body": {
      "content": [ "...response content blocks..." ],
      "usage": {
        "input_tokens": 12540,
        "output_tokens": 1834,
        "cache_read_input_tokens": 8200,
        "cache_creation_input_tokens": 0
      },
      "stop_reason": "tool_use"
    },
    "streaming": true,
    "stream_duration_ms": 3420
  },
  "context_composition": {
    "system_prompt_tokens": 2100,
    "tool_definitions_tokens": 5800,
    "tool_definitions_count": 15,
    "conversation_history_tokens": 3200,
    "user_message_tokens": 1440,
    "total_input_tokens": 12540,
    "window_utilization_pct": 6.1
  },
  "cost": {
    "input_cost_usd": 0.012,
    "output_cost_usd": 0.027,
    "cache_savings_usd": 0.008,
    "total_cost_usd": 0.031
  }
}
```

---

## 6. Metrics Definition

### 6.1 P0: Quality Assurance Metrics  

| Metric ID | Metric Name | Formula | Unit | Collection |
|:---|:---|:---|:---|:---|
| QA-01 | **Tool Call Success Rate** | `successful_tool_calls / total_tool_calls × 100` | % | Near-real-time (worker) |
| QA-02 | **Tool Parameter Validity Rate** | `valid_parameter_calls / total_tool_calls × 100` | % | Near-real-time (worker) |
| QA-03 | **Tool Fallback Rate** | `retried_tool_calls / total_tool_calls × 100` | % | Near-real-time (worker) |
| QA-04 | **Trajectory Relevancy Score** | LLM-judge score of entire session decision path (1-5) | score | Offline eval |
| QA-05 | **Faithfulness Score** | LLM-judge: is output grounded in retrieved context? (1-5) | score | Offline eval |
| QA-06 | **Output Completeness** | Did the agent fully satisfy the user's stated objective? (binary + confidence) | bool | Offline eval |
| QA-07 | **Hallucination Incidents** | Count of responses containing fabricated file paths, APIs, or facts | count | Offline eval |
| QA-08 | **Error Recovery Rate** | `errors_recovered_from / total_errors × 100` | % | Near-real-time (worker) |

### 6.2 P1: Efficiency Tuning Metrics

| Metric ID | Metric Name | Formula | Unit | Collection |
|:---|:---|:---|:---|:---|
| EF-01 | **Context Window Utilization** | `total_input_tokens / model_max_context × 100` | % | Near-real-time (worker) |
| EF-02 | **Context Bloat Factor** | `(conversation_history_tokens + redundant_state) / user_message_tokens` | ratio | Near-real-time (worker) |
| EF-03 | **System Prompt Overhead** | `system_prompt_tokens / total_input_tokens × 100` | % | Near-real-time (worker) |
| EF-04 | **Tool Definition Overhead** | `tool_definition_tokens / total_input_tokens × 100` | % | Near-real-time (worker) |
| EF-05 | **Tool Discovery Accuracy** | `correct_tool_selected_first_attempt / total_tool_invocations × 100` | % | Offline eval |
| EF-06 | **Tool Discovery Latency (p50/p95)** | Time from user request to first tool invocation | ms | Near-real-time (worker) |
| EF-07 | **Token-to-Insight Ratio** | `output_complexity_score / total_tokens_consumed` | ratio | Offline eval |
| EF-08 | **Payload Size per Turn (p50/p95)** | Input tokens per LLM request | tokens | Near-real-time (worker) |
| EF-09 | **Turns per Task** | Number of LLM round-trips to complete a user objective | count | Near-real-time (worker) |
| EF-10 | **Skill/KI Discovery Rate** | `relevant_skills_used / relevant_skills_available × 100` | % | Offline eval |
| EF-11 | **Context Compression Ratio** | Tokens saved by summarization/compression vs. raw history | ratio | Near-real-time (worker) |

### 6.3 P2: Cost Optimization Metrics

| Metric ID | Metric Name | Formula | Unit | Collection |
|:---|:---|:---|:---|:---|
| CO-01 | **Total Cost per Session** | Sum of all LLM request costs in a session | USD | Near-real-time (worker) |
| CO-02 | **Cost per Turn** | Average cost of a single turn across all sessions | USD | Near-real-time (worker) |
| CO-03 | **Cache Hit Rate** | `cache_read_tokens / total_input_tokens × 100` | % | Near-real-time (worker) |
| CO-04 | **Cache Savings** | Cost avoided due to cached token pricing | USD | Near-real-time (worker) |
| CO-05 | **Thinking Token Overhead** | `reasoning_tokens / output_tokens × 100` | % | Near-real-time (worker) |
| CO-06 | **Cost per Provider** | Aggregated cost grouped by provider | USD | Near-real-time (worker) |
| CO-07 | **Cost per Agent** | Aggregated cost grouped by agent | USD | Near-real-time (worker) |
| CO-08 | **Cost per Model** | Aggregated cost grouped by model variant | USD | Near-real-time (worker) |
| CO-09 | **Input/Output Token Ratio** | `input_tokens / output_tokens` per request | ratio | Near-real-time (worker) |
| CO-10 | **Daily Spend Trend** | Rolling daily cost aggregation | USD/day | Derived |

### 6.4 P3: Debugging Metrics

| Metric ID | Metric Name | Formula | Unit | Collection |
|:---|:---|:---|:---|:---|
| DB-01 | **Request Latency (p50/p95/p99)** | Total LLM request duration | ms | Near-real-time (worker) |
| DB-02 | **Time-to-First-Token (p50/p95)** | Delay before streaming begins | ms | Near-real-time (worker) |
| DB-03 | **Tool Execution Latency (p50/p95)** | Duration of tool execution (inferred from request gaps) | ms | Near-real-time (worker) |
| DB-04 | **Error Rate per Agent** | `error_responses / total_responses × 100` per agent | % | Near-real-time (worker) |
| DB-05 | **Error Rate per Provider** | `error_responses / total_responses × 100` per provider | % | Near-real-time (worker) |
| DB-06 | **Finish Reason Distribution** | Breakdown of stop reasons (stop, max_tokens, tool_use) | histogram | Near-real-time (worker) |
| DB-07 | **Latency Decomposition** | Breakdown: proxy overhead (staging write time), TTFT (time to first response byte from provider), remote duration (first response byte to last response byte, i.e. generation tail) | ms per component | Near-real-time (worker) |

> [!NOTE]
> **Latency Decomposition (DB-07)**: The proxy records four timestamps per request: (1) request received from agent, (2) request forwarded to provider, (3) first response byte from provider, (4) last response byte. From these, the worker computes: **proxy overhead** = (2)−(1), **TTFT** = (3)−(2), **remote duration** = (4)−(3). True model-vs-network separation is NOT observable from the proxy — the TTFT and remote duration figures include both provider processing and network transit, which cannot be disaggregated without provider-side instrumentation.

### 6.5 Alerts & Thresholds

| Alert | Condition | Action |
|:---|:---|:---|
| **Tool Definition Budget Exceeded** | `tool_definition_tokens / total_input_tokens > 30%` | Log warning + flag in Phoenix |
| **Context Window Critical** | `context.window_utilization_pct > 80%` | Log warning |
| **Cost Anomaly** | Single session cost > 3× rolling 7-day average | Log warning |
| **High Tool Failure Rate** | `tool_fallback_rate > 20%` over rolling 1-hour window | Log warning |

---

## 7. Pricing Configuration (Multi-Provider)

```yaml
# pricing.yaml - Updated per provider pricing pages
# Version-controlled, update monthly
version: "2026-03-31"

pricing:
  anthropic:
    claude-sonnet-4-20250514:
      input_per_mtok: 3.00
      output_per_mtok: 15.00
      cache_read_per_mtok: 0.30
      cache_creation_per_mtok: 3.75
    claude-opus-4-20250514:
      input_per_mtok: 15.00
      output_per_mtok: 75.00
      cache_read_per_mtok: 1.50
      cache_creation_per_mtok: 18.75

  openai:
    gpt-4.1:
      input_per_mtok: 2.00
      output_per_mtok: 8.00
      cached_input_per_mtok: 0.50
      reasoning_per_mtok: 8.00
    o3:
      input_per_mtok: 2.00
      output_per_mtok: 8.00
      reasoning_per_mtok: 8.00
      cached_input_per_mtok: 0.50

  google:
    gemini-2.5-pro:
      input_per_mtok: 1.25
      output_per_mtok: 10.00
      thinking_per_mtok: 1.25
      cached_input_per_mtok: 0.315

  # AMP, Kilo Code, Nous Research — deferred to future roadmap

# Alert: log warning when a model is encountered with no pricing entry
unknown_model_policy: "warn_and_log"
```

> [!IMPORTANT]
> **Pricing Drift**: Provider pricing changes frequently. The pricing file is version-controlled and should be updated monthly. The system logs a warning when a model is encountered that has no pricing entry.

---

## 8. Context Composition Analysis Engine

### 8.1 Input Decomposition Strategy

The **background worker** parses the raw HTTP request body (from staging files) and categorizes tokens by provider-specific JSON structure:

| Provider | System Prompt Field | Tool Defs Field | Messages Field | History Identification |
|:---|:---|:---|:---|:---|
| Anthropic | `body.system` | `body.tools[]` | `body.messages[]` | All messages except last user message |
| OpenAI | `body.messages[role=system]` | `body.tools[]` | `body.messages[]` | All messages except last user message |
| Google | `body.systemInstruction` | `body.tools[]` | `body.contents[]` | All contents except last user content |

### 8.2 Token Counting: Dual Tokenizer Strategy

Since provider APIs return aggregate `usage.input_tokens` but NOT per-component breakdowns, the system estimates component-level token counts using **two local tokenizers** with a reconciliation algorithm:

#### Tokenizer Selection per Provider
| Provider | Primary Tokenizer | Secondary Tokenizer |
|:---|:---|:---|
| Anthropic | `anthropic` Python SDK token counter | `tiktoken` (cl100k_base) |
| OpenAI | `tiktoken` (model-specific encoding) | `anthropic` SDK counter (cross-check) |
| Google | `tiktoken` (cl100k_base, with 1.1× modifier) | `anthropic` SDK counter (cross-check) |

> [!IMPORTANT]
> **Local Tokenizers Only**: The Gemini `count_tokens` API endpoint is explicitly NOT used. Making a remote API call from the worker would add latency and consume rate limits. Instead, Google token counts are approximated using `tiktoken` with a 1.1× scaling factor (Gemini tokenizers tend to produce ~10% more tokens than cl100k_base for equivalent text). The provider-reported aggregate `usage.promptTokenCount` from the response is used to calibrate the scaling factor over time.

#### Reconciliation Algorithm
```python
def estimate_tokens(text: str, primary_tokenizer, secondary_tokenizer) -> int:
    count_a = primary_tokenizer.count(text)
    count_b = secondary_tokenizer.count(text)
    
    smaller = min(count_a, count_b)
    larger = max(count_a, count_b)
    
    if smaller >= 0.95 * larger:
        # Counts are within 5% — use average
        return round((count_a + count_b) / 2)
    else:
        # Substantial divergence — use the LARGER value (conservative)
        return larger
```

> [!NOTE]
> **Rationale**: Using the larger value when counts diverge significantly ensures we don't underestimate token consumption, which is critical for accurate context window utilization tracking (metric EF-01) and cost calculation.

### 8.3 Content Versioning

The system tracks changes to static context components across requests:

```
content_versions/
  ├── system_prompts/
  │     ├── claude-code_v1_sha256abc.txt
  │     ├── claude-code_v2_sha256def.txt
  │     └── antigravity_v1_sha256ghi.txt
  ├── tool_definitions/
  │     ├── claude-code_tools_sha256jkl.json
  │     └── antigravity_tools_sha256mno.json
  └── diffs/
        └── claude-code_system_v1_to_v2.diff
```

**Logic**: On each request, hash the system prompt and tool definitions. If the hash differs from the last seen version, store the new version and compute a diff. This enables tracking how prompt changes affect quality/efficiency metrics.

---

## 9. Observability Platform: Arize Phoenix (Self-Hosted)

### 9.1 Justification: Why Phoenix Instead of Langfuse

| Criterion | Langfuse v3 | Arize Phoenix | Decision |
|:---|:---|:---|:---|
| **RAM requirement** | ≥16 GB (ClickHouse) | **~1 GB (SQLite)** | **Phoenix** — fits 4 GB budget |
| **Setup complexity** | Multi-container (3 services) | **Single container** | **Phoenix** — minimal ops |
| **Disk footprint** | ≥60 GB | **~5-10 GB (3 months)** | **Phoenix** — lean storage |
| **All features OSS** | Some gated | **100% open source** | **Phoenix** — no feature limits |
| **Trace visualization** | ★★★★★ | ★★★★ | Langfuse slightly richer |
| **Cost analytics** | ★★★★★ (built-in) | ★★★ (via custom attributes) | Langfuse better, but Phoenix sufficient |
| **Eval integration** | ★★★★ | ★★★★★ | **Phoenix** — stronger eval UI |
| **OTel native** | ★★★ | ★★★★★ (OpenInference) | **Phoenix** — OTel-first design |

**Verdict**: Arize Phoenix is the sole platform. It runs in a single container with SQLite, consuming ~1 GB RAM — well within the 4 GB budget.

### 9.2 Phoenix Docker Setup

```yaml
# docker-compose.yml — Arize Phoenix (lightweight)
version: "3.8"
services:
  phoenix:
    image: arizephoenix/phoenix:latest
    ports:
      - "127.0.0.1:6006:6006"    # Phoenix UI (loopback only)
      - "127.0.0.1:4317:4317"    # OTLP gRPC receiver (loopback only)
    environment:
      - PHOENIX_WORKING_DIR=/data
      - PHOENIX_PORT=6006
      - PHOENIX_GRPC_PORT=4317
      # Optional: enable auth to prevent accidental exposure
      # - PHOENIX_ENABLE_AUTH=true
      # - PHOENIX_SECRET=<random-secret>
    volumes:
      - phoenix_data:/data
    deploy:
      resources:
        limits:
          memory: 1536M   # 1.5 GB max
          cpus: "1.5"
    restart: unless-stopped

volumes:
  phoenix_data:
    driver: local
```

> [!NOTE]
> **Phoenix Ingestion Endpoint**: Phoenix accepts traces via **OTLP gRPC on port 4317** (standard OpenTelemetry Collector protocol) or **OTLP HTTP on port 6006** at `/v1/traces`. The background worker exports spans using the `opentelemetry-exporter-otlp` package configured to `http://127.0.0.1:4317`. Do NOT use a custom REST endpoint like `/api/traces` — that does not exist in Phoenix.

> [!IMPORTANT]
> **Loopback-Only Binding**: All Phoenix ports are bound to `127.0.0.1` to prevent network exposure. This is a local-only deployment — no external access to the trace UI or OTLP receiver should be possible.

#### Per-Agent and Per-Session Capture Toggle

Because the system archives full prompts, code, tool outputs, and file contents, operators need the ability to selectively disable capture at the agent or session level. This is controlled via `~/.observability/capture_config.yaml`:

```yaml
# capture_config.yaml — controls what is captured and archived
capture:
  # Global kill switch — disables all capture (proxy still forwards traffic)
  enabled: true

  # Per-agent overrides — agents not listed inherit the global setting
  agents:
    claude-code:
      enabled: true
      log_request_body: true     # Archive full request body in JSONL
      log_response_body: true    # Archive full response body in JSONL
    antigravity:
      enabled: true
      log_request_body: true
      log_response_body: true
    codex:
      enabled: false             # Disable all capture for this agent
    # Agents not listed here: capture follows global `enabled` setting

  # Per-session overrides — evaluated AFTER agent-level rules
  # Sessions matching these prefixes are excluded from capture entirely
  excluded_session_prefixes:
    - "debug-"        # Convention: sessions prefixed with "debug-" are not archived
    - "scratch-"      # Convention: scratch sessions are not archived
```

**Behavior**: The proxy addon checks capture config on each request after extracting the agent name and session ID from proxy-auth. If capture is disabled for that agent or session, the proxy still **forwards the request** (fail-open) but does NOT write staging files — the request passes through invisibly. The capture config file is hot-reloaded on change (watched via `watchdog` or polled every 30 seconds) so toggles take effect without restarting the proxy.

### 9.3 Resource Budget

| Component | RAM | CPU | Disk (3 months) |
|:---|:---|:---|:---|
| mitmproxy + thin addon | ~150 MB | 0.3 cores | ~0 MB |
| Background worker | ~200 MB | 0.5 cores | ~1 GB (staging) |
| Arize Phoenix | ~1-1.5 GB | 1.5 cores | ~5-10 GB |
| JSONL Archive | ~0 MB | ~0 | ~10-20 GB |
| Content Versions | ~0 MB | ~0 | ~1 GB |
| **Total** | **~1.85 GB** | **2.3 cores** | **~32 GB** |

✅ **Well within the 4 GB RAM constraint** with headroom for growth.

---

## 10. Offline Evaluation Pipeline

### 10.1 Architecture

```
JSONL Archive ──▶ Sampler ──▶ Evaluator ──▶ Score Store ──▶ Phoenix
                                 │
                          LLM-as-Judge
                     (via existing OAuth subscription)
```

### 10.2 Judge Model: OAuth-Authenticated Access

> [!IMPORTANT]
> **Zero-Cost Constraint**: The evaluation pipeline uses the same OAuth-authenticated provider access that the AI agents use. No separate API keys or pay-per-use billing is required.

> [!WARNING]
> **OAuth Token Reuse (Open Dependency)**: This is a **partially solved design decision**, not a fully automated feature. The eval pipeline depends on OAuth tokens that agents have already obtained. Key considerations:
>
> - **Token Source**: The eval pipeline reuses cached OAuth tokens from `~/.config/<agent>/` credential stores. It does NOT perform its own OAuth browser flow.
> - **Token Refresh**: If the cached token has expired, the eval pipeline must gracefully fail and log the error. It does NOT auto-refresh tokens (that requires user interaction via browser-based OAuth).
> - **Fallback Behavior**: If no valid token is available, the eval batch is skipped with a warning. The cursor is NOT advanced, so the same records will be retried on the next run.
> - **Multi-Provider**: The eval pipeline attempts tokens from all three providers in priority order (cheapest first: Gemini Flash, Claude Haiku, OpenAI GPT-4.1-mini). If one provider's token is expired but another's works, use the working one.
> - **Phase 1 Expectation**: In practice, the user runs agents daily, refreshing OAuth tokens as a side effect. Token expiry is primarily a concern after extended idle periods (weekends, vacations).

**Implementation**:
1. The eval pipeline routes its LLM judge calls through the **same mitmproxy** as agent traffic.
2. It authenticates using cached OAuth tokens from agent credential stores.
3. Judge calls use proxy-auth identity `eval-pipeline:<eval-batch-id>` so they are identifiable in traces and excluded from agent-level metrics.
4. The existing provider subscription (e.g., Claude Pro, Gemini Advanced) is used at no incremental cost.

**Judge Model Selection**: Use whichever model is available through the active OAuth subscription. Prioritize by cost: Gemini Flash (cheapest), Claude Haiku, GPT-4.1-mini.

### 10.3 Evaluation Scheduler & State Management

#### Scheduler Resilience

The evaluation scheduler is designed for a laptop that sleeps, shuts down, and may not run during traditional cron windows:

- **Trigger**: Elapsed-time based, NOT cron. The scheduler checks on terminal startup (via shell hook) or when the background worker starts: "has 24 hours elapsed since last successful eval run?" If yes, trigger eval.
- **Implementation**: A shell hook in `~/.zshrc` runs `python ~/.observability/eval_trigger.py` which checks the timestamp of the last eval run.
- **Graceful Handling**: If the laptop was asleep for 3 days, the scheduler runs ONE eval covering all accumulated records, not 3 separate daily evals.

#### Evaluation State Tracking

The eval pipeline maintains persistent state in a local SQLite database to prevent re-evaluation and handle interruptions:

```
~/.observability/eval_state.db
  Table: eval_cursor
    - log_file: TEXT (e.g., "agent_logs_2026-03-31.jsonl")
    - last_line_offset: INTEGER (byte offset of last processed line)
    - last_eval_timestamp: TIMESTAMP
    - status: TEXT ("completed", "interrupted", "failed")
```

- **Cursor-based**: The pipeline reads from where it left off. If interrupted mid-batch, it resumes from the last committed cursor position.
- **Idempotency**: Each JSONL record has a unique `trace_id` + `span_id`. Duplicate evaluations are prevented by checking the eval results table.

#### Schedule

| Evaluation | Frequency | Sample Size | Notes |
|:---|:---|:---|:---|
| Trajectory Relevancy (QA-04) | Daily (elapsed) | 10% of sessions | Sampled randomly |
| Faithfulness (QA-05) | Daily (elapsed) | 20% of tool-using turns | Focus on tool output grounding |
| Output Completeness (QA-06) | Daily (elapsed) | 10% of sessions | Binary + confidence score |
| Hallucination Detection (QA-07) | Daily (elapsed) | All turns with file path/code refs | Pattern-based pre-filter + LLM judge |
| Tool Discovery Accuracy (EF-05) | Weekly (elapsed) | All tool-using turns | Compare selected vs. optimal tool |
| Token-to-Insight Ratio (EF-07) | Weekly (elapsed) | 10% of sessions | Output complexity scoring |



### 10.4 Evaluation Rubrics

#### Trajectory Relevancy (QA-04)
```markdown
You are evaluating the decision path of an AI coding agent.

Given:
- The user's original objective
- The sequence of actions taken (tool calls, LLM responses)
- The final output

Score the trajectory on a 1-5 scale:
5: Every step directly contributes to the objective. No wasted actions.
4: Minor deviations but overall efficient path.
3: Some unnecessary steps or redundant reasoning, but objective was met.
2: Significant wasted effort. Multiple unnecessary tool calls or reasoning loops.
1: Agent went completely off-track or failed to address the objective.

Provide your reasoning BEFORE the score.
```

#### Faithfulness (QA-05)
```markdown
You are evaluating whether an AI agent's response is grounded in the 
context it was given (tool outputs, file contents, search results).

Given:
- The context provided to the agent (tool outputs, retrieved content)
- The agent's response to the user

Score faithfulness on a 1-5 scale:
5: Every claim is directly supported by the provided context.
4: All major claims supported; minor inferences are reasonable.
3: Some claims lack direct support but are plausible.
2: Multiple claims are unsupported or contradict the context.
1: Response fabricates information not present in any context.

Provide your reasoning BEFORE the score.
```

---

## 11. Provider-Specific Parsing Rules

Each provider's API response has a different schema. The **background worker** normalizes these into the unified `llm.*` schema:

### 11.1 Anthropic
```python
# Response body: response.usage
{
  "input_tokens": 12540,
  "output_tokens": 1834,
  "cache_read_input_tokens": 8200,    # → llm.usage.cache_read_tokens
  "cache_creation_input_tokens": 0     # → llm.usage.cache_creation_tokens
}
# Anthropic does NOT separate thinking tokens — all included in output_tokens
```

### 11.2 OpenAI
```python
# Response body: response.usage
{
  "prompt_tokens": 15000,             # → llm.usage.input_tokens
  "completion_tokens": 2500,           # → llm.usage.output_tokens
  "prompt_tokens_details": {
    "cached_tokens": 10000             # → llm.usage.cache_read_tokens
  },
  "completion_tokens_details": {
    "reasoning_tokens": 1200           # → llm.usage.reasoning_tokens
  }
}
```

### 11.3 Google (Gemini)
```python
# Response body: response.usageMetadata
{
  "promptTokenCount": 8000,            # → llm.usage.input_tokens
  "candidatesTokenCount": 1500,        # → llm.usage.output_tokens
  "thoughtsTokenCount": 800,           # → llm.usage.reasoning_tokens
  "cachedContentTokenCount": 5000      # → llm.usage.cache_read_tokens
}
```

---

## 12. Functional Requirements: Proxy vs. Worker Split

### 12.1 Proxy Addon Functions (Thin — Critical Path)

The proxy addon runs in the mitmproxy event loop. It MUST be fast and non-blocking. No computation on request/response content.

| # | Function | Description |
|:---|:---|:---|
| P1 | **Request Interception** | Detect outbound HTTPS requests to known provider endpoints (URL match) |
| P2 | **Agent Identity Extraction** | Extract agent name + session ID from proxy-auth credentials |
| P3 | **Proxy-Auth Stripping** | Strip `Proxy-Authorization` header before forwarding to provider |
| P4 | **Timestamp Recording** | Record request received, forwarded, first response byte, last response byte timestamps |
| P5 | **Request Body Staging** | Stream request body to staging file on disk (no parsing) |
| P6 | **SSE Detection & Disk Streaming** | Detect `text/event-stream` responses; stream chunks to staging temp file on disk |
| P7 | **Response Body Staging** | Write non-streaming response body to staging file |
| P8 | **Staging File Finalization** | Rename `.tmp` to `.ready` when request/response cycle is complete |
| P9 | **Non-LLM Pass-through** | Forward non-LLM traffic (OAuth flows, non-API requests) without any processing |

### 12.2 Background Worker Functions (Heavy — Async)

The worker runs as a separate Python process, polling the staging directory. All heavy computation happens here.

| # | Function | Description |
|:---|:---|:---|
| W1 | **Staging Polling** | Watch `~/.observability/staging/` for `.ready` files |
| W2 | **Provider Detection** | Identify provider (Anthropic/OpenAI/Google) from request URL in staging file |
| W3 | **Schema Normalization** | Map provider-specific response fields to unified `llm.*` attribute schema |
| W4 | **Token Usage Extraction** | Parse usage metadata from response body |
| W5 | **Context Decomposition** | Parse request body: system prompt, tools, history, user message |
| W6 | **Component Token Estimation** | Run dual local tokenizer with reconciliation algorithm |
| W7 | **Cost Calculation** | Apply pricing rules from `pricing.yaml` |
| W8 | **Content Hashing** | SHA-256 hash system prompts + tool defs for version tracking |
| W9 | **Credential & Secret Scrubbing** | Two-layer scrubbing: header removal + regex-based body scanner |
| W10 | **JSONL Writing** | Write scrubbed records to daily-rotated JSONL files via async queue |
| W11 | **OTLP Span Emission** | Create and export spans to Arize Phoenix via OTLP gRPC on :4317 |
| W12 | **Tool Span Inference** | Infer tool execution spans from `tool_use` / `tool_result` blocks |
| W13 | **Skill Discovery Inference** | Detect SKILL.md references and KI lookups in message content |
| W14 | **Process Tree Resolution** | PID mapping and `pgrep` for sub-agent parent resolution (fallback) |
| W15 | **Alert Evaluation** | Evaluate alert conditions (tool def budget, context window, cost anomaly) |
| W16 | **Staging Cleanup** | Delete processed staging files after successful worker completion |

### 12.3 Hot-Path Failure Model

Because every agent call goes through the proxy, failures must be handled gracefully to avoid disrupting agent workflows.

| Scenario | Behavior | Rationale |
|:---|:---|:---|
| **Staging directory full** | Proxy logs warning and drops telemetry for that request. Agent traffic continues unimpeded. | Fail-open: never block agent work for observability. |
| **Staging write fails** | Proxy retries once with a 100ms delay. If still fails, drops the request and logs error. | Bounded retry to avoid latency accumulation. |
| **Worker process crashed** | Proxy continues operating normally. Staging files accumulate on disk until worker restarts. | Decoupled: proxy has no dependency on worker liveness. |
| **Phoenix unreachable** | Worker buffers up to 1000 spans in memory. If buffer full, writes overflow spans to a fallback JSONL file. Retries Phoenix connection every 30 seconds. | Bounded queue prevents OOM. Fallback file ensures no data loss. |
| **Worker processing error** | Worker moves the staging file to `~/.observability/staging/errors/` with an error log. Does not delete. | Preserves data for manual inspection. |
| **Disk full (global)** | Proxy enters degraded mode: forwards traffic but emits no staging files. Worker pauses processing. Both log critical warnings. | System degrades gracefully rather than crashing. |

> [!IMPORTANT]
> **Fail-Open Principle**: The proxy NEVER blocks or fails agent requests due to observability issues. If staging, the worker, or Phoenix are unavailable, agent traffic continues normally. Telemetry is best-effort.

---

## 13. Data Retention & Lifecycle

| Data Type | Storage | Retention | Cleanup |
|:---|:---|:---|:---|
| Phoenix spans/traces | SQLite (Docker volume) | 90 days | Phoenix retention config |
| Raw JSONL logs | Local disk (`~/.observability/logs/`) | 90 days | Daily cleanup script: `find ~/.observability/logs/ -name '*.jsonl' -mtime +90 -delete` |
| Staging files | Local disk (`~/.observability/staging/`) | Transient (hours) | Worker deletes after processing; cleanup script removes files older than 24h |
| Content versions (prompts, tools) | Local disk (`~/.observability/content_versions/`) | 180 days | Cleanup script: delete versions older than 180 days, keep latest version per agent |
| Evaluation scores | Phoenix (via annotation API) | 90 days | Matches span retention |
| Evaluation state | SQLite (`~/.observability/eval_state.db`) | Indefinite | Manual cleanup (small file) |
| Aggregated daily metrics | JSONL summary files | 1 year | Compressed rollups |

---

## 14. Success Criteria

### 14.1 Phase 1 (MVP) — Target: 4 weeks

- [ ] mitmproxy thin addon captures Anthropic + OpenAI + Google traffic (full telemetry)
- [ ] AMP, Kilo Code, Nous Research traffic captured as raw pass-through (JSONL only)
- [ ] Background worker processes staging files and emits OTLP spans to Phoenix
- [ ] Proxy-auth based agent identification working for all 7 agents
- [ ] Proxy-auth inject-and-strip mechanism verified (providers never see proxy-auth headers)
- [ ] SSE streams handled via disk-based streaming (no in-memory buffering)
- [ ] Daily-rotated JSONL archive stores full scrubbed request/response bodies
- [ ] Two-layer credential + secret scrubbing pipeline operational
- [ ] Context composition breakdown per request (dual local tokenizer, no remote API calls)
- [ ] Token usage + cost calculation per request
- [ ] Arize Phoenix running on loopback-only ports with OTLP gRPC ingestion
- [ ] Basic dashboards: cost/day, tokens/session, cache hit rate
- [ ] Tool definition budget alert (>30% of context)
- [ ] Hot-path failure model validated: proxy continues when worker/Phoenix are down
- [ ] CA certificate trust validated for all 7 agents (no cert pinning issues)

### 14.2 Phase 2 (Quality) — Target: +3 weeks

- [ ] Offline evaluation pipeline running on elapsed-time trigger (not cron)
- [ ] OAuth token reuse from agent credential stores validated
- [ ] Evaluation state tracking via SQLite cursor (handles interruptions)
- [ ] QA metrics (faithfulness, trajectory relevancy) calculated
- [ ] Tool call success/failure tracking (inferred from tool_use/tool_result analysis)
- [ ] Content versioning for system prompts and tool defs
- [ ] Efficiency metrics: context bloat, tool definition overhead
- [ ] Skill discovery tracking (inferred from SKILL.md references)

### 14.3 Phase 3 (Scale) — Target: +3 weeks

- [ ] Normalized parsing rules for remaining providers (AMP, Kilo Code, Nous Research)
- [ ] Sub-agent correlation and hierarchy visualization
- [ ] Automated agent runs telemetry at scale (hundreds/day)
- [ ] Data retention automation (90-day TTL, 180-day content versions)
- [ ] Per-agent, per-provider comparison dashboards

---

## 15. Future Roadmap: Analytics & Visualization

### 15.1 Advanced Analytics (Post-MVP)

| Capability | Description | Estimated Phase |
|:---|:---|:---|
| **Loop & Redundancy Detection** | Action fingerprinting (hash tool name + normalized params). Detect when the same action repeats in a sliding window. Configurable thresholds. | Phase 4 |
| **Decision Path DAG** | Visualize the full decision tree as a directed acyclic graph. Nodes = LLM calls + tool executions. Edges = causal dependencies. Highlight divergence points. | Phase 4 |
| **Comparative Agent Benchmarking** | Same task → multiple agents. Compare quality scores, token spend, latency, and tool efficiency side-by-side. | Phase 5 |
| **Prompt A/B Testing Framework** | Version-controlled prompt variants with automated eval scoring. Statistical significance testing on quality/efficiency deltas. | Phase 5 |
| **Embedding Drift Detection** | Track vector embedding distributions over time. Alert when retrieval quality degrades due to corpus changes. | Phase 6 |
| **Multi-Agent Orchestration Tracing** | Full trace visibility across orchestrated multi-agent workflows (e.g., CrewAI, AutoGen). Span linking across agent boundaries. | Phase 6 |

### 15.2 Visualization Capabilities Roadmap

```mermaid
gantt
    title Visualization Capability Roadmap
    dateFormat  YYYY-MM
    axisFormat  %b %Y

    section Phase 1 - MVP
    Phoenix Trace Explorer            :done, p1a, 2026-04, 2026-05
    Cost/Token Time-Series Charts     :done, p1b, 2026-04, 2026-05
    Context Composition Pie Charts    :done, p1c, 2026-04, 2026-05

    section Phase 2 - Quality
    Eval Score Trend Lines            :p2a, 2026-05, 2026-06
    Tool Success/Failure Heatmaps     :p2b, 2026-05, 2026-06
    Content Version Diff Viewer       :p2c, 2026-05, 2026-06

    section Phase 3 - Scale
    Agent Comparison Dashboards       :p3a, 2026-06, 2026-07
    Provider Cost Breakdown Sankey    :p3b, 2026-06, 2026-07
    Retention Lifecycle Dashboard     :p3c, 2026-06, 2026-07

    section Phase 4 - Intelligence
    Decision Path DAG Renderer        :p4a, 2026-07, 2026-09
    Loop Detection Alert Panel        :p4b, 2026-07, 2026-08
    Anomaly Detection Dashboard       :p4c, 2026-08, 2026-09

    section Phase 5 - Optimization
    Prompt A/B Test Results UI        :p5a, 2026-09, 2026-10
    Agent Benchmark Leaderboard       :p5b, 2026-09, 2026-10
    ROI Calculator (cost vs quality)  :p5c, 2026-09, 2026-10
```

### 15.3 Future Dashboard Specifications

#### Dashboard 1: Agent Intelligence Scorecard
A single-page view per agent showing:
- Quality score trend (30-day rolling average)
- Efficiency index (tokens consumed vs. task complexity)
- Cost efficiency rank relative to other agents
- Top 5 failure modes (most common error patterns)

#### Dashboard 2: Context Economy Monitor
Real-time view of context window "spend":
- Stacked area chart: system prompt, tool defs, history, user message, dynamic state
- Alert thresholds overlaid (e.g., tool definitions >30%)
- Historical trend of context composition ratios per agent

#### Dashboard 3: Provider Economics
Cost attribution drilldown:
- Sankey diagram: Provider → Model → Agent → Task Category
- Cache efficiency scatter plot (cache hit rate vs. cost per session)
- Token type breakdown (input, output, reasoning, cached) per provider
- Monthly burn rate projection with trend extrapolation

#### Dashboard 4: Decision Path Explorer (Phase 4+)
Interactive trace viewer:
- DAG rendering of agent reasoning steps
- Color-coded nodes by status (success, error, retry)
- Expandable nodes showing full LLM request/response
- Side-by-side comparison of two runs of the same task
- Loop/cycle highlighting with configurable detection sensitivity

#### Dashboard 5: Skill & Tool Discovery Analytics
Efficiency metrics for the agent's ability to find and use the right tools/skills:
- Tool selection accuracy over time (was the first tool choice correct?)
- Skill discovery funnel: available → evaluated → selected → successful
- Tool category usage distribution (file_ops, search, browser, command, etc.)
- Correlation matrix: tool discovery time vs. task complexity

---

## 16. Resolved Decisions

| # | Decision | Resolution |
|:---|:---|:---|
| D1 | Resource allocation | Mac has 16 GB RAM total. Observability stack must fit ≤4 GB. **Switched from Langfuse (16 GB) to Arize Phoenix (~1.5 GB)**. Total stack: ~1.85 GB. |
| D2 | Agent identification | **Proxy-auth URL encoding** (`http://agentname:sessionid@127.0.0.1:18080`). Survives env var scrubbing by sandbox environments. Inject-and-strip mechanism removes `Proxy-Authorization` before forwarding. |
| D3 | Certificate trust | **Agent-specific env vars only** (no system-wide CA trust). `NODE_EXTRA_CA_CERTS`, `REQUESTS_CA_BUNDLE`, `SSL_CERT_FILE` per agent. Cert pinning fallback: OpenLLMetry SDK instrumentation. |
| D4 | Evaluation cost | **Zero cost (partially solved)**. Eval pipeline reuses cached OAuth tokens from agent credential stores. Token expiry, refresh failure, and multi-provider fallback remain **open dependencies** — see Section 10.2 for the full risk description and mitigation strategy. |
| D5 | Tool definition budget | Alert when tool definitions exceed **30%** of total context window tokens. Configurable threshold. |
| D6 | Proxy architecture | **"Dumb and fast" proxy + async background worker**. Proxy only captures and queues; worker does all heavy processing (tokenization, hashing, cost, scrubbing, OTLP export). |
| D7 | Phoenix ingestion | **OTLP gRPC on port 4317** (standard OpenTelemetry protocol). Not a custom REST endpoint. |
| D8 | Proxy port | **Port 18080** (not 8080, which conflicts with common dev servers). |
| D9 | SSE handling | **Disk-based streaming**. Chunks are streamed to staging temp files, not buffered in memory. |
| D10 | Token estimation (Google) | **Local `tiktoken` with 1.1× modifier**. Gemini's `count_tokens` remote API is NOT used (would add latency and consume rate limits). |
| D11 | Secret scrubbing | **Two-layer pipeline**: header scrubbing (OAuth, API keys) + regex-based body scanner (secrets, PII, private keys). |
| D12 | Failure model | **Fail-open**. Proxy never blocks agent requests for observability reasons. Bounded queues, retry/drop policies, disk-full degradation. |
| D13 | Eval scheduler | **Elapsed-time trigger** (not cron). Handles laptop sleep/shutdown. SQLite cursor tracks evaluation progress. |
| D14 | Telemetry scope | **Proxy-observable** (not white-box). LLM req/res are directly observed; tool/skill/reasoning spans are inferred from content analysis. |
| D15 | Loopback binding | All services (proxy, Phoenix UI, OTLP receiver) bind to **127.0.0.1 only**. No network exposure. |

---

## Verification Plan

### Automated Tests
- Unit tests for each provider's response parser (Anthropic, OpenAI, Google schemas)
- Unit tests for dual-tokenizer reconciliation algorithm (test 95% threshold logic)
- Unit tests for proxy-auth extraction and stripping (verify provider never receives `Proxy-Authorization`)
- Unit tests for two-layer secret scrubbing pipeline (test each regex pattern)
- Integration test: proxy addon → staging file → worker → JSONL write → verify schema compliance
- Integration test: worker → OTLP gRPC export → Phoenix span appears in UI
- Integration test: SSE disk-based streaming → staging file → worker reassembly → complete response
- End-to-end test: run a Claude Code session through proxy → verify Phoenix trace appears
- Cost calculation accuracy test: compare computed costs against known inputs
- Hot-path failure test: kill worker process → verify proxy continues forwarding traffic
- Hot-path failure test: stop Phoenix → verify worker buffers spans and writes fallback JSONL
- Daily rotation test: verify JSONL file rolls over at midnight
- Eval state cursor test: interrupt eval mid-batch → resume → verify no duplicate evaluations

### Manual Verification
- Visual inspection of Phoenix trace hierarchy for a multi-tool agent session
- Confirm context composition percentages match manual token counting
- Verify SSE disk-based streaming produces complete response content (no data loss)
- Confirm OAuth flows work uninterrupted through the proxy for each of the 7 agents
- Verify proxy-auth credentials are stripped from provider-bound requests (packet inspection)
- Verify sub-agent spans correctly correlate under parent session in Phoenix UI
- Confirm tool definition budget alert triggers when threshold is exceeded
- Verify secret scrubbing catches embedded API keys and `.env` contents in logged bodies
- Confirm proxy port 18080 does not conflict with any running dev servers
- Verify loopback-only binding: Phoenix UI and OTLP port are not accessible from network
