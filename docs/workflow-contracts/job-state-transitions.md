# Job State Transition Contract

Status: Accepted for MVP 1

## Purpose

This contract defines the authoritative lifecycle for an agent job. An agent job is a single execution attempt associated one-to-one with an agent run for MVP 1.

PostgreSQL is authoritative for job state. Redis only delivers a job identifier and must not decide job eligibility or state transitions.

## States

| State | Meaning |
|---|---|
| `QUEUED` | The job exists in PostgreSQL and is eligible to be claimed. |
| `RUNNING` | A worker owns a valid lease and is executing the job. |
| `CANCELLING` | Cancellation was requested and the worker must stop at its next safe checkpoint. |
| `SUCCEEDED` | Execution and trusted validation completed successfully. This is terminal. |
| `FAILED` | Execution, validation, or worker recovery failed. This is terminal. |
| `CANCELLED` | Execution stopped because cancellation was requested. This is terminal. |
| `TIMED_OUT` | The configured execution deadline was exceeded. This is terminal. |

There is no separate `CLAIMED` state. Claiming a job atomically changes it from `QUEUED` to `RUNNING` and assigns its worker lease.

## Transition Matrix

| Current state | Event | Next state | Triggered by | Required result or context |
|---|---|---|---|---|
| None | `JOB_CREATED` | `QUEUED` | Orchestrator | Agent run, context snapshot, and outbox record |
| `QUEUED` | `JOB_CLAIMED` | `RUNNING` | Worker | Worker ID, unique lease token, lease expiry, and start time |
| `QUEUED` | `CANCEL_REQUESTED` | `CANCELLED` | Orchestrator | Cancellation actor and reason |
| `RUNNING` | `EXECUTION_SUCCEEDED` | `SUCCEEDED` | Lease-owning worker | Validated result and completed artifacts |
| `RUNNING` | `EXECUTION_FAILED` | `FAILED` | Lease-owning worker | Error, logs, and available artifacts |
| `RUNNING` | `DEADLINE_EXCEEDED` | `TIMED_OUT` | Worker or recovery process | Deadline and preserved artifacts |
| `RUNNING` | `LEASE_EXPIRED` | `FAILED` | Recovery process | Last heartbeat, lease expiry, and recovery reason |
| `RUNNING` | `CANCEL_REQUESTED` | `CANCELLING` | Orchestrator | Cancellation actor, reason, and cancellation deadline |
| `CANCELLING` | `CANCELLATION_ACKNOWLEDGED` | `CANCELLED` | Lease-owning worker | Stop time and preserved artifacts |
| `CANCELLING` | `CANCELLATION_DEADLINE_EXCEEDED` | `CANCELLED` | Recovery process | Forced-termination result and preserved artifacts |

## Worker Lease Rules

Moving a job to `RUNNING` stores:

```text
worker_id
lease_token
lease_expires_at
started_at
last_heartbeat_at
attempt
```

- Claiming must be an atomic compare-and-set from `QUEUED` to `RUNNING`.
- Only the worker holding the current lease token may submit heartbeats or execution results.
- A worker periodically renews its lease and updates `last_heartbeat_at`.
- An expired lease marks the job `FAILED` in MVP 1. It is not automatically requeued.
- A result submitted with an expired or incorrect lease token is rejected.
- The failed job's logs, workspace reference, and artifacts are preserved for diagnosis.

## Cancellation Rules

- Cancelling a `QUEUED` job moves it directly to `CANCELLED`.
- Cancelling a `RUNNING` job moves it to `CANCELLING`.
- The worker checks for cancellation at defined safe checkpoints and during heartbeats.
- After observing cancellation, the worker stops the agent and performs no new commit, push, pull-request, or notification side effects.
- The worker preserves available logs and partial artifacts before acknowledging cancellation.
- If the worker does not acknowledge cancellation before the grace period expires, the recovery process terminates the sandbox and records whether termination was successful.
- Once a job reaches `CANCELLING`, a late success result cannot move it to `SUCCEEDED`.

## Terminal-State and Retry Rules

The following states are terminal and immutable:

```text
SUCCEEDED
FAILED
CANCELLED
TIMED_OUT
```

A terminal job is never reset to `QUEUED` or reused, including for a manual retry.

A maintainer-approved retry creates a new agent run and a new agent job. The new records link to the previous attempt:

```text
retry_of_run_id
retry_of_job_id
```

The new execution-context snapshot may contain new maintainer feedback. The previous attempt retains its original state, context, results, logs, and artifacts.

No failed, timed-out, cancelled, or lease-expired execution is retried automatically in MVP 1. The issue returns to its relevant approval state and requires a maintainer to approve another attempt.

## Delivery Retry Rules

Delivery retries do not create another job and do not count as execution retries:

- The dispatcher may retry publishing the identifier of a `QUEUED` job.
- Redis may redeliver a job identifier before it has been successfully claimed.
- A worker must retrieve the authoritative job record from PostgreSQL before claiming it.
- If the job is no longer `QUEUED`, the worker acknowledges or discards the stale delivery without executing it.
- Atomic claiming ensures that only one worker can move the job to `RUNNING`.

## Relationship to Issue State

When a job terminates, the orchestrator applies the corresponding issue transition in the same completion workflow:

- Successful investigation moves the issue to `AWAITING_IMPLEMENTATION`.
- Failed, cancelled, or timed-out investigation returns it to `AWAITING_INVESTIGATION`.
- Successful implementation moves it to `REVIEWING`.
- Failed, cancelled, or timed-out implementation returns it to `AWAITING_IMPLEMENTATION`.
- Successful review opens the pull request and moves it to `HUMAN_REVIEW`.
- Failed, cancelled, timed-out, or rework-requesting review returns it to `AWAITING_IMPLEMENTATION`.

Job completion and its issue transition must be idempotent. If they cannot be performed in one transaction, the system must use an outbox or equivalent recovery mechanism so they eventually agree.

## General Rules

- The orchestrator creates jobs only after an approved issue transition requires agent work.
- Creating the agent run, job, and outbox message occurs in one database transaction.
- Job events are appended to the immutable audit log.
- Direct job-state updates outside the job transition service are forbidden.
- A job result records the model, agent profile, prompt version, base commit, worker, timing, and artifact references.
- Terminal state does not prevent later append-only artifact-upload metadata, but it prevents changes to execution state or result meaning.

## Verification Checklist

- Every non-terminal state has a defined cancellation path.
- Only one worker can claim a job under concurrent load.
- A worker with a stale lease cannot submit a result.
- Lease expiry produces `FAILED` without automatic execution retry.
- Cancelling a queued job prevents it from executing even if its identifier remains in Redis.
- Cancelling a running job prevents late success from being accepted.
- Repeated completion events do not duplicate issue transitions or notifications.
- Redis redelivery does not create another job or execution.
- A manual retry creates linked new run and job records.
- No terminal job can transition back to an active state.

