# Issue State Transition Contract

Status: Accepted for MVP 1

## Purpose

This contract defines the authoritative lifecycle for an issue. Job execution details such as `QUEUED`, `RUNNING`, `FAILED`, `TIMED_OUT`, and `CANCELLED` belong to the separate job state machine.

Issue transitions may only be performed by the workflow orchestrator after it validates the current state, actor permissions, required data, and idempotency key.

## States

| State | Meaning |
|---|---|
| `TRIAGING` | A new report is being normalized and checked for possible duplicates. |
| `AWAITING_DUPLICATE_REVIEW` | A maintainer must decide whether the report is a duplicate. |
| `AWAITING_INVESTIGATION` | A GitHub issue exists and is waiting for investigation approval or rejection. |
| `INVESTIGATING` | An approved investigation run is active. |
| `AWAITING_IMPLEMENTATION` | Investigation or review is complete and implementation or rework requires maintainer approval. |
| `IMPLEMENTING` | An approved implementation or rework run is active. |
| `REVIEWING` | Deterministic validation and independent AI review are active. |
| `HUMAN_REVIEW` | A pull request is open and awaiting maintainer review. |
| `COMPLETED` | The pull request was merged. This is terminal. |
| `CLOSED` | Automated work has permanently stopped. This is terminal. |

`CLOSED` requires a resolution of `REJECTED` or `DUPLICATE` for MVP 1.

## Transition Matrix

| Current state | Event | Next state | Triggered by | Required result or context |
|---|---|---|---|---|
| None | `REPORT_RECEIVED` | `TRIAGING` | Slack intake | Persisted report and external-event ID |
| `TRIAGING` | `NO_DUPLICATE_CANDIDATE` | `AWAITING_INVESTIGATION` | Orchestrator | Newly created GitHub issue |
| `TRIAGING` | `DUPLICATE_REVIEW_REQUESTED` | `AWAITING_DUPLICATE_REVIEW` | Orchestrator | Candidate issues and scoring evidence |
| `TRIAGING` | `TRIAGE_FAILED` | `AWAITING_INVESTIGATION` | Orchestrator | Fallback GitHub issue and recorded failure |
| `AWAITING_DUPLICATE_REVIEW` | `DUPLICATE_CONFIRMED` | `CLOSED` | Maintainer | Selected issue; report appended to it as a comment; resolution `DUPLICATE` |
| `AWAITING_DUPLICATE_REVIEW` | `NEW_ISSUE_APPROVED` | `AWAITING_INVESTIGATION` | Maintainer | Newly created GitHub issue |
| `AWAITING_DUPLICATE_REVIEW` | `REJECTED` | `CLOSED` | Maintainer | Rejection reason; resolution `REJECTED` |
| `AWAITING_INVESTIGATION` | `INVESTIGATION_APPROVED` | `INVESTIGATING` | Maintainer | New investigation run and job |
| `AWAITING_INVESTIGATION` | `REJECTED` | `CLOSED` | Maintainer | Rejection reason; resolution `REJECTED` |
| `INVESTIGATING` | `INVESTIGATION_SUCCEEDED` | `AWAITING_IMPLEMENTATION` | Orchestrator | Validated investigation result |
| `INVESTIGATING` | `INVESTIGATION_FAILED` | `AWAITING_INVESTIGATION` | Orchestrator | Preserved failure and artifacts |
| `INVESTIGATING` | `RUN_CANCELLED` | `AWAITING_INVESTIGATION` | Orchestrator | Preserved cancellation audit event |
| `AWAITING_IMPLEMENTATION` | `IMPLEMENTATION_APPROVED` | `IMPLEMENTING` | Maintainer | New implementation request, run, and job |
| `AWAITING_IMPLEMENTATION` | `REINVESTIGATION_APPROVED` | `INVESTIGATING` | Maintainer | Additional details, new context revision, and new investigation run and job |
| `AWAITING_IMPLEMENTATION` | `REJECTED` | `CLOSED` | Maintainer | Rejection reason; resolution `REJECTED` |
| `IMPLEMENTING` | `IMPLEMENTATION_SUCCEEDED` | `REVIEWING` | Orchestrator | Commit and deterministic validation output |
| `IMPLEMENTING` | `IMPLEMENTATION_FAILED` | `AWAITING_IMPLEMENTATION` | Orchestrator | Preserved failure and artifacts |
| `IMPLEMENTING` | `RUN_CANCELLED` | `AWAITING_IMPLEMENTATION` | Orchestrator | Preserved cancellation audit event |
| `REVIEWING` | `REVIEW_PASSED_AND_PR_OPENED` | `HUMAN_REVIEW` | Orchestrator | Passing review result and created pull request |
| `REVIEWING` | `REWORK_REQUESTED` | `AWAITING_IMPLEMENTATION` | Orchestrator | Review findings stored as rework feedback |
| `REVIEWING` | `REVIEW_FAILED` | `AWAITING_IMPLEMENTATION` | Orchestrator | Preserved failure and artifacts |
| `REVIEWING` | `RUN_CANCELLED` | `AWAITING_IMPLEMENTATION` | Orchestrator | Preserved cancellation audit event |
| `HUMAN_REVIEW` | `CHANGES_REQUESTED` | `AWAITING_IMPLEMENTATION` | GitHub webhook | Review comments stored as rework feedback |
| `HUMAN_REVIEW` | `PR_MERGED` | `COMPLETED` | GitHub webhook | Pull request and merge commit identifiers |
| Any non-terminal state | `REJECTED` | `CLOSED` | Maintainer | Rejection reason; resolution `REJECTED` |

## Duplicate Review Rules

- Duplicate detection only proposes candidates; it never makes the final decision.
- Candidate review occurs in a configured maintainer Slack channel.
- The review must show the original report, candidate issues, matching signals, and model confidence.
- Confirming a duplicate appends the new report to the selected GitHub issue as a comment before closing the internal issue.
- Marking it as new creates a separate GitHub issue and proceeds to `AWAITING_INVESTIGATION`.

## Implementation and Rework Rules

`AWAITING_IMPLEMENTATION` covers both initial implementation and rework. The distinction is stored on an append-only implementation request:

```text
INITIAL_IMPLEMENTATION
REWORK
```

Each implementation request stores:

- the maintainer who approved it;
- approval time;
- maintainer feedback entered for that run;
- review findings and deterministic validation output, when applicable;
- GitHub review comments, when applicable;
- the previous run and commit, when applicable.

New feedback is appended in a new implementation request and must not overwrite feedback from earlier runs. An implementation job receives the specific approved request as part of its immutable execution-context snapshot.

Reinvestigation also permits a maintainer to add or edit current issue details before approval. The system preserves the original report and prior investigation, synchronizes the maintained context to GitHub, and gives the new run a fresh immutable snapshot containing the additional details. Earlier context revisions and run snapshots remain unchanged.

## General Rules

- `COMPLETED` and `CLOSED` are terminal in MVP 1.
- Cancelling a run does not close or reject the issue. It returns the issue to the relevant approval state.
- A failed or timed-out job does not make the issue terminal. It returns the issue to the relevant approval state and preserves failure details.
- A transition and any job/outbox records it creates must be committed in one database transaction.
- Repeating an event with the same idempotency key must return the original outcome without creating another transition or job.
- The orchestrator records every accepted transition in the immutable workflow event log.
- Direct updates to `workflow_state` outside the transition service are forbidden.

## Verification Checklist

- Every state is reachable through at least one valid transition.
- Only `COMPLETED` and `CLOSED` have no normal outgoing transitions.
- Every active agent state defines success, failure, and cancellation behavior.
- Duplicate confirmation always requires a maintainer.
- Initial implementation and every rework run require maintainer approval.
- Rework feedback remains available after later implementation cycles.
- Concurrent approvals cannot create more than one conflicting active run.
- Replayed Slack and GitHub events do not cause duplicate transitions.
