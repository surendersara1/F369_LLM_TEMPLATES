<!-- Template Version: 2.0 | F369 Wave 15 (composite) | Composes: BEDROCK_AGENTS_MULTI_AGENT + BEDROCK_KNOWLEDGE_BASES + BEDROCK_FLOWS_PROMPT_MGMT + SERVERLESS_LAMBDA_POWERTOOLS -->

# Template 36 — Multi-Agent Collaboration (supervisor + specialists · KB + tools · prompt mgmt · 4-6 day production agent system)

## Purpose

Stand up a **production-grade multi-agent system on Bedrock Agents in 4-6 days**. Output: supervisor agent + 3-5 specialist collaborator agents, each with their own KB + action groups + Bedrock Guardrails + Code Interpreter. Used for complex domain queries that span multiple specialty areas.

This is the **canonical "specialist agent team" engagement** — better than a single-mega-agent for non-trivial domains. Examples: customer support (orders + billing + technical), HR assistant (policies + benefits + payroll + onboarding), enterprise IT helpdesk (provisioning + access + troubleshooting).

---

## Role Definition

You are an expert AWS multi-agent architect with deep expertise in:
- Bedrock Agents Multi-Agent Collaboration (Dec 2024 GA) — supervisor router + collaborators
- Bedrock Action Groups (Lambda + OpenAPI + Code Interpreter + Return of Control)
- Bedrock Knowledge Base association per agent
- Bedrock Prompt Management for versioned agent prompts + variants for A/B
- Session state + memory configuration (cross-session continuity)
- Custom orchestration via promptOverrideConfiguration
- Confirmation + ROC flows for sensitive operations
- Multi-tenant agent design + tenant_id propagation

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
ENV:                         [REQUIRED]

# --- AGENT TEAM SHAPE ---
SUPERVISOR_NAME:             [REQUIRED — e.g. customer-support-supervisor]
COLLABORATORS:               [REQUIRED — JSON array of {name, role, model, kb_id, actions}:
                              [
                                {"name":"orders","role":"Handles order queries","model":"sonnet-4-6",
                                 "kb_id":"kb-orders","actions":["lookup_order","cancel_order"]},
                                {"name":"billing","role":"Handles billing","model":"haiku-4-5",
                                 "kb_id":"kb-billing","actions":["get_invoice","refund"]},
                                {"name":"technical","role":"Product Q","model":"sonnet-4-6",
                                 "kb_id":"kb-product-docs","actions":["search_solutions","escalate"]}
                              ]
                             ]

# --- COLLABORATION ---
COLLABORATION_MODE:          [SUPERVISOR_ROUTER default — also: SUPERVISOR (sequential)]
RELAY_HISTORY:               [TO_COLLABORATOR default — also: DISABLED]

# --- MEMORY ---
ENABLE_MEMORY:               [true default — cross-session summaries]
MEMORY_RETENTION_DAYS:       [30 default]

# --- TOOLS ---
ENABLE_CODE_INTERPRETER:     [true default for technical agents]
ENABLE_RETURN_OF_CONTROL:    [true for sensitive ops (refund, charge_card, delete_account)]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
ENABLE_GUARDRAILS:           [true required for prod]
TENANT_AWARE:                [true if multi-tenant SaaS]

# --- SESSION ---
SESSION_TTL_SECONDS:         [600 default — 10 min]
ENABLE_TRACE:                [true in dev; false in prod]

# --- API ---
API_DOMAIN:                  [REQUIRED — e.g. agent.example.com]
COGNITO_USER_POOL_ID:        [REQUIRED]

# --- OBSERVABILITY ---
SNS_ALARM_TOPIC_ARN:         [REQUIRED]
ENABLE_AGENT_EVAL:           [true default — track answer quality]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `BEDROCK_AGENTS_MULTI_AGENT` | Multi-agent collaboration + action groups + ROC + memory |
| `BEDROCK_KNOWLEDGE_BASES` | Per-collaborator KB |
| `BEDROCK_FLOWS_PROMPT_MGMT` | Prompt Management for versioned agent instructions |
| `SERVERLESS_LAMBDA_POWERTOOLS` | Action group Lambdas |
| `SERVERLESS_HTTP_API_COGNITO` | API Gateway + Cognito for inbound |
| `LAYER_OBSERVABILITY` | Trace + metrics + grounding eval |
| `LAYER_SECURITY` | KMS + Secrets Manager |

---

## Architecture (customer support example)

```
   User → Web/Mobile App
        │
        │ Cognito JWT
        ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ API Gateway HTTP API (agent.example.com)                        │
   │   - JWT authorizer                                                │
   │   - WAF + rate limit                                                │
   └────────────────────────┬───────────────────────────────────────┘
                            │
                            ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ Inference Lambda                                                 │
   │   1. Validate JWT, extract user_id + tenant_id                    │
   │   2. invoke_agent on supervisor (with sessionState)               │
   │   3. Stream response back to API GW                                │
   │   4. Log + metrics                                                  │
   └────────────────────────┬───────────────────────────────────────┘
                            │
                            ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ Supervisor Agent                                                 │
   │   Model: Sonnet 4.6                                                │
   │   Mode: SUPERVISOR_ROUTER                                          │
   │   Instruction: "Route customer queries to specialist agents..."     │
   │   Memory: SESSION_SUMMARY (30d retention)                            │
   │   KB: company-wide (general policies, escalation paths)             │
   │   Guardrails: PII anonymize + competitor block                       │
   │                                                                     │
   │   Collaborators:                                                   │
   │     ├─ orders-agent     (Sonnet, KB=orders, actions=lookup, cancel) │
   │     ├─ billing-agent    (Haiku,  KB=billing, actions=invoice, refund) │
   │     ├─ technical-agent  (Sonnet, KB=docs, actions=search, escalate) │
   │     └─ human-handoff    (Haiku,  no KB,    actions=create_ticket)   │
   │                                                                     │
   │   Each collaborator agent:                                          │
   │     - Own foundation model + instruction                            │
   │     - Own KB association                                             │
   │     - Own Action Group Lambda                                        │
   │     - Own Guardrails                                                  │
   │     - require_confirmation: ENABLED on mutating actions               │
   └────────────────────────────────────────────────────────────────┘
                            │
                            ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ Action Group Lambdas (one per collaborator)                     │
   │   - Powertools logger + tracer + idempotency + event_parser     │
   │   - tenant_id propagated via session attributes                   │
   │   - Tool implementations: DDB queries, RDS, external APIs         │
   └────────────────────────────────────────────────────────────────┘
```

---

## Day-by-day execution (4-6 day deploy)

### Day 1 — Per-collaborator KBs + agents draft
- KMS multi-region CMK
- Per-collaborator KB (using `BEDROCK_KNOWLEDGE_BASES` template) — orders KB, billing KB, technical KB
- Initial ingestion of source data per KB
- Bedrock Agent draft for each collaborator (no actions yet) + Guardrails
- **Deliverable:** End of Day 1: each collaborator can be invoked individually + retrieves correct KB content.

### Day 2 — Action Groups for each collaborator
- Action Group Lambda per collaborator (Powertools-based) — implements business actions:
  - orders-agent: `lookup_order`, `cancel_order`, `update_address`
  - billing-agent: `get_invoice`, `refund` (ROC for sensitive), `update_payment_method`
  - technical-agent: `search_solutions`, `escalate_to_engineer`
- Per-action `require_confirmation: ENABLED` on mutating ops
- Test each collaborator individually with sample queries
- **Deliverable:** End of Day 2: each collaborator handles its specialty queries correctly + actions execute.

### Day 3 — Supervisor + Multi-Agent Collaboration
- Supervisor agent created with `agent_collaboration: SUPERVISOR_ROUTER`
- Each collaborator alias associated to supervisor via `CfnAgentCollaborator`
- Supervisor instruction tuned: routing logic + when to invoke single vs multiple collaborators + how to synthesize
- Test routing: "My order is wrong AND I was charged twice" → supervisor routes to orders + billing → synthesizes
- **Deliverable:** End of Day 3: supervisor correctly routes 90%+ of test queries.

### Day 4 — Memory + Sessions + Code Interpreter
- (If `ENABLE_MEMORY`) configure memory on supervisor; verify summaries persist across sessions
- (If `ENABLE_CODE_INTERPRETER`) add Code Interpreter built-in action group on technical-agent (for math, data analysis)
- Session attributes: `tenant_id`, `user_id`, `tier` propagated through collaborator chain
- (If `TENANT_AWARE`) verify tenant_id used in KB filter + DDB query (no cross-tenant leakage)
- **Deliverable:** End of Day 4: cross-session continuity working; technical-agent can solve a "what's 15% discount on $XYZ" via Code Interpreter.

### Day 5 — Inference API + UI integration
- Inference Lambda with Powertools + Pydantic
- API Gateway HTTP API + Cognito JWT + WAF + custom domain
- Streaming response (use invoke_agent with streaming response handling)
- React chat UI (Cognito Hosted UI sign-in) with markdown rendering + citation links
- Test: end-user → web UI → supervisor → collaborators → response within 5-15s
- **Deliverable:** End of Day 5: chat UI live; sample 10 queries demo full flow.

### Day 6 — Eval + observability + handoff
- (If `ENABLE_AGENT_EVAL`) automated eval suite:
  - 50 Q&A pairs covering each collaborator's domain
  - Daily Lambda runs eval → metrics: routing accuracy, answer correctness, citation accuracy
- CloudWatch dashboard: query volume, latency, routing distribution, model token usage, cost per session
- 8 alarms: agent errors, latency p99 > 15s, routing failures (no collaborator picked), Guardrail blocks > 5%, model fallbacks, KB sync errors, cost spike, action Lambda errors
- Pytest validation suite
- **Deliverable:** End of Day 6: eval baseline = ≥ 80% routing accuracy; all alarms green; runbook signed off.

---

## Validation criteria

- [ ] **All collaborator agents PREPARED** + each has KB + action group + Guardrails
- [ ] **Supervisor PREPARED** + collaborators associated
- [ ] **Routing accuracy ≥ 80%** on test set (50+ queries across collaborator domains)
- [ ] **Multi-collaborator routing** works for cross-domain queries
- [ ] **`require_confirmation` enforced** on mutating actions (test: agent doesn't auto-cancel without confirm)
- [ ] **Tenant isolation** — multi-tenant test confirms tenant A's data not surfaced to tenant B
- [ ] **Memory persists across sessions** (if enabled) — verified by 2-session test
- [ ] **Code Interpreter** works on technical-agent (math + data manipulation)
- [ ] **Bedrock Guardrails** redact PII in responses; block forbidden topics
- [ ] **API Gateway custom domain** reachable + JWT-protected
- [ ] **Streaming response** works in React UI (token-by-token)
- [ ] **Citation links** render correctly + click opens source doc
- [ ] **Eval suite passes** ≥ 80% on test set; tracked in CW metrics
- [ ] **8 alarms** OK baseline; firing on injected failures
- [ ] **Cost per session** within budget (typical: $0.05-0.30 per multi-agent session)

---

## Common gotchas (claude must address proactively)

- **Multi-agent latency** — supervisor + collaborator(s) = 2-3× single-agent latency. Sonnet supervisor + Sonnet collaborator = 5-15s typical. Use Haiku for collaborators when domain is simple.
- **Routing accuracy ceiling** — supervisor instruction must be explicit about when to use which collaborator. Generic "route appropriately" gives 60-70% routing; specific examples give 85-95%.
- **`SUPERVISOR_ROUTER` vs `SUPERVISOR` mode** — ROUTER picks one; SUPERVISOR invokes all in sequence. ROUTER is right default.
- **`relay_conversation_history: TO_COLLABORATOR`** gives collaborators context but increases token usage 2-3×. For simple Q&A, set DISABLED.
- **Confirmation flow returns control to caller** for mutating ops. Inference Lambda must handle ROC events + render confirmation UI.
- **Multi-tenant via session attributes** — supervisor receives but must explicitly pass to collaborators via promptSessionAttributes. Verify via trace.
- **Cost** — per session = N × InvokeModel calls × M collaborators. Multi-agent easily 5-20× single-agent cost. Monitor.
- **Memory configuration is preview** — APIs may evolve. Use feature flag.
- **Action Group Lambda response shape strict** — `responseBody.TEXT.body` for function-style. Schema mismatch = silent error.
- **Inference Lambda timeout** — agent invoke can take 15s+ for complex routing. Set Lambda timeout ≥ 30s; API Gateway 29s max → use streaming.
- **Streaming chunks are partial JSON** — concat then parse final.
- **Trace capture in prod** is expensive — disable `enableTrace=true` in prod or sample (10%).
- **Per-agent quota** — 100 agents per account default. Multi-tenant agents (1 per tenant) = quota issue at scale. Use shared agents + tenant_id filter.
- **Versioning + alias** — always create version + alias for prod invokes. Never `DRAFT` in prod.
- **Eval drift** — re-run eval suite after every prompt or model change. Track score over time.

---

## Output artifacts

1. **CDK stacks**:
   - `KbStack` (per collaborator) — KB + vector store + IAM
   - `AgentStack` (per collaborator) — agent + actions Lambda + Guardrails
   - `SupervisorStack` — supervisor + collaborator associations + memory config
   - `InferenceStack` — Lambda + API Gateway + WAF + Cognito
   - `EvalStack` — automated agent eval Lambda + CW metrics

2. **Per-collaborator action Lambdas** (5 separate Powertools-based files)

3. **Bedrock Guardrails JSON** (per agent)

4. **Per-agent prompt instructions** (versioned in Prompt Management)

5. **Inference Lambda** (`src/agent_inference/`) — streaming + ROC handling + tenant propagation

6. **React chat UI** — Vite + Cognito + streaming chat + citation render + confirmation modals

7. **Agent eval suite** — 50+ Q&A pairs in YAML; eval Lambda; CW metric publisher

8. **Pytest validation suite** covering all criteria

9. **CloudWatch dashboards**:
   - Multi-agent overview (routing distribution, latency per collaborator)
   - Cost dashboard (token usage per agent per session)
   - Eval accuracy trend

10. **8 alarms YAML** wired to SNS

11. **Runbook**:
    - Adding a new collaborator
    - Tuning supervisor routing prompt
    - Investigating low routing accuracy
    - Cost optimization (model selection per collaborator)

12. **Cost projection** — per session × monthly volume

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-27 | Initial composite template. Supervisor + collaborators + KBs + actions + memory + Code Interpreter + ROC + eval. 4-6 day deploy. Wave 15. |
