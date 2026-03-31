# Problem Statement: AI Agent Observability for a Solo Developer Workstation

## The Situation

You are a solo developer who uses multiple AI coding agents — Claude Code, Antigravity, Codex, Gemini CLI, AMP Code, Kilo Code, and Hermes — simultaneously on a single workstation to build software. These agents call LLM providers (Anthropic, OpenAI, Google, and others) dozens of times per day on your behalf, consuming tokens, making tool calls, reading your codebase, and generating code. You pay for this through provider subscriptions with OAuth-based authentication.

Today, all of this happens as a **black box**. You launch an agent, give it a task, and get a result — but you have no systematic visibility into what happened in between.

---

## The Problems

### 1. You cannot answer "where is my money going?"

Each agent session triggers a chain of LLM API calls — sometimes dozens for a single task. You have no breakdown of:

- How much each agent costs you per session, per day, or per task.
- Which model variants are being used and at what price.
- Whether prompt caching is working (and how much it's saving you).
- Whether thinking/reasoning tokens are a significant overhead or a worthwhile investment.
- How your daily spend is trending over time.

Without this data, you cannot make informed decisions about which agents to use for which tasks, or whether to adjust configurations to reduce cost.

### 2. You don't know if agents are using your context window efficiently

LLM context windows are finite and expensive. Every token sent as input is a token you pay for. But you have no visibility into:

- How much of the context window is consumed by **system prompts** versus **tool definitions** versus **conversation history** versus your **actual message**.
- Whether tool definitions are bloated (some agents ship 70+ tool schemas, consuming thousands of tokens per request).
- Whether conversation history is growing unchecked across turns, causing context bloat.
- Whether agents are effectively using prompt caching, or re-sending the same static content uncached on every request.
- Whether your custom skills, knowledge items, and rules are actually being discovered and used — or ignored.

This is the "context economy" problem: you're paying for every token in the window, but you have no accounting of what's filling it.

### 3. You cannot tell if agents are making good decisions

When an agent chooses a tool, constructs parameters, or decides on a multi-step plan, you have no way to systematically evaluate:

- **Tool call quality**: Are tool calls succeeding or failing? Are parameters valid? Is the agent retrying the same failed call?
- **Decision path relevance**: Is the agent taking an efficient path to your goal, or wandering through unnecessary steps?
- **Faithfulness**: Is the agent's output actually grounded in the context it retrieved, or is it hallucinating file paths, API signatures, or facts?
- **Completeness**: Did the agent fully satisfy your objective, or did it stop short?
- **Hallucination rate**: How often does the agent fabricate information that isn't present in your codebase?

Without quality metrics, you're relying on gut feel to assess agent performance — and you might not even notice subtle quality degradation until it causes real problems.

### 4. When something goes wrong, you cannot diagnose why

Agent failures manifest as wrong code, missed requirements, or wasted loops — but tracing the root cause is manual and painful:

- You have no request-level latency data (is the agent slow because of the provider, the network, or the agent itself?).
- You cannot inspect the exact prompts and responses that led to a bad outcome.
- You cannot see the decision path: which tools were called, in what order, with what parameters, and what results came back.
- Error rates per agent and per provider are invisible — you don't know if one provider is consistently flakier than another.
- When an agent enters a loop (repeating the same action), you only notice after it has already wasted tokens and time.

### 5. You have no way to compare agents against each other

You use multiple agents, but you have no objective basis for deciding which agent is best for which type of task:

- No side-by-side quality comparison on equivalent tasks.
- No cost-efficiency ranking (which agent achieves the best result per dollar?).
- No latency or throughput comparison across providers.
- No data on which agent discovers and uses your custom skills most effectively.

Agent selection is currently based on intuition, not evidence.

### 6. The agents use OAuth — which breaks traditional monitoring approaches

Unlike typical API integrations where you control the API keys, these agents authenticate with providers via **OAuth 2.0 browser flows**. This means:

- You cannot use an API gateway (like LiteLLM) that expects to own the credentials.
- You cannot use provider-side usage dashboards because OAuth-based subscriptions often don't offer granular per-request analytics.
- You cannot instrument the agents' source code because they are third-party binaries that you don't control.
- Standard observability tools (Datadog, New Relic) are designed for server-side services, not for intercepting client-side LLM calls from local CLI tools.

### 7. Privacy and security are not negotiable

You are sending your proprietary codebase, secrets, environment variables, and business logic through these agents to LLM providers. Any observability system that captures this traffic must:

- Never expose captured data to the network — everything stays local.
- Scrub credentials, API keys, and secrets that agents embed in prompts (not just HTTP headers — agents routinely pass `.env` file contents and API keys inside message bodies).
- Allow you to selectively disable capture for specific agents or sessions.
- Not add meaningful latency to your agent workflows.

---

## What the Solution Must Deliver

Given these problems, the observability system must provide:

| Concern | What You Need |
|:---|:---|
| **Cost visibility** | Per-agent, per-session, per-model cost tracking. Cache hit rates and savings. Daily spend trends. Anomaly alerts. |
| **Context economy** | Per-request breakdown of what fills the context window. Tool definition overhead alerts. Context bloat detection. Prompt/tool version tracking over time. |
| **Quality assurance** | Tool call success rates. LLM-judged trajectory relevancy, faithfulness, completeness, and hallucination detection — all computed offline using your existing provider subscriptions at zero incremental cost. |
| **Debugging** | Full request/response archive with searchable traces. Latency decomposition (proxy overhead, TTFT, remote duration). Error rate tracking per agent and provider. Finish reason distributions. |
| **Agent comparison** | Unified data model across all agents and providers, enabling cross-agent and cross-provider analytics on cost, quality, and efficiency. |
| **Privacy-first design** | Local-only deployment (loopback binding). Two-layer secret scrubbing (headers + body regex). Per-agent and per-session capture toggles. No data leaves the workstation. |
| **Non-invasive architecture** | Works with OAuth authentication without modifying agents. Adds no meaningful latency (fail-open proxy with async processing). Operates within 4 GB RAM on a 16 GB machine alongside the agents themselves. |

---

## Constraints

| Constraint | Detail |
|:---|:---|
| **Hardware** | 16 GB total RAM. Observability stack ≤ 4 GB. |
| **Cost** | Zero incremental cost. Evaluation pipeline reuses existing OAuth subscriptions. |
| **Privacy** | All data stays on the local machine. No cloud telemetry services. |
| **Latency** | The proxy must never perceptibly slow down agent workflows. |
| **Durability** | Laptop sleeps, shuts down, and may be idle for days. The system must handle this gracefully. |
| **Scope** | Proxy-observable telemetry only. No agent source code modification. Tool and skill spans are inferred, not directly instrumented. |
