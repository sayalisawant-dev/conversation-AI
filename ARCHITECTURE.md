# Enterprise Conversational AI Platform — AWS Architecture

> **Version:** 1.0 | **Date:** August 2026  
> **Diagram file:** `architecture.drawio` (open with [draw.io](https://app.diagrams.net) or the VS Code draw.io extension)

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture Layers](#2-architecture-layers)
   - [Layer 0 — Client & Auth](#layer-0--client--auth)
   - [Layer 1 — Pre-Processing Pipeline](#layer-1--pre-processing-pipeline)
   - [Layer 2 — Intelligent Router](#layer-2--intelligent-router)
   - [Layer 3A — RAG Pipeline](#layer-3a--rag-pipeline)
   - [Layer 3B — Agent Core Pipeline](#layer-3b--agent-core-pipeline)
   - [Layer 3C — Live Human Agent Handoff](#layer-3c--live-human-agent-handoff)
   - [Layer 4 — Cross-Cutting Concerns](#layer-4--cross-cutting-concerns)
3. [AWS Services Reference](#3-aws-services-reference)
4. [Data Flow Summary](#4-data-flow-summary)
5. [Response Format Contract](#5-response-format-contract)
6. [Security Architecture](#6-security-architecture)
7. [Scalability & Resilience](#7-scalability--resilience)
8. [Cost Optimization Notes](#8-cost-optimization-notes)

---

## 1. System Overview

This platform is an **enterprise-grade, multi-modal Conversational AI system** built entirely on AWS. It accepts user queries from any client (web, mobile, API), processes them through a sequential pre-processing pipeline, classifies the intent, and routes to one of three execution paths:

| Route | When Used | Key Tech |
|---|---|---|
| **RAG** | Knowledge-base Q&A, document lookup | OpenSearch + Titan Embed + Claude |
| **Agent Core** | API actions, multi-step tasks, tool use | LangGraph + MCP + EC2 |
| **Live Human Agent** | Escalation, sensitive topics, low confidence | Amazon Connect + CRM |

All paths return a **structured JSON response** with a consistent schema.

```
Client → Auth → API Gateway → Pre-Processing Pipeline → Router → [RAG | Agent | Human] → JSON Response
```

---

## 2. Architecture Layers

---

### Layer 0 — Client & Auth

| Component | Service | Purpose |
|---|---|---|
| Web App | React / Angular | Browser-based chat UI |
| Mobile App | iOS / Android | Native mobile client |
| 3rd-Party API | REST client | B2B integrations |
| **Authentication** | **Amazon Cognito** | User Pools, App Client, JWT issuance |
| **API Entry Point** | **Amazon API Gateway** | Edge-optimised REST, rate limiting, WAF, TLS termination |

**Auth Flow:**
1. Client authenticates with Cognito → receives JWT (ID + Access tokens).
2. JWT is attached to every API call as `Authorization: Bearer <token>`.
3. API Gateway validates the JWT using the Cognito authorizer before forwarding.

---

### Layer 1 — Pre-Processing Pipeline

All steps run as **AWS Lambda functions** orchestrated sequentially (Step Functions Express Workflow or Lambda chaining). Each step enriches the request context object passed downstream.

#### ① Request Validation
- **Service:** AWS Lambda
- Schema validation (JSON Schema / Pydantic)
- JWT claim verification (user ID, scopes, tenant ID)
- Rate limit guard via DynamoDB token-bucket counter
- **Output:** Validated request object or `400 Bad Request`

#### ② Language Detection
- **Service:** Amazon Bedrock — Claude Haiku
- Prompt: `"Detect the ISO-639-1 language code for the following text: {input}"`
- Returns: `{ "language": "en", "confidence": 0.99 }`
- Unsupported languages return a localised error response immediately
- **Why Haiku:** Lowest latency and cost for a simple classification task

#### ③ Chitchat Detection
- **Service:** Amazon Bedrock — Claude Haiku
- Classifies input as `chitchat` (greetings, small talk) vs `task` (actionable query)
- Chitchat responses are handled with a lightweight canned/generated reply — **no routing needed**
- Saves cost by short-circuiting the full pipeline for non-task inputs

#### ④ Guardrail Safety
- **Service:** Amazon Bedrock Guardrails
- Configured policies:
  - **PII Redaction** — mask SSN, credit cards, email before any LLM call
  - **Toxic Content Filter** — block harmful, hateful, or unsafe inputs
  - **Topic Policy** — deny out-of-scope topics (configurable per tenant)
  - **Grounding Check** — applied again on output to prevent hallucination leakage
- Blocked requests return `403` with a safe user-facing message

#### ⑤ Query Preprocessing
- **Service:** Amazon Bedrock — Nova Lite
- Operations:
  - Spell correction and grammar fix
  - Acronym expansion (domain-specific glossary injected via system prompt)
  - **HyDE (Hypothetical Document Embedding)** — generates a hypothetical answer to improve vector search quality
  - Query rewriting for multi-turn context resolution
- **Why Nova Lite:** Fast, cheap, sufficient intelligence for rewriting — saves Nova Pro quota for classification

#### ⑥ Intent Classification
- **Service:** Amazon Bedrock — Nova Pro
- Multi-label classification returning:
  ```json
  {
    "primary_intent": "account_inquiry",
    "secondary_intents": ["billing"],
    "confidence": 0.94,
    "requires_tool": true,
    "escalation_risk": 0.12,
    "domain": "finance"
  }
  ```
- This metadata drives the **Router** decision
- **Why Nova Pro:** Highest accuracy model for nuanced intent classification across domains

---

### Layer 2 — Intelligent Router

- **Service:** AWS Lambda (rule engine + LLM fallback)
- **Decision logic:**

```
IF escalation_risk > 0.7  → Live Human Agent
ELSE IF requires_tool = true OR domain in [finance, hr, ops] → Agent Core
ELSE → RAG
```

- LLM fallback (Nova Pro) used when rules are ambiguous
- Router output includes the chosen path + rationale (logged to CloudWatch)

---

### Layer 3A — RAG Pipeline

End-to-end Retrieval-Augmented Generation for knowledge-base questions.

#### ⑧a Embedding
- **Service:** Amazon Bedrock — Titan Text Embeddings v2
- Embeds the **rewritten query** (from step ⑤) into a 1536-dimensional vector
- Multi-lingual: supports 100+ languages natively

#### ⑧b Vector Search
- **Service:** Amazon OpenSearch Serverless (k-NN collection)
- Index: HNSW algorithm, cosine similarity metric
- Filters: tenant ID, document category, date range (metadata filters)
- Returns top-K=10 candidate chunks with similarity scores

#### ⑧c Top-K Retrieval
- Fetches full chunk content from **Amazon S3** (raw document store)
- S3 contains versioned, chunked documents (500-token chunks, 50-token overlap)
- Chunks stored with metadata: `{ source, page, section, last_updated }`

#### ⑧d Re-Ranking
- **Service:** Amazon Bedrock — Cohere Rerank v3
- Cross-encoder reranks the top-10 chunks against the original query
- Narrows to **top-K=3** most relevant chunks
- Eliminates semantic similarity false positives from ANN search

#### ⑧e Generation + Citation
- **Service:** Amazon Bedrock — Claude 3.5 Sonnet
- System prompt enforces:
  - Answer only from provided context
  - Cite every factual claim with `[Source: doc_name, page: N, score: 0.XX]`
  - Respond in the user's detected language
- Output passes through **Bedrock Guardrails** grounding check before returning

#### RAG Response Schema
```json
{
  "session_id": "uuid",
  "answer": "string",
  "citations": [
    { "source": "policy_v3.pdf", "page": 12, "score": 0.91 }
  ],
  "confidence": 0.89,
  "language": "en",
  "path": "RAG",
  "latency_ms": 1240
}
```

---

### Layer 3B — Agent Core Pipeline

Multi-step task execution using an LLM-driven agent graph with real API calls.

#### ⑨a Supervisor Agent
- **Framework:** LangGraph `StateGraph`
- Decomposes the task into sub-tasks
- Selects the appropriate **Subject Matter Expert (SME) agent** node
- Manages memory state across turns (persisted in DynamoDB)
- Implements **ReAct loop**: Reason → Act → Observe → Repeat

#### SME Agent Nodes (LangGraph Nodes)
| Agent | Domain | Model |
|---|---|---|
| Finance SME | Account balance, transactions, limits | Nova Pro |
| Support SME | Order status, returns, complaints | Nova Pro |
| General SME | FAQ, product info, policies | Nova Pro |

Each SME agent has:
- A domain-tuned system prompt
- A defined tool set (MCP tool definitions)
- A maximum iteration budget (prevents infinite loops)

#### ⑨b API Gateway (Internal)
- **Service:** Amazon API Gateway (VPC Link to private subnet)
- JWT-authenticated (Cognito — service account token)
- Rate-limited per SME agent
- Routes tool calls to the correct backend API endpoint

#### ⑨c MCP Tool Server
- **Protocol:** Model Context Protocol (MCP)
- Tool definitions in JSON schema format
- Each tool specifies: `name`, `description`, `inputSchema`, `outputSchema`
- Example tools: `get_account_balance`, `create_support_ticket`, `lookup_order_status`
- MCP server runs as a **Lambda function** or **ECS container**

#### ⑨d EC2 Business API (Private Subnet)
- Actual backend microservices deployed on EC2
- **Network:** Private subnet, no direct internet access
- **Access pattern:** Agent → API GW → VPC Link → ALB → EC2 Auto Scaling Group
- Security Groups allow inbound only from the ALB
- Secrets (DB credentials, API keys) fetched from **AWS Secrets Manager** at runtime

#### Agent Core Response Schema
```json
{
  "session_id": "uuid",
  "answer": "string",
  "tool_calls": [
    { "tool": "get_account_balance", "input": {}, "output": {} }
  ],
  "agent_trace": ["supervisor", "finance_sme", "tool:get_account_balance"],
  "language": "en",
  "path": "AGENT_CORE",
  "latency_ms": 3100
}
```

---

### Layer 3C — Live Human Agent Handoff

Graceful escalation path for sensitive, high-risk, or low-confidence scenarios.

#### ⑩a Conversation Summary
- **Service:** Amazon Bedrock — Claude Haiku
- Generates a concise summary of the full conversation history
- Extracts: user intent, key entities, sentiment, prior attempts, unresolved issues
- Summary is attached to the ticket for the human agent

#### ⑩b Ticket Creation
- **Service:** AWS Lambda → REST API call to CRM
- Supported targets: **Zendesk**, **ServiceNow**, **Salesforce Service Cloud**
- Ticket includes: user ID, conversation transcript, summary, priority score
- Returns ticket ID for tracking

#### ⑩c Queue & Route
- **Service:** Amazon Connect
- Contact Flow routes the ticket to the correct agent queue
- Skill-based routing: language, domain expertise, SLA tier
- Estimated wait time communicated back to the user via the bot

#### ⑩d Agent Notification
- **Service:** Amazon SNS / SES
- Live agent desktop receives: ticket link, full transcript, AI summary
- Push notification to agent workspace (Zendesk widget / Connect CCP)

#### Live Human Agent Response Schema
```json
{
  "session_id": "uuid",
  "ticket_id": "ZD-98234",
  "queue_position": 3,
  "estimated_wait_seconds": 180,
  "summary": "User is unable to reset 2FA and has been locked out for 2 days.",
  "path": "LIVE_HUMAN",
  "language": "en"
}
```

---

### Layer 4 — Cross-Cutting Concerns

These services apply across all paths.

| Service | Role |
|---|---|
| **Amazon CloudWatch** | Metrics, logs, alarms, LLM latency dashboards |
| **AWS X-Ray** | Distributed tracing across Lambda, API GW, Bedrock calls |
| **Amazon DynamoDB** | Conversation history, session state (TTL: 24h), rate-limit counters |
| **Amazon ElastiCache (Redis)** | Response caching, session affinity, token-bucket state |
| **AWS Secrets Manager** | API keys, DB credentials, MCP tool secrets — no hardcoded secrets |
| **AWS KMS** | Encryption at rest for S3, DynamoDB, OpenSearch |
| **VPC + Private Subnets** | EC2, RDS, ElastiCache isolated from internet; PrivateLink for Bedrock |
| **AWS WAF + Shield** | DDoS protection, IP allow/deny, SQL injection rules on API GW |
| **Amazon SQS / SNS** | Async event bus, decouples pipeline stages, dead-letter queues |
| **AWS Step Functions** | Orchestrates the pre-processing pipeline (retry, error handling) |

---

## 3. AWS Services Reference

| # | Requirement | AWS Service | Model / Config |
|---|---|---|---|
| 1 | Auth | Amazon Cognito | User Pool + App Client |
| 2 | Language Detection | Amazon Bedrock | Claude Haiku (claude-3-haiku-20240307) |
| 3 | Guardrail Safety | Amazon Bedrock Guardrails | PII + Toxicity + Topic Policy |
| 4 | Intent Classification | Amazon Bedrock | Nova Pro (amazon.nova-pro-v1:0) |
| 5 | Query Rewrite | Amazon Bedrock | Nova Lite (amazon.nova-lite-v1:0) |
| 6 | Embedding | Amazon Bedrock | Titan Text Embeddings v2 |
| 7 | Vector Database | Amazon OpenSearch Serverless | k-NN collection, HNSW |
| 8 | Response Format | — | JSON (standardised schema) |
| 9 | Agent Orchestration | LangGraph (on Lambda / ECS) | StateGraph + ReAct |
| 10 | API Tool Connection | MCP Tool Server | Lambda / ECS, JSON schema |
| 11 | Live Agent API | Lambda + Amazon Connect + CRM | REST + SNS |

---

## 4. Data Flow Summary

```
User Input
    │
    ▼
[Cognito JWT Auth]
    │
    ▼
[API Gateway — WAF — Rate Limit]
    │
    ▼
[① Request Validation — Lambda]
    │
    ▼
[② Language Detection — Bedrock Haiku]
    │
    ├── Unsupported language → Return localised error
    ▼
[③ Chitchat Detection — Bedrock Haiku]
    │
    ├── Chitchat → Return lightweight response (short-circuit)
    ▼
[④ Guardrail Safety — Bedrock Guardrails]
    │
    ├── Blocked → Return safe error (403)
    ▼
[⑤ Query Preprocessing — Bedrock Nova Lite]
    │
    ▼
[⑥ Intent Classification — Bedrock Nova Pro]
    │
    ▼
[⑦ Router — Lambda Rule Engine]
    │
    ├── RAG ──────────────────────────────────────────────────────────┐
    │   Embed → OpenSearch → Top-K → Rerank → Claude 3.5 → Response  │
    │                                                                  │
    ├── Agent Core ────────────────────────────────────────────────── │
    │   Supervisor (LangGraph) → SME Agent → MCP → API GW → EC2      │
    │                                                                  │
    └── Live Human Agent ─────────────────────────────────────────── │
        Summarise → Ticket → Connect Queue → Human Agent             │
                                                                      ▼
                                                           [JSON Response to Client]
```

---

## 5. Response Format Contract

All three paths return a unified JSON envelope:

```json
{
  "status": "success | error | escalated",
  "session_id": "uuid-v4",
  "path": "RAG | AGENT_CORE | LIVE_HUMAN | CHITCHAT | BLOCKED",
  "language": "ISO-639-1 code",
  "answer": "string (localised, in user's language)",
  "citations": [],        // RAG only
  "tool_calls": [],       // Agent Core only
  "ticket_id": "string",  // Live Human only
  "confidence": 0.0,
  "latency_ms": 0,
  "request_id": "uuid-v4",
  "timestamp": "ISO-8601"
}
```

---

## 6. Security Architecture

### Authentication & Authorisation
- **Cognito User Pools** — user identity, MFA support
- **Cognito App Clients** — machine-to-machine (M2M) for API clients
- **API Gateway** — Cognito authorizer validates JWT on every request
- **IAM Roles** — least-privilege per Lambda, no shared credentials

### Network Security
- All Bedrock API calls go via **VPC PrivateLink** — no public internet
- EC2 business APIs in **private subnets** — accessible only via ALB + VPC Link
- **Security Groups** — deny all inbound by default, allow only specific ports/sources
- **NACLs** — subnet-level second layer of network control

### Data Security
- **KMS Customer Managed Keys (CMK)** — encrypt S3, DynamoDB, OpenSearch at rest
- **TLS 1.2+** enforced on all in-transit connections
- **Secrets Manager** — rotate credentials automatically (no .env files)
- **PII Redaction** via Bedrock Guardrails before any LLM sees raw input

### Compliance Controls
- **CloudTrail** — full audit log of all API calls
- **Config Rules** — continuous compliance checks (encryption, public access blocks)
- **GuardDuty** — threat detection on CloudTrail + VPC flow logs

---

## 7. Scalability & Resilience

| Concern | Solution |
|---|---|
| Lambda cold starts | Provisioned concurrency on critical path functions |
| OpenSearch latency | ElastiCache Redis for repeated query caching |
| Bedrock throttling | Exponential backoff + SQS async queue for burst traffic |
| EC2 API availability | Auto Scaling Group + ALB health checks + multi-AZ |
| Agent loop runaway | LangGraph max_iterations budget per agent invocation |
| Pipeline failure | Step Functions error handling + dead-letter SQS queue |
| Session consistency | DynamoDB for distributed session state (no Lambda affinity needed) |
| Multi-region DR | Route 53 health checks + S3 cross-region replication (optional) |

---

## 8. Cost Optimization Notes

| Strategy | Detail |
|---|---|
| **Chitchat short-circuit** | Haiku is 20x cheaper than Nova Pro; skips the full pipeline for ~30% of traffic |
| **Nova Lite for rewrite** | Query rewriting doesn't need Nova Pro's reasoning capability |
| **OpenSearch Serverless** | Pay per OCU-hour, scales to zero during off-hours |
| **Bedrock on-demand** | No upfront commitment; use Provisioned Throughput only after traffic is proven |
| **Response caching** | ElastiCache caches identical query responses (TTL: 5 min) — reduces Bedrock spend |
| **S3 Intelligent-Tiering** | Document store auto-moves cold chunks to cheaper storage tiers |
| **Lambda ARM64** | ~20% cost reduction vs x86 for same performance |
| **DynamoDB TTL** | Auto-purge old sessions — prevents unbounded storage growth |

---

*Architecture designed for AWS — August 2026*  
*Draw.io diagram: `architecture.drawio`*
