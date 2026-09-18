# Implementation Plan — not-my-problem

## 1. Purpose

References `PRD.md` to plan out a concrete incremental implementation plan. Each step has a verification gate and should be completed before work begins on the next step.

The initial implementation is a modular Python application with separately runnable API, dispatcher, and worker processes. PostgreSQL is the authoritative store for workflow and job state. Redis is a passive delivery mechanism and carries only job identifiers.

## 2. Initial Decisions

### 2.1 Duplicate detection without a vector database

The initial product will not use embeddings or a vector database. Duplicate detection will use two stages:

1. PostgreSQL candidate retrieval using:
   - exact matches for external references, error codes, exception names, and normalized stack frames;
   - filters and boosts for repository, component, and labels;
   - PostgreSQL full-text search over the issue title and description;
   - `pg_trgm` similarity for misspellings and similar titles.
2. LLM classification of only the highest-ranked candidates.

Candidate retrieval must be deterministic and inspectable. Each candidate should include its component scores so maintainers can understand why it was selected.

Example combined score:

```text
0.45 * full_text_rank
+ 0.30 * title_trigram_similarity
+ 0.15 * exact_signal_score
+ 0.10 * component_label_score
```

The weights and LLM confidence thresholds must be configuration rather than hard-coded policy. If no candidate passes a low retrieval threshold, the system creates a new issue without calling the duplicate-classification model.

This approach is intentionally suitable for the initial expected scale. Revisit embeddings only if measured recall becomes inadequate as the issue corpus grows.

### 2.2 Initial operating assumptions

- One Slack workspace and one GitHub repository for MVP 1.
- Docker Compose for local development.
- PostgreSQL with the `pg_trgm` extension; no `pgvector` dependency.
- Redis Streams for passive job delivery.
- A static configurable Slack maintainer allowlist for MVP 1.
- Linux investigation workers.
- AI providers are accessed through an internal interface so tests can use deterministic fakes.
- Agents never receive Slack or GitHub credentials.

## 3. Proposed Project Layout

```text
src/not_my_problem/
    api/             # Slack and GitHub HTTP endpoints
    application/     # Workflow commands and use cases
    domain/          # State machines and policy
    infrastructure/  # PostgreSQL, Redis, GitHub, Slack, AI clients
    workers/         # Trusted worker and job executors
    agents/          # Versioned prompts and output schemas
tests/
    unit/
    integration/
    contract/
migrations/
docker-compose.yml
Makefile
```

The same codebase is deployed as separate processes:

- `api`: Slack/GitHub gateway and workflow orchestrator;
- `dispatcher`: publishes transactional outbox messages to Redis;
- `worker`: claims jobs and runs trusted execution;
- PostgreSQL;
- Redis.

## 4. MVP 1 Implementation Steps

### Step 1 — Define workflow contracts

Create explicit, version-controlled definitions for:

- issue states and transitions;
- job states and transitions, including `CANCELLING`;
- reporter, maintainer, worker, and administrator permissions;
- success, failure, rejection, and cancellation outcomes;
- investigation-result JSON schema;
- Slack and GitHub event idempotency rules.

The accepted transition contracts are maintained in:

- `docs/workflow-contracts/issue-state-transitions.md`;
- `docs/workflow-contracts/job-state-transitions.md`;
- `docs/workflow-contracts/roles-and-permissions.md`;
- `docs/workflow-contracts/investigation-result.md`;
- `docs/workflow-contracts/schemas/investigation-result.v2.schema.json` (with v1 retained for historical compatibility).

At minimum, define transitions equivalent to:

```text
AWAITING_APPROVAL + INVESTIGATE -> INVESTIGATING
AWAITING_APPROVAL + REJECT      -> REJECTED
INVESTIGATING + RUN_SUCCEEDED   -> AWAITING_IMPLEMENTATION_APPROVAL
INVESTIGATING + RUN_FAILED      -> AWAITING_APPROVAL
INVESTIGATING + CANCEL          -> AWAITING_APPROVAL
```

Verification gate:

- Every PRD state is reachable or explicitly terminal.
- Every active state defines success, failure, cancellation, and rejection behavior.
- Each transition identifies its permitted actor roles.
- Product acceptance tests can be derived directly from the transition table.

### Step 2 — Scaffold the application

Create the Python package, FastAPI application, typed settings, structured JSON logging, database migrations, Docker Compose services, and CI workflow.

Establish these commands:

```bash
make up
make migrate
make test
make lint
make typecheck
```

Verification gate:

- `GET /health/live` succeeds without checking dependencies.
- `GET /health/ready` verifies PostgreSQL and Redis.
- A fresh checkout starts using the documented command.
- All five commands above pass locally and in CI.

### Step 3 — Implement the domain state machine

Implement transitions as domain commands rather than direct updates to a state column. Each command includes the actor, expected state, action, idempotency key, timestamp, and optional reason.

Persist an immutable workflow event for every accepted transition.

Verification gate:

- Unit tests cover every allowed transition.
- Unit tests cover forbidden and stale-state transitions.
- Repeating a command with the same idempotency key returns the original result.
- Concurrent `INVESTIGATE` commands produce only one active investigation run.

### Step 4 — Create the database schema

Implement the PRD entities and add:

- `workflow_events`;
- `external_events`;
- `outbox_messages`;
- `artifacts`;
- `issue_context_snapshots`.

Add unique constraints for Slack event IDs, GitHub delivery IDs, and command idempotency keys. Add a partial unique constraint that prevents conflicting active runs for an issue.

Enable PostgreSQL's `pg_trgm` extension and add full-text and trigram indexes for duplicate candidate retrieval.

Verification gate:

```bash
make db-reset
make migrate
make migration-check
make integration
```

- Migrations upgrade a clean database successfully.
- A disposable database can be downgraded and upgraded again.
- Database constraints reject invalid and duplicate records.
- Query plans use the expected search indexes on a representative fixture set.

### Step 5 — Implement reliable queue delivery

In the same database transaction that approves work:

1. Create the agent run.
2. Create the agent job.
3. Create an outbox message containing only `job_id`.

The dispatcher publishes pending outbox messages to Redis. Workers retrieve authoritative details from PostgreSQL and atomically claim the job there.

Verification gate:

- Killing the dispatcher after a database commit does not lose the job.
- Publishing an outbox message twice does not run the job twice.
- A worker crash leaves the job recoverable.
- Stale `RUNNING` jobs are detected and handled by an explicit recovery policy.

### Step 6 — Implement the GitHub adapter

Create an adapter interface and a test fake, followed by GitHub App support for issue search, issue creation, duplicate comments, labels, issue URLs, and webhook signature validation.

Verification gate:

- Contract tests pass against the fake adapter.
- Invalid webhook signatures return `401`.
- Replayed webhooks cause no duplicate effects.
- A sandbox repository receives the expected issue body and labels.

### Step 7 — Implement Slack report intake

Add Slack request signature and timestamp validation, immediate event acknowledgement, background processing, configurable channels, and storage of message/thread references.

Verification gate:

- Invalid signatures and stale timestamps are rejected.
- Replayed events are ignored safely.
- A valid report creates exactly one internal report.
- A sandbox Slack message is acknowledged within Slack's required response window.

### Step 8 — Implement triage

Define and validate a versioned triage result containing at least:

```json
{
  "kind": "bug",
  "title": "...",
  "description": "...",
  "component": "...",
  "reproduction": [],
  "errors": [],
  "confidence": 0.0
}
```

Malformed model output must never directly change workflow state. Model failures should retain the report and route it to manual review.

Verification gate:

- Valid fixtures pass schema validation.
- Missing, malformed, and unexpected fields fail validation safely.
- Model timeout and unavailable-provider tests preserve the report.
- Prompt and schema versions are recorded with each result.

### Step 9 — Implement duplicate detection

Normalize searchable content, extract exact signals, retrieve candidates with full-text and trigram queries, calculate an inspectable combined score, and send only the best candidates to the classification model.

Start with configuration equivalent to:

```text
LLM confidence >= 0.90       -> automatically mark duplicate
0.65 <= confidence < 0.90   -> maintainer duplicate review
confidence < 0.65            -> create a new issue
```

These thresholds must be tuned against labeled examples before production use.

Verification gate:

- Create a labeled fixture set containing clear duplicates, near matches, and unrelated issues.
- Measure candidate-retrieval recall separately from LLM classification accuracy.
- Known duplicates appear within the configured candidate limit.
- Exact error identifiers strongly rank their matching issues.
- Unrelated reports fall below the retrieval threshold.
- The returned explanation exposes retrieval signals and LLM reasoning summary.

### Step 10 — Complete report-to-approval

Connect Slack intake, triage, duplicate detection, GitHub issue creation, workflow persistence, and the Slack review notification.

Slack buttons should reference an opaque internal action token and must not be treated as authoritative workflow state.

Verification gate:

1. Post a new report in sandbox Slack.
2. Confirm a GitHub issue is created.
3. Confirm the internal issue is `AWAITING_APPROVAL`.
4. Confirm Slack shows `Investigate`, `Reject`, and `View Issue`.
5. Confirm no agent job exists yet.
6. Replay the Slack event and confirm nothing is duplicated.
7. Submit a known duplicate and confirm the configured duplicate path is used.

### Step 11 — Implement approval and rejection

Validate the Slack signature, maintainer identity, role, workflow state, action token, and absence of conflicting runs. Rejection stores the actor, reason, timestamp, and previous state without creating an agent job.

Verification gate:

- An unauthorized reporter cannot approve work.
- Double-clicking `Investigate` creates one run and one job.
- `Reject` creates no job.
- A stale action returns a clear response and makes no state change.

### Step 12 — Implement the investigation worker

The trusted worker:

1. claims and revalidates the job;
2. snapshots issue and repository context;
3. creates an isolated worktree at the recorded base commit;
4. launches the agent without external-service credentials;
5. validates its structured result;
6. stores logs and artifacts;
7. completes the run and workflow transition;
8. notifies Slack.

For MVP 1, the repository should be read-only to the agent where practical. Any changed files must cause validation failure.

Verification gate:

- The agent cannot access Slack or GitHub credentials.
- The main checkout remains unchanged.
- Unexpected workspace changes fail validation.
- Cancellation stops execution at a safe checkpoint.
- A crash preserves logs and produces the expected failed/recoverable state.
- A successful report contains a conclusion, concise summary, and confidence. Confirmed or probable conclusions also contain a diagnosis with suspected files and a reason for each; reproduction notes and open questions are included only when applicable.
- Trusted run metadata and artifact relationships are recorded by the worker outside the agent-authored result.

### Step 13 — Run the MVP 1 acceptance suite

Exercise the complete path:

```text
Slack report
-> triage
-> duplicate check
-> GitHub issue
-> maintainer approval
-> queued investigation
-> isolated execution
-> structured result
-> Slack notification
```

Inject failures at Slack, GitHub, AI provider, database, queue, dispatcher, and worker boundaries.

MVP 1 exit criteria:

- All transitions and external effects are auditable.
- Replayed external events cause no duplicate effects.
- Restarting the API, dispatcher, or worker loses no accepted work.
- Agents receive no external-service credentials.
- Maintainers can reject or cancel work safely.
- The full happy path passes repeatedly in the sandbox integrations.

## 5. Later Milestones

### MVP 2 — Implementation

Add writable worktrees, repository policy parsing, changed-file validation, build/test execution, local commits, cancellation checkpoints, and trusted branch pushes.

### MVP 3 — Review and pull requests

Add independent review jobs, rework loops, pull request creation, CI status reconciliation, and Slack PR notifications.

### MVP 4 — Hardening and scale

Add container isolation, richer policies, multiple workers, scheduling and priority, operational dashboards, artifact retention controls, and tuning based on duplicate-detection measurements.

Each milestone must reuse the same workflow, idempotency, outbox, artifact, and audit foundations.

## 6. Decisions and Open Questions

Resolved:

- MVP 1 supports one configured GitHub repository. The data model retains `repository_id` so multi-repository support can be added without redesigning issue and run relationships.
- MVP 1 runs the API, dispatcher, and worker on one local machine as separate processes or containers. Remote worker deployment is deferred, while process boundaries and interfaces remain explicit.

Resolve these before completing Step 1:

1. What GitHub issue volume should duplicate retrieval be tested against?
2. Which local vLLM model and API contract will be used for triage and investigation?
3. How long must logs, prompts, patches, and investigation artifacts be retained?
