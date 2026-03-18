# Multi-Agent Insurance Chatbot — Design Document

## 1. Problem Statement

Design a multi-agent system that powers a customer-facing insurance chatbot. The system must:

- Answer customer queries about policies, claims, billing, and general insurance topics
- Support both **authenticated (logged-in)** and **anonymous (logged-out)** users
- Leverage short-term and long-term memory for personalized experiences
- Be read-only — no action-taking (e.g., filing claims, making payments)
- Gracefully handle edge cases: off-topic queries, low-confidence answers, and compliance-sensitive responses

The system is **tech-stack agnostic** — the design should be implementable in any language/framework.

---

## 2. Why Multi-Agent?

A single monolithic LLM call could technically answer questions, but it leads to:

- **Bloated system prompts** — one prompt trying to cover policy, claims, FAQ, and routing logic
- **Poor specialization** — the model can't be tuned/optimized per domain
- **Difficult guardrails** — compliance rules differ between claims vs. general FAQ
- **No observability** — you can't tell which "capability" failed when things go wrong

A multi-agent architecture gives us **separation of concerns**, **independent tunability**, and **clear failure boundaries**.

---

## 3. System Architecture

### 3.1 High-Level Component Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Client (Web/Mobile)                         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     API Gateway / Chat Interface                     │
│              (Auth check, session management, rate limiting)         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Input Guardrails   │
                    │  (PII, injection,    │
                    │   off-topic filter)  │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │    Orchestrator /    │
                    │    Router Agent      │◄────── Short-term Memory
                    │  (intent + routing)  │◄────── Long-term Memory (if authed)
                    └──┬───────┬───────┬──┘
                       │       │       │
              ┌────────▼┐  ┌──▼────┐  ┌▼────────┐
              │ Policy   │  │Claims │  │ FAQ /   │
              │ Agent    │  │Agent  │  │ General │
              │(+billing)│  │       │  │ Agent   │
              └────┬─────┘  └──┬────┘  └──┬──────┘
                   │           │          │
          ┌────────▼───────────▼──────────▼───────┐
          │          Knowledge / Data Layer         │
          │  ┌─────────────┐  ┌──────────────────┐ │
          │  │ Knowledge    │  │ Customer Data    │ │
          │  │ Base (RAG)   │  │ Layer (authed    │ │
          │  │ (all users)  │  │ users only)      │ │
          │  └─────────────┘  └──────────────────┘ │
          └────────────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Output Guardrails   │
                    │ (hallucination check,│
                    │  disclaimers,        │
                    │  compliance filter)  │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Response to User   │
                    └─────────────────────┘

        ┌───────────────────────────────────────────┐
        │           Cross-cutting Concerns           │
        │  ┌─────────────┐  ┌─────────────────────┐ │
        │  │ Observability│  │  Human Handoff      │ │
        │  │ (traces,logs,│  │  (escalation to     │ │
        │  │  metrics)    │  │   live agent)       │ │
        │  └─────────────┘  └─────────────────────┘ │
        └───────────────────────────────────────────┘
```

### 3.2 Request Lifecycle

A user query goes through the following steps:

1. **Client** sends a message via the chat interface
2. **API Gateway** authenticates the request, creates/resumes a session
3. **Input Guardrails** scan the query for prompt injection, PII leakage, and off-topic content
4. **Orchestrator/Router Agent** receives the sanitized query along with:
   - Auth state (logged-in or anonymous)
   - Short-term memory (recent conversation context)
   - Long-term memory (if authenticated — past interactions, user profile summary)
5. Router **classifies intent** and **selects the specialist agent** (Policy, Claims, or FAQ)
6. **Specialist Agent** processes the query using:
   - Knowledge Base (RAG) for document-grounded answers
   - Customer Data Layer (if authenticated) for personalized answers
7. **Output Guardrails** validate the response for hallucinations, inject disclaimers, and apply compliance filters
8. **Response** is returned to the user
9. If at any point confidence is low or the user expresses frustration, **Human Handoff** is triggered

---

## 4. Component Deep Dives

### 4.1 Orchestrator / Router Agent

**Role:** The entry point for every query. Classifies user intent and routes to the appropriate specialist agent.

**How it works:**
- Takes the user query + conversation context (short-term memory) + user profile (long-term memory, if available)
- Classifies intent into categories: `policy`, `claims`, `billing`, `general/faq`, `out-of-scope`, `needs-human`
- Routes to the appropriate specialist agent
- If auth is required but user is logged out, responds with a prompt to log in (e.g., "I can look up your specific policy details if you log in")

**Auth-aware routing logic:**

| Intent | Logged Out | Logged In |
|--------|-----------|-----------|
| Generic policy question | FAQ Agent (generic KB) | Policy Agent (generic KB) |
| My policy details | Prompt to log in | Policy Agent (customer data) |
| Claims status | Prompt to log in | Claims Agent (customer data) |
| General insurance question | FAQ Agent | FAQ Agent |
| Out-of-scope | Decline gracefully | Decline gracefully |
| Frustrated / confused | Human handoff | Human handoff |

**Tradeoff — LLM-based routing vs. classifier model:**
- LLM-based: More flexible, understands nuance, but slower and costlier per request
- Trained classifier (e.g., fine-tuned small model): Faster, cheaper, but needs labeled training data and handles edge cases poorly
- **Recommendation:** Start with LLM-based routing (simpler to build, easier to iterate). Move to a hybrid (classifier for common intents + LLM fallback for ambiguous ones) once you have traffic data.

### 4.2 Specialist Agents

Each specialist agent has:
- A focused **system prompt** with domain-specific instructions
- Access to relevant **knowledge base** sections
- Access to **customer data** (if authenticated)
- A defined **scope boundary** — if a query falls outside its domain, it signals the router to re-route

#### Policy Agent (includes billing)
- Answers questions about coverage, terms, benefits, exclusions, premiums, billing cycles, payment history
- Data sources: policy documents (RAG), customer policy records (authed)
- Key guardrail: must never state coverage exists or doesn't exist without grounding in the actual policy document

#### Claims Agent
- Answers questions about claims status, required documentation, claims process, timelines
- Data sources: claims process documentation (RAG), customer claims records (authed)
- Key guardrail: must not provide legal advice or make promises about claim outcomes

#### FAQ / General Agent
- Handles general insurance education, product comparisons, "how does insurance work" questions, finding a local agent
- Data sources: FAQ knowledge base, product pages (RAG only — no customer data needed)
- Lightest guardrail requirements since it doesn't deal with user-specific data

### 4.3 Knowledge & Data Layer

#### Knowledge Base (RAG)

Serves all agents with grounded, document-backed answers.

**Key design decisions:**
- **Chunking strategy:** Chunk by policy section / logical document boundaries, not arbitrary token windows. Insurance documents have clear sections (e.g., "Coverage A: Dwelling", "Exclusions") — preserve these as atomic retrieval units.
- **Metadata tagging:** Each chunk is tagged with metadata (document type, product line, section, effective date) to enable filtered retrieval. An agent asking about "auto claims" shouldn't retrieve "homeowner's exclusions."
- **Retrieval:** Hybrid search — semantic (vector similarity) + keyword (BM25) — to handle both natural language queries and specific term lookups (e.g., policy numbers, coverage codes).

**Tradeoff — single vector store vs. per-agent stores:**
- Single store is simpler to maintain but agents may retrieve irrelevant cross-domain chunks
- Per-agent stores offer cleaner retrieval but add operational overhead
- **Recommendation:** Single store with metadata filtering. Simpler, and metadata filters achieve most of the isolation benefit.

#### Customer Data Layer

- Accessed only for authenticated users
- Provides policy records, claims data, billing information via internal APIs or database queries
- **Must not expose raw data to the LLM** — a data access service should format/filter data before it reaches the agent (e.g., mask full account numbers, limit fields to what's relevant)

### 4.4 Memory System

#### Short-term Memory (Within a Session)

**Approach:** Sliding window + summarization hybrid

- Keep the **last K turns** (e.g., 5-10) verbatim in the context window for recency and detail
- Everything before that is **summarized into a running episode summary** that gets prepended to the context
- The summary is updated every N turns (or when the window slides)

```
Context passed to the agent:
┌──────────────────────────────────┐
│  Episode Summary                 │  ← compressed history
│  "User asked about auto policy   │
│   coverage, then billing cycle.  │
│   Has policy #AX-4421."         │
├──────────────────────────────────┤
│  Recent turns (verbatim)         │  ← last K turns, full detail
│  User: "What's my deductible?"  │
│  Bot: "Your deductible is..."   │
│  User: "Is that for collision?" │
└──────────────────────────────────┘
```

**Tradeoff:**
| Approach | Pros | Cons |
|----------|------|------|
| Full conversation buffer | Maximum accuracy | Hits token limits, expensive |
| Pure summarization | Token efficient | Lossy — may drop details user refers back to |
| **Sliding window + summary** | **Balances both** | **Slightly more complex to implement** |

#### Long-term Memory (Across Sessions — Logged-in Only)

**Approach:** Semantic memory store with user-scoped namespaces

- At the end of each session (or periodically), persist:
  - **Conversation summary** — what was discussed, what was resolved, what's pending
  - **User profile context** — extracted facts (e.g., "has 2 auto policies", "asked about roof damage claim 3 times")
  - **Unresolved issues** — questions the bot couldn't answer or that need follow-up
- Stored in a **vector database** with user-scoped namespaces (each user's memories are isolated)
- On new session start, retrieve relevant long-term memories based on the new query's semantic similarity

**Why vector DB for long-term memory:**
- User references to past interactions are semantic ("what did we discuss last time about my claim?") — keyword search fails here
- Vector similarity finds relevant past context even when phrasing differs

This can be implemented using solutions like Mem0, or a custom implementation backed by any vector database (Pinecone, Qdrant, Weaviate, pgvector, etc.).

### 4.5 Guardrails

#### Input Guardrails (Pre-processing)

Applied **before** the query reaches the router:

| Check | Purpose | Action |
|-------|---------|--------|
| Prompt injection detection | Prevent adversarial manipulation of agent behavior | Block + log, return generic "I can't process that" |
| PII detection | Prevent users from pasting sensitive data (SSN, credit card) into chat | Redact PII before forwarding, warn user |
| Off-topic filter | Detect clearly non-insurance queries | Polite decline: "I'm here to help with insurance questions" |
| Input length / rate limiting | Prevent abuse | Reject at gateway level |

#### Output Guardrails (Post-processing)

Applied **after** the specialist agent generates a response, **before** returning to the user:

| Check | Purpose | Action |
|-------|---------|--------|
| Grounding / hallucination check | Ensure claims about coverage, terms, or status are backed by retrieved data | If response references specifics not found in retrieved context, flag or remove |
| Disclaimer injection | Add legal/compliance disclaimers where needed | Append: "This is for informational purposes only. Please refer to your policy document for exact terms." |
| Compliance filter | Block responses that could be construed as legal, medical, or financial advice | Rephrase or add caveats |
| Tone check | Ensure response is professional and empathetic | Rewrite if flagged |

**Tradeoff — guardrails as separate LLM call vs. rule-based:**
- LLM-based: More flexible, catches nuanced issues, but adds latency and cost
- Rule-based (regex, keyword matching): Fast and cheap, but brittle
- **Recommendation:** Rule-based for input guardrails (injection patterns, PII regex), LLM-based for output guardrails (hallucination and compliance checks require understanding context)

### 4.6 Human Handoff

**Triggers:**
- Agent confidence score below threshold (e.g., retrieval similarity score too low)
- User explicitly asks for a human ("let me talk to someone")
- Sentiment detection: frustration, anger, repeated same question
- Query involves sensitive topics (complaints, legal disputes, claim denials)

**Handoff flow:**
1. System informs the user: "Let me connect you with a specialist who can help"
2. Passes the **conversation summary + context** to the human agent (not raw LLM internals)
3. Routes to the appropriate human queue (claims team, billing team, general support)

This is critical for a production system — no chatbot should be a dead-end.

---

## 5. Observability

Observability is not an afterthought — it's how you debug, improve, and build trust in a multi-agent system. In a multi-agent setup, a single user query passes through multiple components, and without proper tracing, diagnosing failures is nearly impossible.

### 5.1 What We Observe

#### Traces (Per-request lifecycle)

Every request gets a **trace ID** that follows it through the entire pipeline:

```
Trace: req_abc123
├── Gateway: auth check (2ms)
├── Input Guardrails: passed (15ms)
├── Router Agent: intent=claims, confidence=0.92 (180ms)
├── Claims Agent:
│   ├── RAG Retrieval: 4 chunks, top score=0.87 (120ms)
│   ├── Customer Data Fetch: claims record #CL-9921 (45ms)
│   └── LLM Generation: 312 tokens (850ms)
├── Output Guardrails: passed, disclaimer added (200ms)
└── Total latency: 1412ms
```

This tells you exactly where time is spent, where failures happen, and which agent handled the query.

#### Metrics

| Metric | Why |
|--------|-----|
| **Latency per component** (p50, p95, p99) | Identify bottlenecks — is the LLM slow or is retrieval slow? |
| **Routing accuracy** | Are queries going to the right agent? Track re-routes and fallbacks |
| **Retrieval quality** | Top-k similarity scores — are we retrieving relevant chunks? |
| **Guardrail trigger rates** | How often are input/output guardrails firing? High rates may indicate issues |
| **Human handoff rate** | If this is too high, the bot isn't doing its job. If too low, it may be overconfident |
| **User satisfaction signals** | Thumbs up/down, repeat questions (implicit dissatisfaction), session length |
| **Token usage per request** | Cost tracking, per agent and total |

#### Logs

- Every agent decision (intent classification, confidence score, selected data sources)
- Guardrail activations with the triggering content (redacted of PII)
- Memory operations (what was stored, what was retrieved, relevance scores)

### 5.2 Approach

Use a **tracing-first** observability platform. For LLM-based systems, tools like **Arize AI**, **LangSmith**, **Langfuse**, or **Phoenix** provide purpose-built observability that standard APM tools (Datadog, New Relic) don't cover well — specifically:

- LLM input/output logging with token counts
- Retrieval quality metrics (relevance scoring, chunk attribution)
- Prompt version tracking (which system prompt version produced this response)
- Evaluation pipelines (automated quality checks on a sample of responses)

**Tradeoff — build vs. buy:**
- Building custom observability is expensive and distracts from core product
- LLM observability platforms (Arize, Langfuse) give you 80% of what you need out of the box
- **Recommendation:** Use an LLM observability platform for tracing and evaluation. Supplement with standard infrastructure monitoring (for API latency, error rates, uptime).

### 5.3 Feedback Loop

Observability data feeds back into system improvement:

```
Observe → Identify low-quality responses → Analyze traces
    → Fix: improve prompts, add KB content, tune retrieval, adjust guardrails
        → Deploy → Observe again
```

This creates a continuous improvement cycle rather than a "deploy and hope" approach.

---

## 6. Logged-in vs. Logged-out: Unified Architecture

Rather than building two separate systems, we use a **single architecture with an auth-aware router** that adjusts behavior based on user state.

| Capability | Logged Out | Logged In |
|-----------|-----------|-----------|
| FAQ / general questions | Yes | Yes |
| Generic policy info | Yes | Yes |
| Personalized policy answers | No (prompt to log in) | Yes |
| Claims status | No (prompt to log in) | Yes |
| Billing details | No (prompt to log in) | Yes |
| Short-term memory | Yes (session only) | Yes (session only) |
| Long-term memory | No | Yes (cross-session) |
| Human handoff | Yes | Yes (with context) |

This avoids duplication while ensuring logged-out users still get value and are nudged toward logging in for personalized help.

---

## 7. Key Tradeoffs Summary

| Decision | Option A | Option B | Our Choice | Rationale |
|----------|----------|----------|------------|-----------|
| Routing approach | LLM-based | Trained classifier | LLM-based (start), hybrid (later) | Faster to build, iterate with traffic data |
| Memory strategy | Full buffer | Pure summary | Sliding window + summary hybrid | Balances accuracy and token efficiency |
| RAG store topology | Per-agent stores | Single shared store | Single store + metadata filtering | Simpler ops, metadata filters give sufficient isolation |
| Input guardrails | LLM-based | Rule-based | Rule-based | Injection/PII patterns are well-defined; speed matters here |
| Output guardrails | LLM-based | Rule-based | LLM-based | Compliance/hallucination checks need contextual understanding |
| Observability | Build custom | Use platform | LLM observability platform | Build vs. buy — platforms give 80% of value, focus engineering on core product |
| Auth handling | Two separate systems | Single auth-aware system | Single auth-aware system | Less duplication, simpler maintenance |

---

## 8. Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---------|--------|------------|
| LLM hallucination about coverage | High — user acts on wrong info | Output guardrails + grounding check + disclaimers |
| Router misclassifies intent | Medium — wrong agent responds | Agents detect out-of-scope queries and signal re-route |
| RAG retrieves irrelevant chunks | Medium — inaccurate response | Metadata filtering + relevance threshold cutoff |
| Customer data service is down | High — can't answer personalized queries | Graceful degradation: fall back to generic answers + inform user |
| LLM provider outage | Critical — full outage | Canned responses for common queries as fallback + human handoff |
| Memory store corruption/loss | Low — degraded personalization | Memory is supplementary, not critical path. System works without it |

---

## 9. Future Considerations (Out of Scope)

These are intentionally excluded from the current design but worth calling out:

- **Action-taking agents** — filing claims, making payments, updating policy details
- **Multi-language support** — serving non-English speaking customers
- **Voice channel** — extending the chatbot to phone/IVR systems
- **A/B testing framework** — testing different prompts, agent configurations, retrieval strategies
- **Fine-tuned models** — replacing general-purpose LLMs with insurance-domain fine-tuned models for specialist agents

---

## 10. Summary

This design provides a **pragmatic multi-agent architecture** that:

- **Decomposes cleanly** into 4 agents (router + 3 specialists) with clear boundaries
- **Handles both user states** through a unified, auth-aware routing system
- **Manages memory intelligently** with a hybrid short-term strategy and semantic long-term store
- **Prioritizes safety** with layered guardrails (input + output) appropriate for the insurance domain
- **Enables continuous improvement** through comprehensive observability and feedback loops
- **Degrades gracefully** with human handoff as the ultimate fallback

The architecture is tech-stack agnostic and can be implemented using any LLM provider, vector database, and programming language. The key insight is not the specific tools but the **separation of concerns** — each component has a clear responsibility, a defined interface, and can be independently improved or replaced.
