# Agentic AI Platform Vendor Onboarding Brief

**Document purpose**

This document is intended for vendor onboarding and solution alignment. It summarizes the current in-house Agentic AI codebase, what has already been built, what should be built next using industry-standard patterns, and how the proposed LLM gateway, new agents, and global guardrail module fit into the target architecture.

**Current date**

17 June 2026

## 1. Executive Summary

The repository already contains a multi-agent, FastAPI-based enterprise AI platform with several working domain agents, a centralized marketing agent stack, a master routing layer, MIS natural-language-to-SQL capabilities, ticketing automation, knowledge-base support, observability hooks, and shared deployment scaffolding.

The next step is not to start from scratch. The right move is to standardize the platform layer around:

- a shared LLM gateway
- a centralized guardrail module built in-house
- common identity, logging, and telemetry patterns
- a consistent agent contract for new agents such as AI Hire Tool and MIS Agent

This document should be used as the baseline for vendor discussion, gap analysis, and implementation planning.

## 2. What Has Already Been Built

### 2.1 Platform and API Surface

The root application mounts multiple FastAPI sub-applications under a single API surface. This is already a multi-service architecture rather than a single monolith.

Built components include:

- root API aggregator with CORS and health endpoint
- marketing agent service
- advisor service
- customer service agent
- intranet knowledge assistant
- master agent router
- MIS agent
- issue resolver / ticketing assistant
- knowledge-base manager
- Streamlit front-end launcher and service selector

### 2.2 Marketing Agent Platform Layer

The `marketing_agent` service is the most mature platform layer in the repo. It already includes:

- FastAPI request handling
- request/response DTOs
- configurable provider routing
- Gemini, Bedrock, Vertex AI, and OpenAI-compatible provider wrappers
- prompt registry and profile-based prompting
- context retention and session rollover
- file ingestion and output generation
- MongoDB-based query logging and cost logging
- LangSmith tracing hooks
- basic guardrails
- usage quota checks
- health and cost APIs

This is effectively the foundation for a more generic enterprise LLM platform.

### 2.3 Domain Agents

The repository already has several domain-specific agents:

- `advisor`
  - insurance / product assistant
  - retrieval over brochure content
  - policy and product workflows

- `customer_service`
  - customer profile and update workflows
  - login and authenticated service flow
  - document and OCR handling

- `Intranet`
  - internal HR / policy / employee knowledge assistant
  - vector retrieval over internal policy documents

- `Issue_Resolver`
  - ticket creation and classification workflow
  - structured routing into application / infrastructure categories

- `MIS Agent`
  - natural-language-to-SQL reporting assistant
  - SQL execution against business database
  - business glossary / knowledge-base support

- `master_agent`
  - query classification and routing across specialist agents
  - fallback conversational assistant

- `kb_manager`
  - knowledge-base support layer

### 2.4 Shared Utilities and Supporting Pieces

The repo also includes supporting assets and utilities:

- shared logging helpers
- database / MongoDB helpers
- vector database assets
- FAISS indexes for retrieval-backed agents
- sample user and environment files
- deployment workflow definitions
- Streamlit demo UI

## 3. Existing Codebase Review

### 3.1 Strengths

- Clear separation between API layer, domain logic, data access, and LLM provider abstractions in `marketing_agent`
- Multiple agent domains already represented in code
- Multi-provider strategy exists, which is a good base for a future gateway
- Logging and telemetry are already considered rather than being added as an afterthought
- File handling and context management are already implemented for enterprise-style workflows
- MIS capability already exists as a dedicated service rather than being embedded inside an unrelated app

### 3.2 Gaps and Risks

- There is no single centralized LLM gateway yet across the whole platform
- Guardrails are implemented locally in the marketing service, not as a shared policy module
- Agent contracts are not fully standardized across all services
- Some services appear to be independently evolved, which increases maintenance cost
- Authentication, authorization, and tenant controls are uneven across apps
- The current codebase mixes platform concerns and solution concerns in multiple places
- Some legacy or demo-oriented wiring still exists in the UI and routing layer

### 3.3 Architectural Implication

The current codebase is already valuable, but it should now be treated as a platform consolidation problem rather than a greenfield build.

## 4. What Should Be Built Next

The following items are the industry-standard next components for a platform at this stage.

### 4.1 Shared LLM Gateway

This should be the next major platform component.

#### Responsibilities

- normalize provider access behind one interface
- enforce authentication, authorization, and tenant policy
- apply prompt / request / response guardrails consistently
- route requests to the correct model or provider
- support fallback and retry logic
- track token usage, latency, cost, and error classes
- provide provider health checks and capability discovery
- standardize request metadata and audit logs
- enable caching for safe reuse scenarios
- support future tool-calling and agent orchestration

#### Why it matters

Today, provider logic is present in a few places. A gateway will prevent each agent from directly owning provider complexity.

The gateway should become the only place where provider-specific logic is allowed to live.

### 4.2 Global Guardrail Module

This should be built in-house and reused by all agents.

#### Responsibilities

- prompt injection detection
- policy and scope enforcement
- SQL / code / shell safety checks
- PII and sensitive-data handling
- jailbreak and exfiltration detection
- unsafe file handling checks
- response shaping for safe refusals
- domain-scoped allow/deny policy

#### Design principle

The guardrail module should be platform-level, not agent-level. Each agent can add domain rules, but all agents must pass through a common policy layer first.

### 4.3 Unified Agent Contract

Every new agent should follow one standard shape:

- request schema
- response schema
- health endpoint
- capability description
- logging metadata
- guardrail entrypoint
- provider gateway integration
- cost and trace reporting

This reduces integration friction for product teams and vendors.

### 4.4 Common Observability Stack

Industry-standard observability should include:

- request tracing
- prompt/version tracing
- token and cost telemetry
- structured audit logs
- error classification
- latency histograms
- per-agent usage metrics

### 4.5 Security and Governance

The platform should include:

- RBAC or scoped access control
- per-agent permission boundaries
- secrets management
- environment-based deployment config
- PII masking in logs
- human review hooks for sensitive actions
- audit-ready storage for requests and decisions

## 5. Proposed New Agents

## 5.1 AI Hire Tool

This appears to be a new agent to be built, not a fully implemented component in the current codebase.

#### Intended business role

- support hiring and recruitment workflows
- help with candidate screening
- summarize resumes
- generate interview notes
- compare profiles against job requirements
- assist with hiring decision support

#### Recommended capabilities

- resume parsing and normalization
- JD-to-candidate matching
- interview question generation
- structured evaluation summaries
- shortlist ranking with explainability
- policy-aware refusal for protected characteristics
- HR data privacy controls

#### Recommended integrations

- ATS or HRMS
- document parsing / OCR
- role-based access
- hiring manager approval workflow
- audit log for each recommendation

#### Industry-standard next build items

- candidate scoring rubric
- explainability layer
- structured feedback capture
- duplicate candidate detection
- redaction of sensitive information

### 5.2 MIS Agent

The MIS Agent already exists in the repo as a standalone FastAPI + Gemini + SQLAlchemy service.

#### What it currently does

- accepts natural-language business questions
- generates SQL
- executes against a database
- returns results and a language summary
- exposes endpoints for table info, query execution, and knowledge-base access

#### What should be standardized next

- move SQL generation behind the shared LLM gateway
- move SQL safety policy into the global guardrail module
- add strict read-only enforcement for production use
- add query explain / dry-run support
- add query approval for sensitive datasets
- add result-size limits and pagination
- add structured metrics and lineage logging

#### Recommended enterprise positioning

MIS Agent should become the governed analytics assistant for business users, not just a raw SQL generator.

## 6. Target Platform Architecture

### 6.1 Proposed Layering

1. Client applications
2. API gateway / service router
3. Global guardrail module
4. LLM gateway
5. Agent services
6. Data stores, vector stores, and enterprise systems
7. Observability and audit layer

### 6.2 How Requests Should Flow

1. Client sends request to an agent endpoint or platform entrypoint
2. Request is authenticated and normalized
3. Global guardrail evaluates policy and safety
4. LLM gateway selects provider / model / fallback strategy
5. Agent business logic runs
6. Output is logged, traced, costed, and returned

### 6.3 Integration Principle

No agent should directly implement its own provider-selection logic once the gateway is introduced. The agent should ask the gateway for model execution and focus only on domain behavior.

## 7. Industry-Standard Components Still to Build

The platform should eventually include:

- LLM gateway
- in-house global guardrail module
- standard auth and role management
- workflow engine for approval-based actions
- vector retrieval service
- prompt registry and versioning
- evaluation harness
- A/B testing framework for prompts and models
- rate limiting and quota enforcement
- human-in-the-loop review queues
- agent registry and capability catalog
- data retention and deletion controls
- monitoring dashboards and alerting

## 8. Build Priority Recommendation

### Phase 1

- build the shared LLM gateway
- formalize the global guardrail module
- standardize request/response contracts
- align logging and observability

### Phase 2

- harden MIS Agent into a governed analytics agent
- design and implement AI Hire Tool
- connect both agents to the platform standards

### Phase 3

- add agent registry
- add evaluation and prompt/version management
- add workflow approval and human review steps
- expand metrics, dashboards, and governance

## 9. Vendor Discussion Points

Use this list during vendor onboarding:

- confirm if the vendor will build or integrate the LLM gateway
- confirm whether guardrails are vendor-managed or in-house owned
- confirm support for Gemini, Bedrock, Vertex, and OpenAI-compatible models
- confirm support for retrieval, file handling, and structured outputs
- confirm how the vendor handles RBAC, auditability, and data privacy
- confirm how metrics, tracing, and cost governance will be implemented
- confirm integration plan for AI Hire Tool and MIS Agent
- confirm how the vendor will avoid duplicate provider logic inside agents

## 10. Summary

The current codebase already has the raw ingredients of an enterprise agent platform. The next step is to consolidate those ingredients into a governed platform layer with a shared LLM gateway, a global guardrail module, and standardized agent contracts.

For this vendor onboarding cycle, the most important message is:

- do not rebuild what already exists
- do standardize what is fragmented
- do centralize model access and policy enforcement
- do treat AI Hire Tool and MIS Agent as governed productized agents rather than isolated scripts

