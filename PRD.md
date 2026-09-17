# Product Requirements Document — not-my-problem

## 1. Product Overview

`not-my-problem` is an internal workflow automation system for converting user-reported bugs and simple feature requests into structured GitHub issues, AI-assisted investigation and implementation jobs, automated code review, and pull requests.

The system is designed around a human-in-the-loop workflow. Users report issues through Slack, while maintainers approve or reject workflow transitions. AI agents only perform work after explicit approval and operate inside isolated workspaces on dedicated branches.

The product must separate:

- issue/workflow state,
- agent execution state,
- queue delivery,
- code execution,
- and human approval.

The workflow orchestrator is the central authority responsible for deciding when jobs should be created and queued. The job queue itself remains a passive delivery mechanism.

---

## 2. Goals

The system should:

- Allow users to report bugs and suggest small feature requests through Slack.
- Convert unstructured Slack reports into normalized issue data.
- Detect likely duplicate GitHub issues before creating new tickets.
- Create or update GitHub issues automatically.
- Notify maintainers when an issue requires review or approval.
- Allow maintainers to trigger workflow actions from Slack.
- Queue AI investigation, implementation, and review jobs only after explicit workflow approval.
- Execute AI jobs in isolated workspaces using dedicated Git branches or worktrees.
- Prevent agents from directly modifying the main working copy.
- Validate builds, tests, file scope, and repository policy before accepting agent output.
- Use a separate AI review step before creating a pull request.
- Notify maintainers when a pull request is ready for human review.
- Maintain an auditable record of issue state, job state, agent runs, outputs, commits, and failures.

---

## 3. Non-Goals

The initial version does not need to:

- Automatically merge pull requests.
- Allow agents unrestricted repository access.
- Allow agents direct access to GitHub credentials.
- Replace deterministic tests or CI checks with LLM judgment.
- Support arbitrary infrastructure changes or production deployments.
- Require Kubernetes, Kafka, or other large-scale distributed infrastructure.

---

## 4. High-Level Architecture

```text
                              ┌────────────────────┐
                              │       Slack        │
                              │                    │
                              │ Reports / Commands │
                              └─────────┬──────────┘
                                        │
                         events / buttons / commands
                                        │
                                        ▼
                              ┌────────────────────┐
                              │  Slack API Gateway │
                              └─────────┬──────────┘
                                        │
                                        ▼
                         ┌───────────────────────────┐
                         │   Workflow Orchestrator   │
                         │                           │
                         │ - State machine           │
                         │ - Permissions             │
                         │ - Issue triage            │
                         │ - Policy checks           │
                         │ - Job creation            │
                         └───────┬─────────┬─────────┘
                                 │         │
                    workflow     │         │ GitHub API /
                    state        │         │ webhooks
                                 ▼         ▼
                     ┌───────────────┐   ┌────────────────┐
                     │  PostgreSQL   │   │     GitHub     │
                     │               │   │                │
                     │ Issues        │   │ Issues         │
                     │ Workflow      │   │ Branches       │
                     │ Agent Runs    │   │ PRs            │
                     │ Job Status    │   │ CI / Checks    │
                     └───────┬───────┘   └───────┬────────┘
                             │                   │
                             │                   │ webhooks
                             │                   │
                             └─────────┬─────────┘
                                       │
                                       ▼
                              ┌────────────────────┐
                              │ Workflow Events    │
                              │                    │
                              │ INVESTIGATE        │
                              │ IMPLEMENT          │
                              │ REVIEW             │
                              │ REJECT             │
                              │ CANCEL             │
                              └─────────┬──────────┘
                                        │
                                        ▼
                              ┌────────────────────┐
                              │     Job Queue      │
                              │                    │
                              │ investigation jobs │
                              │ implementation jobs│
                              │ review jobs        │
                              └─────────┬──────────┘
                                        │
                                  worker claims
                                        │
                                        ▼
                              ┌────────────────────┐
                              │   Agent Worker     │
                              └─────────┬──────────┘
                                        │
                                        ▼
                           ┌─────────────────────────┐
                           │  Disposable Sandbox     │
                           │                         │
                           │ Git Worktree            │
                           │ Pi Coding Agent         │
                           │ Build / Tests           │
                           └────────────┬────────────┘
                                        │
                               result / commits
                                        │
                                        ▼
                              ┌────────────────────┐
                              │   Trusted Worker   │
                              │                    │
                              │ Validate result    │
                              │ Push branch        │
                              │ Update job state   │
                              └─────────┬──────────┘
                                        │
                                        ▼
                              ┌────────────────────┐
                              │ Review Pipeline    │
                              └─────────┬──────────┘
                                        │
                                  PASS / FAIL
                                   ┌────┴────┐
                                   │         │
                                   ▼         ▼
                                Rework      PR
                                             │
                                             ▼
                                          GitHub
                                             │
                                             ▼
                                           Slack
```

---

## 5. Core Design Principles

### 5.1 Workflow state is not queue state

Issues are persistent workflow entities and must not live inside the job queue.

An issue may remain in `AWAITING_APPROVAL` for days without any queued agent work.

Jobs are temporary execution requests created only when the workflow requires an agent action.

### 5.2 The orchestrator creates jobs

The job queue does not inspect GitHub or decide that a ticket changed state.

Instead:

1. Slack or GitHub sends an event.
2. The workflow orchestrator validates the requested state transition.
3. The orchestrator updates PostgreSQL.
4. If the new workflow state requires AI work, the orchestrator creates an `AgentRun` / `AgentJob`.
5. The orchestrator publishes the job ID to the queue.
6. A worker claims the queued job and performs the work.

### 5.3 The queue is passive

The queue should primarily transport a small identifier such as:

```json
{
  "job_id": 812
}
```

The worker should retrieve authoritative job details from PostgreSQL.

### 5.4 Agents are untrusted execution components

Agents may:

- read the assigned repository workspace,
- edit permitted files,
- build and test code,
- create local commits.

Agents should not:

- possess GitHub credentials,
- merge branches,
- modify the main working tree,
- modify forbidden files,
- change workflow state directly.

Trusted worker components perform validation and GitHub operations.

---

## 6. Workflow States

Recommended issue lifecycle:

```text
NEW
 ↓
TRIAGING
 ↓
AWAITING_APPROVAL
 ├──────────────→ REJECTED
 │
 ↓
INVESTIGATING
 ↓
INVESTIGATION_COMPLETE
 ↓
AWAITING_IMPLEMENTATION_APPROVAL
 ├──────────────→ REJECTED
 │
 ↓
IMPLEMENTING
 ↓
VALIDATING
 ↓
REVIEWING
 ├──────────────→ REWORK_REQUIRED
 │                    │
 │                    └────→ IMPLEMENTING
 │
 ↓
READY_FOR_PR
 ↓
PR_OPEN
 ↓
HUMAN_REVIEW
 ├──────────────→ REWORK_REQUIRED
 │
 ↓
MERGED
```

Optional terminal states:

```text
REJECTED
CANCELLED
FAILED
DUPLICATE
MERGED
```

---

## 7. Job States

Agent jobs use a separate lifecycle:

```text
QUEUED
 ↓
RUNNING
 ↓
SUCCEEDED
```

Alternative terminal states:

```text
FAILED
CANCELLED
TIMED_OUT
```

A single issue may have multiple jobs over its lifetime.

Example:

```text
Issue #143
 │
 ├── Job #812  INVESTIGATION     SUCCEEDED
 ├── Job #829  IMPLEMENTATION    FAILED
 ├── Job #831  IMPLEMENTATION    SUCCEEDED
 └── Job #840  REVIEW            SUCCEEDED
```

---

## 8. User Workflow — Report New Issue

```text
[ Report New Issue ]
      │
      ▼
User posts bug / feature request in Slack
      │
      ▼
Slack Event
      │
      ▼
Workflow Orchestrator
      │
      ├─ parse / normalize report
      ├─ classify bug / feature / question
      ├─ search for possible duplicate issues
      │
      ├─ duplicate found?
      │      │
      │      ├─ YES
      │      │    ├─ update existing GitHub issue
      │      │    ├─ add reporter context / comment
      │      │    ├─ mark workflow result as DUPLICATE
      │      │    └─ notify review channel
      │      │
      │      └─ NO
      │           ├─ create new GitHub issue
      │           ├─ create workflow record in PostgreSQL
      │           └─ set Issue → AWAITING_APPROVAL
      │
      ▼
Slack Review Notification
      │
      ├─ [ Investigate ]
      ├─ [ Reject ]
      └─ [ View Issue ]
```

No agent job is created while the issue is in `AWAITING_APPROVAL`.

---

## 9. User Workflow — Investigate

```text
[ Investigate ]
      │
      ▼
Slack interaction
      │
      ▼
Workflow Orchestrator
      │
      ├─ validate current state
      ├─ validate user permissions
      ├─ ensure no conflicting active run exists
      ├─ update Issue → INVESTIGATING
      ├─ snapshot relevant issue context
      ├─ create AgentRun
      └─ enqueue INVESTIGATION job
                 │
                 ▼
             Job Queue
                 │
                 ▼
              Worker
                 │
                 ▼
            Agent Sandbox
                 │
                 ▼
       Investigation Result
                 │
                 ▼
       Workflow Orchestrator
                 │
                 ├─ mark job SUCCEEDED / FAILED
                 └─ update issue workflow state
```

Successful investigation should produce a structured result containing at minimum:

- probable root cause,
- affected files/components,
- reproduction notes where available,
- recommended implementation approach,
- expected risk/scope,
- confidence level.

The issue then transitions to `INVESTIGATION_COMPLETE` or `AWAITING_IMPLEMENTATION_APPROVAL`.

---

## 10. User Workflow — Implement Fix

```text
[ Implement Fix ]
      │
      ▼
Workflow Orchestrator
      │
      ├─ validate investigation exists
      ├─ validate workflow state
      ├─ update Issue → IMPLEMENTING
      ├─ create AgentRun
      └─ enqueue IMPLEMENTATION job
                 │
                 ▼
             Job Queue
                 │
                 ▼
              Worker
                 │
                 ▼
         Disposable Sandbox
                 │
                 ├─ create/reuse isolated worktree
                 ├─ launch implementation agent
                 ├─ modify permitted files
                 ├─ build
                 ├─ run tests
                 └─ create local commit
                 │
                 ▼
          Trusted Worker
                 │
                 ├─ validate changed files
                 ├─ validate tests/build
                 ├─ validate repository policy
                 └─ push permitted branch
```

---

## 11. User Workflow — Review

```text
[ Review ]
      │
      ▼
Workflow Orchestrator
      │
      ├─ validate implementation result
      ├─ update Issue → REVIEWING
      ├─ create AgentRun
      └─ enqueue REVIEW job
                 │
                 ▼
             Job Queue
                 │
                 ▼
          Review Worker
                 │
                 ▼
           Review Agent
                 │
          ┌──────┴──────┐
          │             │
        PASS           FAIL
          │             │
          ▼             ▼
   READY_FOR_PR   REWORK_REQUIRED
          │             │
          │             └──→ implementation workflow
          ▼
       Create PR
          │
          ▼
        GitHub
          │
          ▼
 Slack PR Notification
```

The review agent should receive the issue, investigation result, implementation diff, and deterministic validation output.

Where practical, it should not receive the implementation agent's reasoning transcript in order to encourage an independent review.

---

## 12. User Workflow — Reject

Reject is a workflow action and does not require an agent job.

### 12.1 Reject Before Investigation

```text
[ Reject ]
      │
      ▼
Slack interaction
      │
      ▼
Workflow Orchestrator
      │
      ├─ validate user permissions
      ├─ validate current state
      ├─ optionally require rejection reason
      ├─ update Issue → REJECTED
      ├─ store rejection metadata
      ├─ optionally update GitHub issue label/status
      └─ notify relevant Slack channel
```

Suggested metadata:

```text
rejected_by
rejected_at
rejection_reason
previous_state
```

Possible rejection reasons:

- not reproducible,
- out of scope,
- insufficient information,
- duplicate,
- not worth implementing,
- invalid request,
- manual handling required,
- other.

### 12.2 Reject After Investigation

```text
Investigation Complete
      │
      ├─ [ Implement Fix ]
      └─ [ Reject ]
                │
                ▼
        Workflow Orchestrator
                │
                ├─ preserve investigation report
                ├─ update Issue → REJECTED
                ├─ store rejection reason
                └─ notify reporter / review channel
```

The investigation history must remain available for audit and future reconsideration.

### 12.3 Reject During or After Review

If the implementation itself is unacceptable, the maintainer should distinguish between:

```text
REWORK_REQUIRED
```

and:

```text
REJECTED
```

`REWORK_REQUIRED` means the requested feature/fix remains valid but the implementation must change.

`REJECTED` means no further automated work should continue unless the issue is explicitly reopened.

### 12.4 Rejection With an Active Job

If rejection occurs while an agent job is already running:

```text
[ Reject ]
      │
      ▼
Workflow Orchestrator
      │
      ├─ Issue → REJECTED
      ├─ AgentRun → CANCELLED / CANCELLING
      ├─ invalidate queued job if not started
      └─ signal active worker if already running
```

The worker should stop at the next safe cancellation point and must not push additional changes after cancellation.

---

## 13. User Workflow — Cancel

```text
[ Cancel ]
      │
      ▼
Workflow Orchestrator
      │
      ├─ validate current state
      ├─ mark AgentRun → CANCELLED / CANCELLING
      ├─ invalidate queued job if unclaimed
      ├─ signal worker if already running
      └─ update workflow state appropriately
```

`Cancel` differs from `Reject`:

- `Cancel` stops the current execution.
- `Reject` is a workflow decision that the issue should no longer proceed.

---

## 14. Duplicate Detection

Duplicate detection should use a two-stage approach.

### Stage 1 — Candidate Retrieval

Create embeddings from issue information such as:

- title,
- description,
- error messages,
- stack traces,
- labels/components.

Use PostgreSQL with `pgvector` to retrieve a small set of likely matching issues.

### Stage 2 — LLM Classification

Provide the new report and likely candidate issues to a triage model.

Expected structured output:

```json
{
  "classification": "duplicate",
  "issue": 143,
  "confidence": 0.91,
  "reason": "Same failure mode in EntityManager::LoadEntities"
}
```

The system should use configurable confidence thresholds.

Example policy:

```text
confidence >= 0.90
    → update likely duplicate automatically

0.65 <= confidence < 0.90
    → require maintainer review

confidence < 0.65
    → create new issue
```

Thresholds should be configurable and tuned using real project data.

---

## 15. Agent Roles

The system should support specialized agent profiles.

### 15.1 Triage Agent

Responsibilities:

- normalize Slack reports,
- classify bug/feature/question,
- extract structured information,
- assist with duplicate classification,
- generate a clean GitHub issue description.

### 15.2 Investigation Agent

Responsibilities:

- inspect repository context,
- reproduce the issue where possible,
- identify probable root cause,
- identify affected components/files,
- propose an implementation plan.

### 15.3 Implementation Agent

Responsibilities:

- operate only within assigned workspace,
- modify permitted files,
- implement the approved fix/feature,
- build the project,
- execute configured tests,
- create local commits.

### 15.4 Review Agent

Responsibilities:

- independently inspect the implementation,
- compare implementation against issue requirements,
- inspect the diff,
- inspect deterministic build/test results,
- identify regressions or missing tests,
- recommend PASS or REWORK_REQUIRED.

---

## 16. Workspace and Sandbox Requirements

Each agent execution must use an isolated workspace.

Recommended layout:

```text
/repos/<repository>/
/workspaces/<issue>-<run-id>/
```

Example:

```text
/repos/simulation-engine/
/workspaces/issue-143-4f91/
```

The workspace should be created using a dedicated Git worktree and branch, for example:

```text
ai/issue-143/run-4f91
```

The sandbox should expose only required resources.

Recommended mounts:

```text
/workspace     read/write
/tooling       read-only
/pi-config     read-only
```

Credentials should not be exposed directly to the sandbox where avoidable.

---

## 17. Trusted Worker Responsibilities

The trusted worker sits between the untrusted agent and GitHub.

It should:

- claim queued jobs,
- create sandboxes/worktrees,
- start Pi with the appropriate profile,
- monitor execution,
- capture logs/results,
- run or verify deterministic validation,
- inspect changed files,
- enforce repository policy,
- create commits where required,
- push approved branches,
- report completion/failure to the orchestrator.

The worker should own GitHub credentials rather than the agent process.

---

## 18. Repository Policy

Each repository should provide explicit agent constraints.

Example:

```yaml
build:
  command: cmake --build build

test:
  command: ctest --test-dir build

agent:
  writable:
    - src/**
    - tests/**

  requires_approval:
    - CMakeLists.txt
    - Dockerfile
    - database/**

  forbidden:
    - .github/**
    - deployment/**
    - credentials/**
```

The trusted worker must validate changed files before pushing results.

---

## 19. Validation Pipeline

AI review must not be the only quality gate.

Recommended implementation validation sequence:

```text
Implementation Agent
        ↓
Build
        ↓
Unit / Integration Tests
        ↓
Static Analysis
        ↓
Formatting / Lint Checks
        ↓
Repository Policy Check
        ↓
Review Agent
        ↓
PASS / REWORK_REQUIRED
```

For C++ repositories, possible deterministic checks include:

- CMake build,
- CTest,
- clang-format,
- clang-tidy,
- project-specific tests,
- repository-specific validation scripts.

---

## 20. GitHub Integration

Prefer a dedicated GitHub App rather than a personal access token.

Expected capabilities:

- read/write issues,
- read/write pull requests,
- read/write repository contents where required,
- read checks/CI results,
- receive issue, pull request, review, comment, check, and push webhooks.

GitHub should remain the source of truth for:

- issue numbers,
- branches,
- commits,
- pull requests,
- CI/check results.

PostgreSQL remains the source of truth for the internal automation workflow.

---

## 21. Slack Integration

Slack acts as the human-facing control surface.

### Reporter Channel

Used for:

- bug reports,
- feature requests,
- follow-up information.

### Maintainer / Review Channel

Used for:

- issue approval,
- investigation results,
- implementation approval,
- review results,
- failures,
- pull request notifications.

Recommended actions:

```text
[ Investigate ]
[ Implement Fix ]
[ Review ]
[ Reject ]
[ Cancel ]
[ Re-investigate ]
[ View Issue ]
[ View PR ]
```

Slash commands may be supported for advanced users, but buttons/interactions should be the primary workflow interface.

---

## 22. Suggested PostgreSQL Entities

Initial tables:

```text
users
repositories
reports
issues
agent_runs
agent_jobs
agent_events
```

### `issues`

Suggested fields:

```text
id
github_issue_number
repository_id
workflow_state
created_at
updated_at
```

### `agent_runs`

Suggested fields:

```text
id
issue_id
repository_id
job_type
status
branch
workspace
worker_id
model
agent_profile
prompt_version
started_at
finished_at
commit_sha
result_json
error
```

### `agent_jobs`

Suggested fields:

```text
id
agent_run_id
priority
queue_state
created_at
claimed_at
completed_at
```

### `agent_events`

Suggested fields:

```text
id
agent_run_id
timestamp
event_type
payload
```

---

## 23. Execution Context Snapshot

When an agent job is created, the orchestrator should snapshot relevant context.

Suggested contents:

```text
github_issue_number
github_issue_updated_at
title
description
comments
labels
repository
base_branch
base_commit_sha
requested_action
investigation_result
```

Before execution, the worker may compare the stored `github_issue_updated_at` against GitHub and refresh context when required.

---

## 24. Logging and Artifact Storage

PostgreSQL should store metadata and workflow state.

Large logs and run artifacts should use filesystem or object-style storage.

Example:

```text
/data/runs/<run-id>/
    agent.log
    build.log
    test.log
    review.json
    investigation.md
    patch.diff
```

Artifacts should remain traceable to their associated `AgentRun`.

---

## 25. Suggested Technology Stack

### Backend / Orchestrator

- Python
- FastAPI
- Pydantic
- SQLAlchemy

### Database

- PostgreSQL
- pgvector

### Queue

Initial implementation:

- Redis-backed job queue or equivalent lightweight queue

The queue implementation should remain replaceable behind an internal abstraction.

### Slack

- Slack Bolt

### GitHub

- GitHub App
- GitHub REST API
- GitHub Webhooks

### AI Agents

- Pi Coding Agent
- dedicated triage / investigation / implementation / review profiles

### Local Inference

- existing local vLLM endpoints

### Isolation

Initial:

- Git worktrees

Target:

- disposable Docker sandbox around each worktree/agent run

### Observability

- structured JSON logs
- run-level event history
- worker/job status tracking

---

## 26. Reliability Requirements

The system must:

- prevent duplicate active jobs for the same workflow action,
- make workflow transitions idempotent,
- safely retry failed queue delivery,
- safely retry workers without duplicating commits or PRs,
- preserve failed job logs,
- detect orphaned `RUNNING` jobs,
- support cancellation,
- recover workflow state after backend restart,
- validate that a job still applies before executing it.

---

## 27. Security Requirements

The system must:

- authenticate Slack interactions,
- validate GitHub webhook signatures,
- enforce user permissions before workflow changes,
- isolate agent execution,
- prevent agents from accessing secrets where possible,
- use least-privilege GitHub credentials,
- prevent agents from merging directly into the default branch,
- enforce permitted/forbidden file policies,
- retain an audit trail of automated changes.

---

## 28. MVP Scope

The first milestone should intentionally exclude autonomous code modification.

### MVP 1

```text
Slack report
    ↓
Triage / normalization
    ↓
Duplicate detection
    ↓
GitHub issue creation
    ↓
AWAITING_APPROVAL
    ↓
[ Investigate ] / [ Reject ]
    ↓
Investigation Job Queue
    ↓
Sandboxed Pi Investigation
    ↓
Structured Investigation Report
    ↓
Slack notification
```

### MVP 2

Add:

- implementation approval,
- isolated implementation agent,
- worktree/branch management,
- build/test execution,
- local commits,
- trusted worker validation.

### MVP 3

Add:

- independent review agent,
- rework workflow,
- automated branch push,
- pull request creation,
- PR notifications.

### MVP 4

Add:

- richer repository policies,
- better duplicate detection,
- dashboards/metrics,
- multiple worker scheduling,
- run prioritization,
- stronger container isolation.

---

## 29. Success Criteria

The product is successful when a maintainer can reliably perform the following workflow:

```text
User reports issue in Slack
      ↓
System creates or updates GitHub issue
      ↓
Maintainer approves investigation
      ↓
Agent investigates in isolation
      ↓
Maintainer approves implementation
      ↓
Agent implements on isolated branch
      ↓
Deterministic validation passes
      ↓
Independent agent review passes
      ↓
System creates pull request
      ↓
Maintainer reviews and merges manually
```

At every stage, the system must retain human control, isolate agent execution, and preserve an auditable history of what happened and why.
