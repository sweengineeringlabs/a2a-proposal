# [Proposal] Acceptance Criteria as a first-class Task concept

## Summary

Add a first-class `AcceptanceCriteria` structure to the Task schema, a new
`PENDING_ACCEPTANCE` task state, and two new operations — `AcceptTask` and
`RejectTask` — to support both automated output validation and human-in-the-loop
approval flows before a task is marked `COMPLETED`.

---

## Problem

Currently, `COMPLETED` means the agent stopped working — not that the output
meets the requester's intent. There is no protocol-level mechanism to:

- Declare what "done" looks like before submitting a task
- Gate `COMPLETED` on explicit client or human approval
- Signal back "this output is rejected, retry with this feedback" without
  creating an entirely new task and re-describing the problem from scratch

The only current workaround is for orchestrators to validate artifacts out-of-band
and submit a new task if dissatisfied. This loses task context, breaks
conversation continuity via `contextId`, and makes closed-loop quality control
an orchestrator-level concern rather than a protocol-level guarantee.

---

## Use Cases

**1. Automated pipeline validation**
An orchestrator submits a code-generation task with a schema requiring the
artifact to be valid JSON matching a specific structure. The agent produces
output; before `COMPLETED` is signaled, the output is validated against the
schema. If it fails, the task re-enters `WORKING` with the validation errors
as feedback — no new task required.

**2. Human-in-the-loop approval**
A document-drafting agent produces a report. Before it's sent downstream to
a publishing agent, a human reviewer must explicitly approve it. The task
enters `PENDING_ACCEPTANCE`; the reviewer reads the artifact and either
accepts (→ `COMPLETED`) or rejects with a note (→ back to `WORKING`).

**3. Multi-agent quality gates**
Agent A delegates a task to Agent B. Agent A declares acceptance criteria.
Agent B self-evaluates before transitioning to `PENDING_ACCEPTANCE`, avoiding
a round-trip if it knows the output doesn't meet criteria yet.

---

## Proposed Design

### 1. `AcceptanceCriteria` structure on `Task`

```proto
message AcceptanceCriteria {
  // Human-readable or LLM-interpretable success condition
  optional string description = 1;

  // JSON Schema the artifact Parts must conform to (for automated validation)
  optional google.protobuf.Struct artifact_schema = 2;

  // Artifact names that must be present in the output
  repeated string required_artifacts = 3;

  // Who is responsible for evaluation
  AcceptanceEvaluator evaluator = 4;

  // How long to wait in PENDING_ACCEPTANCE before auto-failing (seconds)
  optional int32 acceptance_timeout_seconds = 5;

  // Max rejection cycles before transitioning to FAILED
  optional int32 max_rejections = 6;
}

enum AcceptanceEvaluator {
  ACCEPTANCE_EVALUATOR_UNSPECIFIED = 0;
  CLIENT = 1;   // orchestrator evaluates programmatically
  AGENT = 2;    // agent self-evaluates before surfacing output
  HUMAN = 3;    // task pauses for explicit human accept/reject
}
```

### 2. New Task State: `PENDING_ACCEPTANCE`

Inserted between `WORKING` and `COMPLETED`. The task has produced an artifact
but awaits explicit acceptance.

Updated lifecycle:

```
SUBMITTED
  └─► WORKING
        ├─► PENDING_ACCEPTANCE ──► COMPLETED  (accepted)
        │         └──────────────► WORKING    (rejected, feedback injected)
        │                          └─► FAILED (max_rejections exceeded)
        ├─► INPUT_REQUIRED
        ├─► AUTH_REQUIRED
        ├─► FAILED
        ├─► CANCELED
        └─► REJECTED
```

`PENDING_ACCEPTANCE` is an interrupted state — like `INPUT_REQUIRED`, it
pauses the task and awaits external input (an accept or reject signal).

### 3. New Operations

**`AcceptTask`**
```proto
rpc AcceptTask(AcceptTaskRequest) returns (Task);

message AcceptTaskRequest {
  string task_id = 1;
  optional string context_id = 2;
  optional Message message = 3;  // optional acceptance note
}
```
Transitions `PENDING_ACCEPTANCE` → `COMPLETED`.

**`RejectTask`**
```proto
rpc RejectTask(RejectTaskRequest) returns (Task);

message RejectTaskRequest {
  string task_id = 1;
  optional string context_id = 2;
  string reason = 3;              // machine-readable rejection code
  optional Message message = 4;  // structured feedback injected into history
}
```
Transitions `PENDING_ACCEPTANCE` → `WORKING` (agent retries with feedback
in message history) or → `FAILED` if `max_rejections` is exceeded.

### 4. New Error Code

`AcceptanceCriteriaNotMetError` — returned by the agent when `evaluator = AGENT`
and the agent determines it cannot produce output meeting the declared criteria
(e.g., criteria are contradictory or out of scope). This prevents the agent from
spinning through retry cycles it cannot win.

### 5. HTTP/REST Bindings

```
POST /tasks/{id}:accept   → AcceptTask
POST /tasks/{id}:reject   → RejectTask
```

---

## Backward Compatibility

- `AcceptanceCriteria` is optional on `Task`. Omitting it preserves current
  behavior exactly — tasks go `WORKING` → `COMPLETED` as today.
- `PENDING_ACCEPTANCE` is a new state; existing clients that don't handle it
  should treat it like any other interrupted state and poll until resolution.
- `AcceptTask` / `RejectTask` are additive operations; no existing operations change.

---

## Alternatives Considered

**A. Use `metadata` field on Task**
Unstructured, no protocol enforcement. Each implementation invents its own
convention. Rejected: doesn't solve the interoperability problem.

**B. Client creates a new Task on dissatisfaction**
Current workaround. Loses `contextId` continuity, forces re-description of
the full problem, no standard feedback injection mechanism. Rejected: too leaky.

**C. Add AC field only, no new states or operations**
Simpler but doesn't support human approval flows or explicit reject-with-feedback
cycles. The `PENDING_ACCEPTANCE` state is what makes this a protocol guarantee
rather than a convention.

---

## Open Questions

1. Should `PENDING_ACCEPTANCE` be grouped with other interrupted states
   (`INPUT_REQUIRED`, `AUTH_REQUIRED`) or treated as a distinct category?
2. Should rejection support a `retry: false` option that goes directly to
   `FAILED` rather than re-entering `WORKING`?
3. Should agents be able to declare in their Agent Card whether they support
   `AcceptanceCriteria` (similar to `streaming` / `pushNotifications`)?
4. Should `artifact_schema` validation be performed by the agent, the client,
   or a neutral middleware layer?
5. How does `PENDING_ACCEPTANCE` interact with push notification configs —
   should entering this state trigger a webhook event?

---

## Proto Diff

A concrete diff against `specification/a2a.proto` is available at
[`acceptance-criteria.diff`](./acceptance-criteria.diff).

**Summary of changes:**
- `TaskState` — adds `TASK_STATE_PENDING_ACCEPTANCE = 9`
- `Task` — adds `acceptance_criteria` field (field 7)
- `SendMessageConfiguration` — adds `acceptance_criteria` field (field 5)
- `AgentCapabilities` — adds `acceptance_criteria` capability flag (field 5)
- New enum: `AcceptanceEvaluator` (`CLIENT`, `AGENT`, `HUMAN`)
- New message: `AcceptanceCriteria`
- New messages: `AcceptTaskRequest`, `RejectTaskRequest`
- New RPCs: `AcceptTask`, `RejectTask` with HTTP/REST bindings

---

## Related Issues

- #1942 — v1.1 scope alignment (good milestone target for this proposal)
- #1956 — Intent Descriptor Extension (complementary: describes task *intent*;
  this describes task *completion conditions*)
- #1960 — Synchronous-to-asynchronous transition (intersects with how
  PENDING_ACCEPTANCE is surfaced in streaming vs. polling)
