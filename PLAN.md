# PLAN.md — Multi-Agent RAG Orchestration

## 1. Context and Goals

This project extends an existing RAG system into a multi-agent orchestration demo. A manager LLM receives a natural-language query and autonomously decides how to route it across two workers: Worker A (the existing RAG endpoint, specialized for grounded factual retrieval) and Worker B (Amazon Bedrock Nova Pro, used for open-ended generative reasoning). The manager synthesizes their outputs into a final answer.

The goal is to demonstrate **autonomous agent-driven orchestration** — the manager decides which workers to call, in what order, and when to stop, based on its own reasoning. The control flow is emergent from the LLM, not pre-defined in infrastructure.

Success criteria: the system answers a small set of representative queries correctly, the manager's reasoning is visible in structured logs, and the routing decisions are sensible (factual queries go primarily to Worker A, creative queries primarily to Worker B, hybrid queries to both).

Out of scope: UI, multi-tenancy, authentication beyond a static API key, persistent learning across queries, local LLM hosting, and any features from the Strands framework (dynamic prompts, sub-agent spawning, self-modification).

## 2. Architecture Overview

The system is request-response with an async submit/poll pattern to accommodate multi-minute wall-clock runs.

```
[Client]
   │
   │  POST /jobs       { "question": "..." }
   ▼
[API Gateway] ──▶ [Submit Lambda] ──▶ [DynamoDB: job_id → PENDING]
                                           │
                                           │ async invoke
                                           ▼
                                  [Worker Lambda]
                                           │
                                           ▼
                                  [Manager Agent (smolagents)]
                                  ── reasoning loop ──
                                           │
                                           │ calls workers as tools
                                           ├─────────────┐
                                           ▼             ▼
                                  [Worker A:        [Worker B:
                                   RAG HTTP]         Bedrock
                                                     Nova Pro]
                                           │
                                           ▼
                                  [DynamoDB: job_id → COMPLETED + answer]

[Client] ──── GET /jobs/{id} ──▶ [Poll Lambda] ──▶ [DynamoDB read]
```

The Worker Lambda runs the manager agent (built on smolagents) as a continuous reasoning loop within a single invocation. The manager freely chooses which workers to call and when to terminate. The Anthropic API is called from within the manager's loop on each reasoning step; it is not a separate component in the request flow.

## 3. Technology Choices and Rationale

**smolagents over Strands.** The agent's task space is bounded (two workers, on-demand queries), the deployment is not long-lived, and there is no need for self-modification, sub-agent spawning, or accumulated learning. Strands' additional capabilities would sit unused while their complexity would still cost. smolagents' simpler, more inspectable agent loop — with fewer framework-level moving parts — is the right fit.

**Anthropic Claude for the manager.** Strong reasoning quality and tool-use reliability matter most for the orchestrator role. The manager makes few but consequential decisions per query.

**Bedrock Nova Pro for Worker B.** Demonstrates AWS-native generative capability; reasonable latency; integrates cleanly via boto3 from within the Worker Lambda.

**Existing RAG endpoint reused as Worker A.** Zero rebuild cost, validates the multi-agent thesis on real infrastructure, and exercises a realistic worker contract (HTTPS in, JSON out).

**Lambda + API Gateway + DynamoDB.** Serverless, low idle cost, appropriate for portfolio-grade on-demand workloads. DynamoDB holds job state (PENDING / RUNNING / COMPLETED / FAILED) and the final answer. CDK is used for infrastructure-as-code.

## 4. Component Breakdown

**Manager agent** — smolagents-based, system prompt describes the two workers and their strengths, runs an autonomous think-act-observe loop until it produces a final answer. Pure Python module (`manager.py`) with no Lambda-specific dependencies.

**Worker A wrapper** — thin function that calls the existing RAG endpoint over HTTPS with the manager's query, returns parsed JSON.

**Worker B wrapper** — thin function that calls Bedrock Nova Pro via boto3 with a prompt constructed by the manager.

**API layer** — two endpoints: `POST /jobs` (submit) and `GET /jobs/{id}` (poll). Authentication via API key in header.

**Persistence layer** — single DynamoDB table keyed by `job_id`, storing state, timestamps, question, answer, and a serialized reasoning trace for observability.

**Logging** — structured JSON to CloudWatch, one log entry per manager step (decision, worker call, worker response, synthesis), tagged with `job_id` for traceability.

## 5. Development Approach

Two-phase workflow:

**Phase 1 — Local iteration.** Application logic (manager, workers, synthesis) developed as plain Python modules and exercised via `python main.py "test question"`. AWS services (RAG endpoint, Bedrock) are called normally over the network; only the Lambda wrapper is absent. This phase optimizes feedback loop speed for prompt engineering and agent behavior tuning. The same `run_query()` function used here is the one Lambda will call.

**Phase 2 — AWS integration.** A thin `lambda_handler.py` wraps `run_query()`, adds DynamoDB writes and structured logging, and is deployed via CDK along with API Gateway and the DynamoDB table. This phase tests integration concerns: API contract, IAM permissions, timeouts, cold starts, and end-to-end submit/poll behavior.

Code is structured so the same logic runs both ways without duplication.

## 6. Validation Strategy

A small set of test queries with expected behaviors:

- **Factual, in-corpus question** (e.g., "What does our policy say about X?") — manager should call Worker A only, return a grounded answer with citations.
- **Open-ended creative question** (e.g., "Suggest three approaches to Y") — manager should call Worker B only.
- **Hybrid question** (e.g., "Given our policy on X, suggest approaches to Y") — manager should call both, synthesize.
- **Ambiguous or unanswerable question** — manager should either ask for clarification (in the answer) or return a confident "I don't know" rather than hallucinate.

For each, verify: final answer quality, the reasoning trace in the logs shows sensible decisions, and total wall-clock time stays within the target budget (Section 7).

## 7. Risks and Open Questions

**Lambda 15-minute timeout.** The existing end-to-end run is ~17 minutes, which exceeds the limit. The plan is to address this via optimization, not architectural decomposition. A pre-defined multi-step Lambda chain is explicitly rejected: it would relocate orchestration logic from the LLM into infrastructure, undermining the project's autonomy thesis. Optimization steps: (1) profile the current run to identify bottlenecks; (2) parallelize independent worker calls (currently sequential); (3) tighten the manager prompt to reduce unnecessary reasoning turns. Target: under 12 minutes wall-clock with tail-latency headroom. Fallback if optimization is insufficient: migrate the Worker Lambda to ECS Fargate (no 15-minute limit) — deferred decision, not pre-committed.

**Tail latency near the ceiling.** Even with optimization, a slow Anthropic response or an unlucky manager reasoning trajectory could push runtime up. Mitigation: log per-step timings, set internal timeouts on individual worker calls, surface near-limit runs as warnings.

**Manager makes a bad routing decision.** The manager might over-call workers (waste tokens) or under-call them (incomplete answer). Mitigation: prompt engineering during Phase 1, validated against the test query set; reasoning traces in logs make bad decisions diagnosable.

**Cold starts.** First invocation after idle may add 5–15 seconds. Acceptable given the multi-minute total runtime; not optimized further.

**Cost.** Anthropic API calls dominate per-query cost; Bedrock and Lambda are secondary. Expected per-query cost: under one USD for typical queries. No budget alerts beyond AWS account defaults for portfolio scope.

## 8. Explicit Non-Goals

- No UI; `curl` is the client.
- No authentication beyond a static API key.
- No multi-tenancy or per-user state.
- No learning or memory across queries; each job is independent.
- No Strands features (dynamic prompts, sub-agent spawning, self-modification, persistent behavioral changes).
- No local LLM hosting; all model inference is via managed APIs.
- No Step Functions or multi-Lambda orchestration; the manager's reasoning loop is the orchestration.
