# NEXUS Product Requirements Document

**Product:** NEXUS — AI Operations & Decision Platform  
**Version:** 1.0 review candidate  
**Date:** 5 October 2026  
**Phase:** 0 — Product and Architecture  
**Owner:** Project owner  
**Repository location:** `docs/product/PRD.md`  
**Status:** Complete for product review; not yet approved or frozen.

## 1. Product definition

NEXUS turns a business objective into a controlled workflow that analyzes operational data, recommends permitted actions, obtains approval when required, executes those actions, verifies their results, and records the evidence.

The first release focuses on **Operations Exception Resolution**: identifying customer orders at risk of missing delivery dates and managing the interventions needed to address them. It uses simulated order and production data initially, while keeping integration boundaries suitable for later business-system connections.

The product must demonstrate both business usefulness and engineering depth. A manager should understand which orders need attention and what happened next. An engineer should be able to inspect persistent execution state, authorization decisions, tool calls, retries, verification evidence, and measured outcomes.

The operating loop is **Goal → Plan → Analyze → Decide → Approve → Execute → Verify → Measure → Audit**. Approval applies according to policy; audit events are recorded throughout the loop.

## 2. Baseline and decisions in this version

The supplied project conversation establishes the product name, first use case, six specialist agents, controlled orchestration model, technology stack, and phased delivery sequence. This PRD carries those decisions forward.

The following are proposed product defaults introduced here to make implementation testable: role permissions, exact action boundaries, demo dataset size, freshness and approval expiry rules, performance targets, evaluation thresholds, and the restricted workflow-editor scope. They become the baseline when this PRD is approved. They are requirements and targets, not claims about an existing implementation.

Detailed service design, database tables, endpoint contracts, authentication implementation, model selection, and risk formulas belong in the remaining Phase 0 documents.

## 3. Problem and desired outcome

Operations teams can have order, production, inventory, and dispatch information without a reliable process for acting on exceptions. Staff must assemble evidence, identify bottlenecks, coordinate owners, obtain decisions, and check whether the intended action happened. Recommendations without execution tracking leave the same coordination work unresolved.

NEXUS should give each exception a traceable journey from detection to intervention and eventual outcome. It must make uncertainty visible, keep people accountable for restricted decisions, and distinguish successful automation from actual improvement in delivery performance.

| Problem | Required product outcome |
| --- | --- |
| At-risk orders are identified late or inconsistently | A prioritized inbox with reproducible risk indicators and source timestamps |
| Staff repeatedly assemble the same evidence | One exception view with facts, calculations, missing data, and recommendations |
| Recommendations lack ownership | A named owner, proposed action, deadline, and approval status |
| Actions happen without adequate control | Server-enforced permissions and action-specific approval rules |
| A successful API response is mistaken for business success | Separate execution, verification, and business-outcome states |
| Management cannot assess automation value | Metrics tied to completed runs, explicit denominators, and labeled estimates |

## 4. Users and permissions

The MVP is one organization with workspace-scoped data and multiple users. Public signup, billing, and commercial multi-tenant administration are outside the MVP. Authorization must still prevent access to a workspace where a user has no membership.

| Role | Main responsibility | Permitted actions |
| --- | --- | --- |
| Administrator | Configure the product | Manage users, workspace membership, integrations, model settings, policies, and workflow versions; inspect workspace activity |
| Operations Manager | Own operational decisions | Review exceptions, run workflows, assign owners, approve or reject permitted business actions, and review outcomes |
| Operations Analyst | Investigate and coordinate | Import data, start workflows, review evidence, propose interventions, and work assigned tasks; cannot approve restricted actions |
| Executive Viewer | Monitor performance | Read executive dashboards, exceptions, and authorized run summaries; cannot initiate writes or approve actions |
| Auditor | Review accountability | Read and export authorized audit and execution records; cannot change business or system state |

Roles can be combined explicitly. Administrator access does not automatically confer Operations Manager approval authority. For approval-required actions, the initiating user cannot approve their own run's action. The demo must include separate initiator and approver accounts. AI agents and integration credentials never hold human approval authority.

## 5. MVP scope

### Required for the first complete release

- Authenticated web application with server-side role and workspace enforcement.
- Seeded demonstration data and validated CSV imports for the order-risk workflow.
- Goal entry constrained to the supported workflow, with explicit order scope and time horizon.
- Operations inbox, exception detail, and ownership tracking.
- Persistent, versioned workflows with bounded agent and tool execution.
- Six specialist capabilities: Planner, Data Analyst, Risk, Decision, Automation, and Verification.
- Deterministic policy enforcement, approval queue, expiry, rejection, and resumption.
- Internal task creation, approved sandbox email, and approved simulated ERP priority updates.
- n8n integration for at least one external action with authenticated completion handling.
- Verification based on observable records, rather than model assertions.
- Run timeline, AI Control Room, append-only application audit records, and measurable usage.
- Workflow editor limited to the supported node types and approved tools.
- Repeatable demo, evaluation fixtures, failure-recovery tests, and deployment documentation.

### Explicitly deferred

Sales lead prioritization and finance invoice exceptions are later workflows. Also deferred: real ERP write access, payments, purchasing, order cancellation, autonomous machine allocation, customer delivery promises, unrestricted SQL or code execution, arbitrary tool creation, arbitrary workflow loops, parallel graph execution, marketplace connectors, SSO, billing, mobile apps, RAG, vector search, fine-tuning, and multimodal processing.

The architecture may support extension points, but deferred features do not block this MVP.

## 6. First complete user journey

1. An analyst selects a validated data snapshot and enters: “Identify customer orders at risk of missing their delivery date and resolve the highest-risk exceptions.”
2. NEXUS displays the interpreted scope, analysis horizon, snapshot time, and supported action boundaries. The analyst starts the run.
3. The Planner produces a structured plan using the published workflow and allowed capabilities. Unsupported objectives receive a clear explanation and no execution.
4. The Data Analyst retrieves scoped records and uses deterministic calculations to compare remaining work, available capacity, material availability, and delivery dates.
5. The Risk capability produces delivery-risk indicators, evidence, data-quality flags, and possible causes. Uncertain causes are labeled as hypotheses.
6. The Decision capability proposes a permitted intervention, its intended result, owner, and rationale. The policy engine determines whether execution is automatic, requires approval, or is blocked.
7. The manager reviews any approval request with the exact action, destination, payload, evidence, and expected effect. Approval authorizes only that action version.
8. The executor dispatches the action with a stable idempotency key. An external action can pass through n8n, but NEXUS remains authoritative for authorization and run state.
9. The Verification capability reads the target system or receipt and checks the expected result. A mismatch creates an escalation rather than a success label.
10. The dashboard and audit timeline update. Creating an escalation task is recorded as a verified intervention; the order remains open until subsequent evidence supports resolution.

Each action has its own status and approval. In a run covering multiple orders, one pending approval must not prevent other independent approved or automatic actions from progressing. The engine may process ready actions sequentially; parallel graph execution is not required.

## 7. Data requirements

### Required input

| Data group | Minimum fields |
| --- | --- |
| Orders | External order ID, customer reference, article or SKU, ordered quantity, completed quantity, unit, order status, required delivery date |
| Production | Order ID, planned completion date or production schedule, available daily capacity, capacity unit, production status, observed timestamp |
| Materials | Order or article link, material availability status, expected availability date when short, observed timestamp |
| Dispatch assumptions | Shipping lead time, dispatch cutoff or working-calendar convention, destination reference where needed |
| Ownership and provenance | Responsible department or owner, source identifier, source record ID, snapshot ID, import time, source update time, workspace timezone |

The seed pack must contain these related datasets. MVP CSV imports follow fixed documented templates; free-form Excel interpretation is deferred. Optional fields include order value, currency, customer priority, supplier reference, and shortage quantity. Missing optional fields must not be invented.

### Validation and freshness

- Validate field types, nonnegative quantities, completed quantity not exceeding ordered quantity, units, date ordering, required relationships, and duplicate IDs.
- Show a validation report before committing an import. Reject the batch if required rows are invalid; do not silently discard records or mix partial imports with the previous snapshot.
- Importing the same batch again must not create duplicate orders or exceptions. A newer valid snapshot is preserved as a distinct version.
- Missing capacity or material information produces an “insufficient evidence” flag. It must not be interpreted as zero risk.
- Preserve source dates separately from import dates. Reimporting old data does not make it fresh.
- Proposed demo default: operational observations older than 24 hours are stale. Analysis may continue with a warning; write actions depending on stale facts are blocked pending refresh. Replay datasets use an explicit simulated clock with a visible “Demo” label.
- Store timestamps consistently and display the workspace timezone. Date-only delivery commitments are interpreted using a configured business cutoff and calendar, not an assumed midnight deadline.
- Send only the minimum necessary fields to the model. Credentials and unnecessary customer contact details must not enter prompts or traces.

## 8. Functional requirements

All requirements below are MVP requirements unless explicitly marked as future work.

| ID | Requirement | Acceptance evidence |
| --- | --- | --- |
| FR-01 | Authenticate users and enforce role/workspace permissions on every API and event stream | Unauthorized reads, writes, approvals, exports, and workspace switching are denied in integration tests |
| FR-02 | Import, validate, version, and select operational snapshots | Invalid imports show row-level errors; repeat imports do not duplicate records |
| FR-03 | Start a supported goal as an asynchronous run | UI receives a run ID promptly; duplicate submission keys return the same run |
| FR-04 | Maintain an operations inbox with owner, due date, delivery risk, data quality, action state, and age | Filters for owner, status, urgency, customer, and order return matching records; repeated detection links to existing open exceptions |
| FR-05 | Produce structured, evidence-linked analysis and recommendations | Each displayed conclusion references source records or is explicitly an inference; numerical calculations are reproducible |
| FR-06 | Enforce action policy independently of model output | Model-generated approval claims and unauthorized tools cannot cause execution |
| FR-07 | Support approve, reject, expire, and resubmit | Approval records capture authorized actor, timestamp, action version, and decision; rejection requires a reason |
| FR-08 | Execute allowlisted actions with safe recovery | Duplicate dispatches do not duplicate supported target effects; ambiguous outcomes are reconciled before retry |
| FR-09 | Verify each action's postcondition | Target evidence is stored; missing evidence cannot produce a verified status |
| FR-10 | Persist run and step state across interruptions | Worker restart resumes from durable state without replaying verified side effects |
| FR-11 | Publish immutable workflow versions | Existing runs retain their original definition; edits create a new version |
| FR-12 | Provide a restricted visual workflow editor | Administrator can duplicate a template, configure allowed nodes, validate connections, and publish a new version |
| FR-13 | Show execution and agent activity | Run detail exposes stage, timestamps, sanitized inputs/outputs, tools, model, usage, retries, approval decisions, and errors |
| FR-14 | Record application audit events | Data imports, permissions, policy changes, publishing, decisions, approvals, actions, verification, and cancellations are traceable |
| FR-15 | Report operational and model metrics | Dashboard values reconcile with underlying records and show period, denominator, environment, and missing measurements |
| FR-16 | Configure bounded integrations and execution limits | Admin can disable an integration, set budgets and limits, and see failed health checks without exposing secrets |

## 9. Risk and approval policy

**Delivery risk and action risk are different.** Delivery risk prioritizes an order; action risk controls what NEXUS may do. A high delivery-risk order may need only a low-impact internal task. A low delivery-risk order still requires approval before customer communication.

Delivery risk uses a documented, versioned rule model with reproducible calculations. Its formula and thresholds will be defined in the agent architecture. Any displayed 0–100 score is a heuristic prioritization score, not a calibrated probability of delay. Model confidence is not authorization and is not presented as measured accuracy.

| Action | MVP policy | Required verification |
| --- | --- | --- |
| Read permitted operational data and calculate risk | Automatic within authorized scope | Source references and calculation outputs |
| Create a NEXUS internal escalation task | Automatic when prerequisites pass and duplicate suppression applies | Read back task ID, owner, linked exception, and requested fields |
| Draft customer communication | Automatic draft only | Saved draft linked to the exception |
| Send email to an allowlisted test recipient | Manager approval required | Sandbox provider acceptance record with message ID and intended recipient; never label this as proof of delivery or reading |
| Change priority in the simulated ERP | Manager approval required | Read back the exact order and new priority |
| Cancel an order, spend money, change a customer commitment, or modify a real ERP | Blocked in MVP | Record attempted action and escalation; approval cannot override the block |
| Unknown tool, missing permission, stale prerequisite, or insufficient evidence | Blocked pending remediation | Explicit reason and required next step |

Proposed approval expiry is 24 hours. An approval binds workspace, run, action ID, action payload, target, source version, and policy version. Payload or relevant evidence changes invalidate it. At dispatch, the system rechecks current permissions, policy, expiry, and prerequisites. A stricter current policy wins over a previous approval.

Approval decisions are atomic: duplicate clicks or concurrent approve/reject requests produce one effective decision and a clear response to the other request. Expiry and rejection cause no side effect. A changed proposal creates a new action version and new approval request.

## 10. Workflow and execution behavior

The Workflow Engine owns durable business-process state and transitions. The Agent Runtime provides bounded reasoning and tool proposals. Agents cannot modify the workflow definition, grant approval, or directly bypass the tool gateway.

Supported node types are `TRIGGER`, `AGENT`, `TOOL`, `CONDITION`, `APPROVAL`, `WAIT`, `NOTIFICATION`, and `END`. The editor supports an acyclic graph with a single trigger, valid reachable endings, conditional branches, and configured finite waits. Loops, concurrent branches, arbitrary code, and arbitrary tools are excluded. Validation rejects paths that could bypass mandatory policy enforcement; the gateway independently enforces policy even if graph validation fails.

Runs expose created, queued, running, waiting for approval or timer, verifying, completed, failed, cancelled, timed out, and escalated conditions. “Resumed” is an event returning work to a runnable state. Precise enums and transition rules belong in the workflow-engine specification.

Each order produces separate exception and action records. A run is completed only when all required actions are verified or explicitly determined unnecessary. Rejected or unresolved required interventions produce an escalated outcome with partial results visible. A successful run does not automatically mark every business exception resolved.

Retries apply only to classified transient failures and have bounded attempts and elapsed time. Validation failures, rejected approvals, and blocked permissions are not automatically retried. A timeout after dispatch is an unknown outcome, not proof that an action failed. The system checks the target by idempotency key or external record ID before any further dispatch; if it cannot establish the outcome, it escalates.

Cancelling a run prevents new dispatches. Already dispatched actions may complete and must still be reconciled and audited. The UI must not promise rollback of external effects. A resumed process and a separately requested new run remain distinguishable.

## 11. Specialist agent responsibilities

| Capability | Output | Boundary |
| --- | --- | --- |
| Planner | Structured plan and supported scope | Uses only published workflow capabilities; no arbitrary process generation |
| Data Analyst | Facts, calculations, anomalies, and data-quality flags | Uses scoped read tools and deterministic calculation functions; no unrestricted SQL |
| Risk | Prioritization and supporting evidence | Uses the versioned rule model; cannot grant authorization |
| Decision | Permitted action proposal, alternatives, and concise rationale | Cannot invent tools, commitments, or missing business facts |
| Automation | Authorized tool execution request and result summary | Calls pass through policy and idempotency controls; no unrestricted credentials |
| Verification | Postcondition result with observable evidence | Deterministic checks decide success; model text can explain the result |

An orchestrator invokes these specialists for bounded tasks. Separate responsibilities do not require a separate model call for every step; deterministic code should handle arithmetic and objective checks. Structured outputs must be schema-validated. After bounded repair attempts, invalid output escalates rather than executing a guessed action.

Observability exposes concise rationale, evidence, tool calls, and outcomes. Hidden model chain-of-thought is neither required nor treated as an audit artifact. Source data and external text are untrusted content and cannot alter system permissions or approval rules.

## 12. User interface requirements

| Screen | Required content and interactions |
| --- | --- |
| Login | Authentication, session-expired handling, actionable errors |
| Executive Dashboard | Active runs, pending approvals, open exceptions, verified actions, failures, and labeled impact estimates |
| Operations Inbox | Search/filter, priority, owner, delivery deadline, freshness, risk, and next action |
| Exception Detail | Order facts, evidence, calculations, proposed intervention, ownership, action history, and outcome history |
| Workflow Builder | Template canvas, allowed node palette, configuration, validation errors, draft/published versions |
| Workflow Runs | Status filters, initiating user, workflow version, start time, elapsed time, and result summary |
| Run Detail | Stage timeline, per-order results, approval/action status, errors, retries, cancellation, and verification evidence |
| Approval Center | Exact proposed change, recipient/target, justification, evidence time, expiry, approve/reject controls |
| AI Control Room | Agent activity derived from actual runs, model usage, latency, failure and retry information |
| Integrations | Environment, connection status, allowed actions, last error, and enable/disable controls |
| Audit Logs | Filterable actor/action/resource timeline and permission-controlled export |
| Analytics | Metric definitions, date range, denominators, assumptions, and drill-down to underlying runs |
| Settings | Users, memberships, policies, models, budgets, timeouts, and workspace timezone |

The interface is desktop-first and usable at tablet widths. Status uses text and icons as well as color. Forms support keyboard navigation and visible focus. Every data-driven screen needs loading, empty, permission-denied, stale-data, and recoverable-error states. Demo data and simulated outcomes must be visibly labeled. Reconnecting the UI must recover run state without launching another run.

The Lovable phase may use mocked data and interactions. Before the product release, every required action must use real backend behavior; unsupported controls must be visibly disabled rather than appearing functional.

## 13. Nonfunctional requirements

These are initial acceptance targets to measure on a documented test environment.

| Area | Requirement and acceptance target |
| --- | --- |
| Capacity | Validate imports of 1,000 orders; support 10 concurrent users and 5 active runs, with excess work queued |
| API latency | p95 below 1 second for ordinary indexed reads/writes and below 2 seconds for run submission at the reference load; exclude import processing, model inference, and external service duration |
| UI visibility | Persisted state changes appear within 5 seconds under normal connectivity; show disconnect status and refresh on reconnection |
| Analysis time | Analyze the 15-order reference pack within 120 seconds excluding human wait and external outages; measure and report actual results |
| Durability | Accepted runs, approvals, and actions survive worker restart; PostgreSQL retains authoritative state independently of queue state |
| Recovery | Demonstrate checkpoint recovery and duplicate suppression under repeated jobs, timeouts, and duplicate callbacks |
| Security | Backend authorization, encrypted transport in deployment, protected server-side secrets, validated input, bounded tools, rate limits, and redacted logs |
| Cost control | Enforce administrator-configured per-run call/token/spend limits; reserve estimated cost before dispatch and reconcile actual usage; unknown price data must not appear as zero cost |
| Audit | Application users cannot edit or delete audit events; restricted database administration and backups protect history without claiming cryptographic immutability |
| Backup | Before an external pilot, demonstrate database backup and restore; proposed recovery targets are at most 24 hours of data loss and 4 hours to restore |
| Retention | Proposed pilot defaults: 30 days for sanitized detailed model traces and 90 days for operational/audit records, followed by controlled policy-based retention processing |
| Deployability | Document environment variables, migrations, health checks, seed setup, worker startup, and rollback procedure; production secrets never enter source control |

No uptime SLA or real-world accuracy claim is established by this PRD. Infrastructure sizing, model limits, retention suitability, and measured performance must be reviewed before a real-data pilot.

## 14. Success metrics

### Engineering release gates

| Metric | Definition | Proposed target |
| --- | --- | --- |
| Policy enforcement | Prohibited or unapproved write attempts prevented / attempts in the release test suite | 100%; zero authorization bypasses |
| Duplicate suppression | Duplicate external effects during supported retry/callback tests | Zero |
| Verification coverage | Actions labeled verified that have valid postcondition evidence / all actions labeled verified | 100% |
| Risk classification quality | Precision and recall against independently reviewed reference labels | At least 90% each for actionable delay risk; report both and raw counts |
| Unsupported-input handling | Missing/stale/unsupported cases safely flagged / such test cases | 100% in the reference suite |
| Audit completeness | Required audited transitions with linked records / required transitions | 100% |
| Recovery correctness | Defined interruption scenarios recovering or escalating without unauthorized/duplicate effects | All critical scenarios pass |

Build at least 50 labeled evaluation cases spanning normal orders, late production, material shortage, insufficient capacity, missing data, stale observations, contradictory facts, and adversarial source text. Keep evaluation cases separate from prompt examples and record dataset, policy, prompt, and model versions. Report poor results plainly and fix failures before declaring the gate passed.

### Business measures

- **Time to triage:** time from eligible exception detection to a reviewable recommendation; report median and p95.
- **Approval latency:** decision timestamp minus request timestamp, separately from active processing time.
- **Verified execution rate:** verified actions divided by terminal attempted actions in the selected period; show pending and unknown outcomes separately.
- **Resolution rate:** exceptions meeting the business-resolution criteria divided by exceptions eligible for follow-up; a created task alone does not qualify.
- **Estimated hours saved:** matched manual handling baseline minus measured human handling time, with sample size and assumptions shown. Do not treat model wall-clock runtime as human time.
- **Estimated net value:** estimated hours saved × a disclosed hourly cost assumption, minus attributable model and infrastructure costs. Keep currencies separate unless a dated conversion method is specified.

Initial business target: at least 30% lower median human handling time on matched demo cases, without worsening decision correctness. This is a pilot hypothesis, not an achieved saving. Real delivery impact requires subsequent operational evidence and cannot be inferred from simulated priority changes.

## 15. Demo and acceptance scenarios

The reference demo contains 15 orders covering on-time work, production delays, material shortages, constrained capacity, and insufficient data. All values and outcomes are labeled simulated. Use deterministic fixtures for reliability tests and separately recorded live-model evaluations.

| ID | Scenario | Required result |
| --- | --- | --- |
| AC-01 | Analyst starts the supported goal | Persisted run and plan are visible; source snapshot and workflow version are recorded |
| AC-02 | Order is healthy | No unnecessary action; evidence supports the result |
| AC-03 | Delayed order needs internal escalation | Exactly one task is created and verified; exception remains open for business follow-up |
| AC-04 | Proposed simulated ERP priority change | Execution pauses until a different authorized manager approves |
| AC-05 | Manager rejects or approval expires | No external write; reason and escalation are visible |
| AC-06 | Payload or relevant source facts change after approval | Old approval is unusable; revised action requires a new decision |
| AC-07 | Model proposes a prohibited action or forged approval | Gateway blocks it regardless of model text |
| AC-08 | Data is missing, stale, or contains hostile instructions | Uncertainty is visible; instructions cannot change permissions or trigger unsafe execution |
| AC-09 | Worker dies after target accepts an action | Recovery reconciles target evidence; no duplicate effect |
| AC-10 | n8n sends duplicate, forged, or out-of-order callbacks | Authentication/correlation checks reject invalid callbacks and duplicate events do not repeat actions |
| AC-11 | Target returns success but read-back contradicts it | Action is unverified and escalated; dashboard does not count it as verified |
| AC-12 | Viewer attempts a write or another workspace is requested | Backend denies access and records the appropriate security event |
| AC-13 | Integration is disabled, times out, or becomes unavailable | Bounded recovery or escalation; no infinite retries or silent data loss |
| AC-14 | Model exceeds budget or returns invalid structured output | Controlled stop or escalation with understandable cause |
| AC-15 | Workflow or policy is edited during an active run | Definition version remains pinned; current stricter dispatch policy still applies |
| AC-16 | Run is cancelled while an action is in flight | No further dispatches; in-flight outcome is reconciled and visible |
| AC-17 | Different orders need different decisions | Ready permitted actions progress; waiting and rejected actions retain separate outcomes |
| AC-18 | User refreshes or reconnects | Run state is restored without duplicate submission |

The release is accepted only after critical scenarios pass, measured targets are reported, and remaining limitations are documented. A polished UI or successful happy-path recording alone is insufficient.

## 16. Technical constraints carried forward

| Concern | Agreed baseline |
| --- | --- |
| Frontend | Next.js and TypeScript; Tailwind with a consistent component system |
| Backend | Python and FastAPI |
| Durable data | PostgreSQL |
| Queue and workers | Redis and Celery |
| Agent runtime | OpenAI Agents SDK initially, behind an application-owned model boundary |
| External automation | n8n and controlled REST/webhook integrations |
| Local infrastructure | Docker |
| Source control | GitHub monorepo |
| Deployment target | Railway initially |
| Build workflow | Lovable for UI prototype; Cursor for main implementation; Claude Code for optional review/debugging |

These are inherited project choices, not assertions about current vendor features or pricing. Versions, compatibility, hosting limits, and costs must be verified when writing implementation plans. One model provider is sufficient for MVP; multiple providers are an extension, not a release gate.

The UI prototype must be adapted to the agreed Next.js frontend if its exported framework differs. A prototype's generated backend or database does not silently replace the FastAPI/PostgreSQL architecture. Module boundaries do not require independently deployed microservices.

## 17. Delivery phases and exit conditions

| Phase | Deliverable | Exit condition |
| --- | --- | --- |
| 0 — Product and Architecture | PRD and five supporting specifications | Product rules and technical contracts agree; blocking decisions are resolved |
| 1 — Product UI | Clickable product shell with mock data | Complete user journey and required error/empty states can be reviewed |
| 2 — Engineering Foundation | Auth, RBAC, database, migrations, API, configuration, logging | Frontend persists data through authorized backend operations |
| 3 — Workflow Engine | Durable nodes, transitions, versions, queueing, retries | Deterministic non-AI workflow survives interruptions |
| 4 — AI Engine | Specialists, structured outputs, evidence, model budgets | Reference data produces reviewable proposals; writes remain sandboxed and policy-gated |
| 5 — Human Approval | Approval policy and lifecycle | Restricted actions cannot execute without valid authorization |
| 6 — Automation | n8n, sandbox email, simulated ERP, callback verification | Approved side effects execute and are independently verified |
| 7 — Observability | Timeline, control room, usage, audit, impact metrics | Each displayed outcome can be traced to supporting records |
| 8 — Hardening | Recovery, security, evaluation, load checks, backup exercise | Critical acceptance scenarios and agreed targets pass |
| 9 — Deployment | Documented deployed environment | Health checks and end-to-end smoke test pass after deployment |
| 10 — Portfolio and MD Demo | README, diagrams, test evidence, screenshots, 3–5 minute demonstration | Technical claims and business claims match demonstrated evidence |

Authorization, policy checks, audit foundations, and idempotency begin with the relevant implementation phase; later hardening strengthens them. No interim phase permits restricted live writes before approval enforcement exists.

## 18. Dependencies and risks

| Risk or dependency | Planned response |
| --- | --- |
| Scope expands into a generic enterprise platform | Keep one complete order-risk workflow; defer other departments |
| ERP access is unavailable | Deliver against a documented simulator; later request an authorized read-only API or export before real integration |
| AI produces plausible but unsupported recommendations | Structured outputs, source-linked evidence, deterministic calculations, evaluation, and visible uncertainty |
| External systems do not support idempotency | Use reconciliation where reliable; block automatic retry when the outcome cannot be established |
| Model or integration latency/cost is excessive | Bound context, reuse deterministic calculations, queue work, enforce budgets, and expose measurements |
| Attractive demo overstates impact | Separate simulated data, verified execution, observed business outcomes, and estimated savings |
| Sensitive information enters prompts or logs | Minimize context, redact traces, restrict access, and apply retention policy |
| A single developer cannot maintain the scope | Use a modular application, constrained editor, one provider, and phase gates; defer optional infrastructure |

## 19. Remaining Phase 0 documents

| Document | Decisions it must settle |
| --- | --- |
| `docs/architecture/system-architecture.md` | Service/module boundaries, deployment topology, authentication approach, trust boundaries, data flow, UI event delivery, and failure ownership |
| `docs/architecture/agent-architecture.md` | Agent input/output schemas, tool permissions, risk formula, model configuration, budgets, prompts, and evaluation design |
| `docs/architecture/workflow-engine.md` | Graph validation, run/node/action states, transitions, timeouts, retry classes, approvals, cancellation, reconciliation, and dispatch guarantees |
| `docs/architecture/database-schema.md` | Entities, constraints, workspace isolation, source versions, exception deduplication, action/approval linkage, audit and retention design |
| `docs/api/api-specification.md` | Endpoint contracts, authentication, pagination, errors, idempotency keys, approval conflicts, callbacks, and event schemas |

The system architecture is the next document. UI requirements and roadmap are established here and can be expanded into dedicated specifications later. Credentials, vendor access, and real-data permissions are prerequisites for a real pilot, not blockers for the simulated MVP.

## 20. Freeze criteria and change control

This PRD is ready to freeze when the project owner accepts the MVP scope, proposed role and approval rules, simulated integration boundary, and measurable release gates. Approval has not been presumed.

Phase 0 as a whole remains open until all six documents are consistent and the risk formula, state machine, auth approach, data contracts, and tool authorization contracts are defined. Product approval is not authorization to send real customer messages or modify external production systems.

After freezing, any change to scope, action permissions, risk policy, or success criteria requires a versioned change entry with reason and impact. Implementation may clarify details without silently expanding product authority.

| Version | Date | Change |
| --- | --- | --- |
| 1.0 review candidate | 5 October 2026 | Initial PRD based on the supplied NEXUS planning conversation; adds measurable requirements and proposed implementation defaults |
