# Investigation Result Contract

Status: Accepted for MVP 1

Current machine-readable schema: `schemas/investigation-result.v2.schema.json`

Superseded schema: `schemas/investigation-result.v1.schema.json`

## Purpose

This contract defines the versioned, agent-authored findings from a successful investigation job. A successful job means that the agent completed its investigation and returned a valid structured report; it does not guarantee that the root cause was identified.

Trusted execution data is not part of the agent-authored result. The worker stores run and job identifiers, issue identifiers, commit references, timestamps, model and prompt details, and artifact relationships on their authoritative records. This prevents the agent from authoring trusted metadata and avoids duplicating it inside the report.

## Why Version 2 Is Smaller

Version 1 attempted to represent both an investigation report and its execution record. It also required reproduction details, affected areas, an implementation plan, risk analysis, open questions, and artifact references for every successful investigation. Investigations cannot always establish those facts, so the structure encouraged empty fields or speculative content.

Version 2 changes the contract to:

- store trusted run metadata and artifacts outside the agent payload;
- keep one overall confidence value;
- require only a conclusion, summary, and confidence from the agent;
- make reproduction and open questions conditional on what the investigation learned;
- replace line-level evidence, affected components, implementation steps, and risk analysis with a concise diagnosis and file-level suspected areas;
- require a reason for every suspected file, without brittle line-number references.

The aim is to capture facts useful to the next workflow step without asking the investigation agent to invent a complete implementation plan.

## Result Shape

Every result contains:

- `schema_version`, equal to `2.0`;
- `conclusion`;
- a concise `summary`;
- one overall `confidence` value between `0` and `1`.

The following fields are included only when supported by the investigation:

- `diagnosis`, containing a description and suspected repository files with a reason for each file;
- `reproduction`, containing a status and optional concise notes;
- `open_questions`.

A confirmed or probable root cause requires a diagnosis. `NEEDS_INFORMATION` requires at least one open question. Inconclusive results do not require the agent to speculate about affected files or implementation details.

## Conclusions

The `conclusion` field uses one of:

```text
CONFIRMED_ROOT_CAUSE
PROBABLE_ROOT_CAUSE
INCONCLUSIVE
NEEDS_INFORMATION
```

All four conclusions represent a successfully completed investigation when the result is valid. They move the issue to `AWAITING_IMPLEMENTATION`.

From `AWAITING_IMPLEMENTATION`, a maintainer may approve implementation, add details and approve reinvestigation, or reject the issue.

An execution failure, invalid result, timeout, or cancellation is not an inconclusive investigation. It follows the job failure path and returns the issue to `AWAITING_INVESTIGATION`.

## Reproduction Status

```text
REPRODUCED
PARTIALLY_REPRODUCED
NOT_REPRODUCED
NOT_ATTEMPTED
```

Reproduction is optional because it may not be practical or relevant. When supplied, notes should summarize what was attempted or observed. Detailed command and test output belongs in run artifacts.

## Trusted Run Records and Artifacts

The worker stores the following outside the investigation result:

```text
run_id
job_id
issue_id
github_issue_number
base_commit_sha
issue_updated_at
completed_at
model
agent_profile
prompt_version
artifact references
```

The persisted result remains associated with its run, so consumers can obtain trusted metadata and artifacts through that relationship. Repository paths in a diagnosis are interpreted at the run's recorded base commit.

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
- Require `schema_version` to equal `2.0`.
- Restrict confidence to the inclusive range `0` through `1`.
- Require a diagnosis with at least one suspected file for confirmed or probable conclusions.
- Require a non-empty reason for every suspected file.
- Require at least one open question for `NEEDS_INFORMATION`.
- Require repository-relative, forward-slash paths; reject absolute paths, backslashes, and `.` or `..` path segments.
- Verify suspected files exist at the run's recorded base commit.
- Reject oversized inline strings; detailed output belongs in artifacts.
- Redact detected secrets before persistence.
- Treat all agent-authored strings as untrusted data when rendering Slack or GitHub content.
- Store structured conclusions, not a private model reasoning transcript.

File existence, trusted run metadata, artifact ownership, and secret redaction are enforced by worker code in addition to JSON Schema validation.

## Verification Checklist

- A valid fixture exists for every conclusion.
- Confirmed or probable root cause without a diagnosis is rejected.
- A diagnosis without a suspected file and reason is rejected.
- `NEEDS_INFORMATION` without an open question is rejected.
- Confidence outside `0` through `1` is rejected.
- Absolute paths, backslashes, parent traversal, and nonexistent files are rejected.
- Optional fields may be omitted when the investigation cannot support them.
- Unknown fields are rejected.
- Agent-supplied trusted run metadata is rejected as an unknown field.
- Oversized inline logs are rejected.
- A valid inconclusive result is accepted and moves the issue to `AWAITING_IMPLEMENTATION`.
- Reinvestigation requires maintainer approval and creates a new linked run and job.
- The reinvestigation snapshot includes the newly added details and prior investigation result.
- Earlier snapshots and context revisions remain unchanged.
- Invalid results produce no successful issue-state transition.

## Version 1 Compatibility

Version 1 remains in the repository as a historical contract. New investigation jobs produce version 2. Existing stored version 1 results remain readable and are not rewritten; consumers select the appropriate schema using `schema_version`.
