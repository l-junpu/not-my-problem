# Investigation Result Contract

Status: Accepted for MVP 1

Machine-readable schema: `schemas/investigation-result.v1.schema.json`

## Purpose

This contract defines the versioned output of a successful investigation job. A successful job means that the agent completed its investigation and returned a valid structured report; it does not guarantee that the root cause was identified.

The final result has two layers:

- trusted run metadata assembled by the worker;
- structured findings produced by the investigation agent and validated by the worker.

The agent cannot author or override trusted identifiers, timestamps, commit references, model information, prompt versions, or artifact IDs.

## Conclusions

The `conclusion` field uses one of:

```text
CONFIRMED_ROOT_CAUSE
PROBABLE_ROOT_CAUSE
INCONCLUSIVE
NEEDS_INFORMATION
```

All four conclusions represent a successfully completed investigation when the result is valid. They move the issue to `AWAITING_IMPLEMENTATION`.

From `AWAITING_IMPLEMENTATION`, a maintainer may:

- approve implementation;
- add details and approve reinvestigation;
- reject the issue.

An execution failure, invalid result, timeout, or cancellation is not an inconclusive investigation. It follows the job failure path and returns the issue to `AWAITING_INVESTIGATION`.

## Required Findings

A valid result contains:

- conclusion and concise summary;
- overall confidence between `0` and `1`;
- reproduction status and available reproduction details;
- affected components and repository-relative files;
- a recommended implementation approach and test recommendations;
- risk level, scope, and concerns;
- open questions;
- root-cause description and evidence when the conclusion is confirmed or probable.

Large logs and command output are stored as artifacts and referenced by trusted artifact IDs rather than embedded in the result.

## Controlled Values

Reproduction status:

```text
REPRODUCED
PARTIALLY_REPRODUCED
NOT_REPRODUCED
NOT_ATTEMPTED
```

Evidence type:

```text
CODE
TEST
LOG
COMMAND
ISSUE_CONTEXT
DOCUMENTATION
```

Risk level:

```text
LOW
MEDIUM
HIGH
```

## Reinvestigation Context

Reinvestigation requires explicit maintainer approval. Before approval, the maintainer may add or edit details intended to improve the next investigation's accuracy.

The system:

1. records the new details as an append-only context revision;
2. synchronizes the maintained context section of the GitHub issue;
3. preserves the original report and previous revisions in the audit history;
4. creates a new agent run and job linked to the previous investigation;
5. creates a fresh immutable execution-context snapshot before the new job starts;
6. includes the original report, current GitHub issue, previous investigation result, new maintainer details, and relevant follow-ups in that snapshot.

Editing current issue context does not erase what an earlier investigation received. Each run remains linked to the exact snapshot it used.

The reinvestigation action stores:

```text
requested_by
requested_at
additional_details
previous_run_id
context_revision_id
```

If an associated reinvestigation job is still `QUEUED`, further accepted edits refresh its pending context using the same synchronization and cutoff rules as reporter follow-ups. Once the job changes to `RUNNING`, its snapshot is immutable.

## Validation Rules

- Validate against JSON Schema Draft 2020-12.
- Reject unknown properties.
- Require `schema_version` to equal `1.0`.
- Restrict all confidence values to the inclusive range `0` through `1`.
- Require repository-relative paths and reject absolute paths or parent traversal.
- Verify referenced files exist at the recorded base commit.
- Verify line ranges are positive, ordered, and within the referenced file.
- Require non-empty steps, expected behavior, and observed behavior when status is `REPRODUCED`.
- Require a root-cause description and at least one evidence item for confirmed or probable conclusions.
- Require at least one open question for `NEEDS_INFORMATION`.
- Reject oversized inline strings; detailed output belongs in artifacts.
- Redact detected secrets before persistence.
- Treat all agent-authored strings as untrusted data when rendering Slack or GitHub content.
- Store structured conclusions, not a private model reasoning transcript.

Some semantic checks, including file existence, line bounds, cross-field line ordering, trusted metadata assembly, artifact ownership, and secret redaction, are enforced by worker code in addition to JSON Schema validation.

## Verification Checklist

- A valid fixture exists for every conclusion.
- Confirmed or probable root cause without evidence is rejected.
- `REPRODUCED` without steps, expected behavior, or observed behavior is rejected.
- `NEEDS_INFORMATION` without an open question is rejected.
- Confidence outside `0` through `1` is rejected.
- Absolute paths, parent traversal, nonexistent files, and invalid line ranges are rejected.
- Unknown fields are rejected.
- Agent-supplied trusted metadata cannot override worker metadata.
- Oversized inline logs are rejected.
- A valid inconclusive result is accepted and moves the issue to `AWAITING_IMPLEMENTATION`.
- Reinvestigation requires maintainer approval and creates a new linked run and job.
- The reinvestigation snapshot includes the newly added details and prior investigation result.
- Earlier snapshots and context revisions remain unchanged.
- Invalid results produce no successful issue-state transition.

