# GitHub issue (paste into `a2aproject/A2A`)

> Per the [Extension and Binding Governance](https://github.com/a2aproject/A2A/blob/main/docs/topics/extension-and-binding-governance.md)
> "Proposal Phase," this is an **extension proposal**, not a core-spec change.
> Headings below map to the repo's Feature Request template. Full rationale:
> [`proposal.md`](./proposal.md). Full spec:
> [`../extension/experimental-ext-acceptance/v1/spec.md`](../extension/experimental-ext-acceptance/v1/spec.md).

**Title:** `[Feat]: Extension proposal — Acceptance Criteria (completion gating with accept/reject)`

---

## Is your feature request related to a problem? Please describe.

In core A2A, `COMPLETED` means "the agent stopped working," not "the output meets
the requester's intent." There is no interoperable, cross-vendor way to:

- declare, up front, what "done" looks like for a task;
- gate the `WORKING → COMPLETED` transition on an explicit client or human approval;
- reject an output **with structured, machine-readable feedback** and have the *same*
  task retry — preserving `contextId` and history — instead of opening a brand-new
  task and re-describing the problem.

Today this is solved out-of-band: an orchestrator validates artifacts itself and
spins up a fresh task when dissatisfied, losing task context. Because every team
invents its own convention, an orchestrator from one vendor and an agent from
another cannot interoperate on approval gating — the exact fragmentation A2A exists
to prevent.

## Describe the solution you'd like

A **negotiated extension** that standardizes completion gating **with zero
core-protocol changes**. (The `https://a2a-protocol.org/extensions/` URI prefix is
assigned only at graduation to official; while experimental the identifier is the
GitHub tree URL of the sponsored `experimental-ext-acceptance` repo.) In brief:

- **Declaration:** agents advertise support via an `AgentExtension` entry in
  `AgentCapabilities.extensions[]`; `params` declares supported evaluator modes.
- **Activation:** per-request via the `A2A-Extensions` header; the agent echoes the
  URI to confirm. Non-activating clients see ordinary `WORKING → COMPLETED`.
- **Criteria (data):** an `AcceptanceCriteria` object (description, optional JSON
  `artifactSchema`, `requiredArtifacts`, `evaluator`, timeout, max-rejections)
  carried in `Task.metadata` keyed by the extension URI.
- **Pending-acceptance (state machine):** modeled as the existing `input-required`
  state **plus a metadata flag** — per A2A's documented "annotate, don't add enum
  values" rule. **No new `TaskState` value.**
- **Accept / reject (flow):** ordinary `message/send` calls carrying a `decision`
  payload (`accept`, or `reject` with a **closed-enum** `reason` + feedback). The
  agent — not the client — performs the resulting transition. Reject re-enters
  `WORKING` with feedback in history, or `FAILED` past `maxRejections`.

**Why this cannot be done in the core protocol** (governance requires this):

1. Core has no completion-gate semantics, and extensions **MUST NOT** add new enum
   values or new core fields — so the condition can only be expressed as an
   annotation on an existing state.
2. `INPUT_REQUIRED` already means "agent needs more input to proceed"; reusing it
   for "done, awaiting your verdict" without a standardized flag is ambiguous to any
   client that didn't author the convention, and core offers no way to carry the
   verdict back.
3. Core defines no schema for success criteria nor a closed set of rejection
   reasons, so unstructured `metadata` alone cannot deliver interoperability.

## Describe alternatives you've considered

- **Core-protocol change** (new `PENDING_ACCEPTANCE` state + `AcceptTask`/`RejectTask`
  RPCs): rejected — extensions explicitly **MUST NOT** add enum values or core
  fields, and the need is opt-in, not universal.
- **Ad hoc `input-required` + private `metadata`:** works only inside a single
  vendor's stack; no cross-vendor discovery or shared vocabulary.
- **Client opens a new task on dissatisfaction** (status quo): loses `contextId`
  continuity and forces full re-description; no standard feedback channel.

## Additional context

- **Scope requested:** incubate as `experimental-ext-acceptance` under `a2aproject`.
- **What I'm asking for now:** community feedback, and a **Maintainer sponsor** to
  proceed to experimental status (per governance, I cannot create the repo myself).
- **Commitments toward graduation:** a reference implementation (mock agent that
  parks in `input-required` with the pending flag + a client that sends accept/reject
  decisions) and interop tests. License: Apache-2.0.
- **Open questions:** (a) is `input-required` the right anchor state, or should
  pending-acceptance annotate `working`? (b) should `artifactSchema` validation be
  mandated or merely recommended, and performed by agent vs. client? (c) per-task vs.
  per-context criteria.

## Code of Conduct

- [ ] I agree to follow this project's Code of Conduct
