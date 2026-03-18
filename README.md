# Multi-Agent Insurance Chatbot: Design Document

A system design for a multi-agent chatbot that answers customer queries in the insurance domain. Supports both authenticated and anonymous users, with short-term and long-term memory for personalized experiences.

## What's Inside

- [**DESIGN_DOC.md**](./DESIGN_DOC.md) - Full design document covering architecture, component design, tradeoffs, and evaluation strategy
- [**architecture.svg**](./architecture.svg) - System architecture diagram

## Architecture Overview

![System Architecture](./architecture.svg)

## Key Highlights

- **4-agent decomposition**: Router + Policy Agent + Claims Agent + FAQ Agent
- **Auth-aware routing**: Single system that adapts behavior for logged-in vs. logged-out users
- **Hybrid memory**: Sliding window + summarization for short-term, vector DB for long-term
- **Layered guardrails**: Rule-based input filtering, LLM-based output validation
- **Latency-conscious**: Streaming, parallel retrieval, semantic caching
- **Evaluation-ready**: Golden test set + LLM-as-judge for quality measurement
