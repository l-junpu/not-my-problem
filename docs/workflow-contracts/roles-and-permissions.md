# Roles and Permissions Contract

Status: Accepted for MVP 1

## Purpose

This contract defines which human roles and service identities may request or perform actions. All authorization is default-deny. Only the orchestrator may apply issue transitions or create agent work.

## Human Roles

### Reporter

A reporter is an authenticated member of the configured Slack workspace.

A reporter may:

- submit a report;
- add or edit follow-up information for their report before an associated job starts running;
- cause accepted pre-execution follow-up information to be reflected in the GitHub issue;
- view workflow status and links they already have permission to access;
- receive investigation and pull-request notifications.

A reporter may not:

- confirm duplicate candidates;
- approve investigation, implementation, reinvestigation, rework, or retry;
- reject an issue;
- cancel an active job;
- view restricted execution logs or artifacts;
- directly change internal workflow or job state.

### Maintainer

A maintainer is a Slack user whose immutable Slack user ID is in the configured maintainer allowlist.

A maintainer may:

- perform every reporter action;
- confirm a duplicate or request creation of a separate GitHub issue;
- reject a report or issue with a reason;
- approve investigation or reinvestigation;
- approve initial implementation;
- add or edit feedback before approving an implementation attempt;
- approve rework with new feedback;
- cancel queued or active jobs;
- approve a new attempt after failure, timeout, or cancellation;
- view job logs, artifacts, validation output, and audit history;
- review and merge pull requests through their existing GitHub permissions.

A maintainer may approve investigation, implementation, and rework for a report they submitted themselves. These actions receive the same audit treatment as approvals of another user's report.

A maintainer may not:

- directly edit database workflow state;
- reset or reuse a terminal job;
- bypass the state-transition service;
- cause an agent to receive GitHub, Slack, database, or queue credentials.

### Administrator

An administrator may perform every maintainer action, including approving work for their own reports.

An administrator may also:

- manage the maintainer allowlist;
- change repository, Slack channel, model, timeout, and retention configuration;
- pause or resume dispatch of new jobs;
- run documented recovery operations;
- view system-wide operational information.

An administrator does not have permission to force an arbitrary workflow or job state. Administrative recovery must use validated and audited commands.

## Human Permission Matrix

| Action | Reporter | Maintainer | Administrator |
|---|:---:|:---:|:---:|
| Submit report | Yes | Yes | Yes |
| Add/edit pre-execution follow-up | Own report | Yes | Yes |
| Synchronize accepted follow-up to GitHub issue | Own report | Yes | Yes |
| View ordinary workflow status | Yes | Yes | Yes |
| View restricted logs and artifacts | No | Yes | Yes |
| Confirm duplicate | No | Yes | Yes |
| Create issue after duplicate review | No | Yes | Yes |
| Approve investigation | No | Yes | Yes |
| Approve reinvestigation | No | Yes | Yes |
| Approve implementation | No | Yes | Yes |
| Provide implementation/rework feedback | No | Yes | Yes |
| Approve rework | No | Yes | Yes |
| Reject issue | No | Yes | Yes |
| Cancel queued or active job | No | Yes | Yes |
| Approve manual retry | No | Yes | Yes |
| Approve own report | No | Yes | Yes |
| Configure system | No | No | Yes |
| Pause new dispatch | No | No | Yes |
| Force arbitrary state transition | No | No | No |

## Reporter Follow-up Rules

Reporter follow-ups are mutable only until the associated job atomically changes from `QUEUED` to `RUNNING`.

Before that cutoff:

1. The system accepts the reporter's addition or edit.
2. It records the revision and actor in the audit history.
3. It updates the maintained follow-up section of the GitHub issue.
4. It marks the pending execution context as stale.
5. The job cannot be claimed until the GitHub synchronization completes or fails visibly.
6. At claim time, the system creates or refreshes the immutable execution-context snapshot from the current approved issue content.

This permits useful last-minute context without allowing the input to change underneath a running agent.

After the job reaches `RUNNING`:

- the active run's context snapshot remains immutable;
- new follow-up information is still recorded and may be appended to the GitHub issue;
- the information is marked as not included in the active run;
- a maintainer may cancel the run, wait for it to finish, or approve reinvestigation/rework using the new information.

The application only manages the designated reporter-context portion of the GitHub issue. It must not silently overwrite maintainer edits to unrelated title, body, label, or metadata fields.

Direct GitHub edits received before execution are incorporated through the same context-refresh mechanism. Direct GitHub edits received after execution starts do not mutate the active snapshot.

## Service Identities

| Identity | Allowed responsibilities | Explicitly prohibited |
|---|---|---|
| Orchestrator | Validate commands and transitions; update workflow state; create runs, jobs, snapshots, and outbox records; invoke Slack and GitHub adapters | Running agent code or bypassing transition policy |
| Dispatcher | Read pending outbox records; publish `job_id`; record delivery status | Changing issue state or job execution state |
| Worker | Claim jobs; renew leases; manage worktrees and sandboxes; run validation; submit results | Approving work or changing issue state directly |
| Recovery process | Detect expired leases and cancellation deadlines; request defined recovery transitions | Reusing terminal jobs or creating unapproved work |
| Agent process | Read its assigned snapshot and permitted repository content; produce scoped output | Accessing PostgreSQL, Redis, Slack, GitHub, secrets, or the main working tree |
| Webhook handlers | Authenticate and normalize external events; submit commands to the orchestrator | Directly changing database state |

Each service has a distinct credential and receives only the permissions needed for its responsibilities.

## Authorization Rules

- Human authorization uses immutable Slack user IDs, never display names or email addresses.
- Slack signatures and request timestamps are verified before resolving human permissions.
- GitHub webhook signatures are verified before accepting GitHub-originated events.
- Authorization is checked when executing a command, not only when rendering a button.
- A removed role takes effect immediately for new command attempts, including clicks on old Slack buttons.
- Slack buttons contain opaque action references and are not proof that an action remains allowed.
- Agent output is untrusted data and cannot authorize subsequent actions.
- Self-approval by maintainers and administrators is permitted and audited.
- Rejection requires an explicit reason.
- Cancelling a `RUNNING` job requires an explicit reason. A configured default reason may be used for a job that is still `QUEUED`.
- Existing GitHub review and merge permissions remain governed by GitHub.

Every privileged command records:

```text
actor_id
actor_type
effective_role
action
target_type
target_id
timestamp
reason_or_feedback
idempotency_key
```

## Verification Checklist

- An unknown Slack user is denied all privileged commands.
- A reporter can modify only follow-up information associated with their own report.
- A reporter cannot approve, reject, cancel, retry, or view restricted artifacts.
- A maintainer and administrator can approve their own report.
- Removing a maintainer invalidates their old interactive controls.
- Concurrent follow-up and worker-claim operations produce a deterministic cutoff.
- A job cannot be claimed while an accepted pre-execution GitHub synchronization is pending.
- The final execution snapshot contains every follow-up accepted before the claim cutoff.
- Follow-ups accepted after the cutoff do not change the active snapshot.
- Every privileged action records the resolved role and immutable actor ID.
- No human or service identity can force an arbitrary state transition.
- The agent sandbox has no external-service or database credentials.

