**Title:** Extension proposal — Acceptance Criteria (gate task completion on accept/reject)

---

- **Proposed by:** Amu Hlongwane (Software Engineering Labs)
- **Repository:** https://github.com/sweengineeringlabs/a2a-proposal
- **Full spec:** https://github.com/sweengineeringlabs/a2a-proposal/blob/main/extension/experimental-ext-acceptance/v1/spec.md
- **Rationale:** https://github.com/sweengineeringlabs/a2a-proposal/blob/main/proposal/proposal.md

This is an **extension proposal** (Proposal Phase, per the [Extension and Binding Governance](https://github.com/a2aproject/A2A/blob/main/docs/topics/extension-and-binding-governance.md)) — not a core-spec change. I'm looking for community feedback and a Maintainer sponsor to incubate it as `experimental-ext-acceptance`.

## Problem

In core A2A, `COMPLETED` means "the agent stopped working," not "the output meets the requester's intent." There is no interoperable, cross-vendor way to:

- declare, up front, what "done" looks like for a task;
- gate the `WORKING → COMPLETED` transition on an explicit client or human approval;
- reject an output with structured, machine-readable feedback and have the *same* task retry — preserving `contextId` and history — instead of opening a brand-new task and re-describing the problem.

Today this is handled out-of-band: an orchestrator validates artifacts itself and spins up a fresh task when dissatisfied, losing task context. Because every team invents its own convention, an orchestrator from one vendor and an agent from another can't interoperate on approval gating — the exact fragmentation A2A exists to prevent.

## Proposed solution

A negotiated extension that standardizes completion gating with **zero core-protocol changes**:

- **Declaration:** agents advertise support via an `AgentExtension` entry in `AgentCapabilities.extensions[]`; `params` declares supported evaluator modes.
- **Activation:** per-request via the `A2A-Extensions` header; the agent echoes the URI to confirm. Non-activating clients see ordinary `WORKING → COMPLETED`.
- **Criteria (data):** an `AcceptanceCriteria` object (description, optional JSON `artifactSchema`, `requiredArtifacts`, `evaluator`, timeout, max-rejections) carried in `Task.metadata`, keyed by the extension URI.
- **Pending-acceptance (state machine):** modeled as the existing `input-required` state **plus a metadata flag** — following A2A's documented "annotate, don't add enum values" rule. **No new `TaskState` value.**
- **Accept / reject (flow):** ordinary `message/send` calls carrying a `decision` payload (`accept`, or `reject` with a closed-enum `reason` + feedback). The agent — not the client — performs the resulting transition. Reject re-enters `WORKING` with feedback in history, or `FAILED` past `maxRejections`.

## Why this can't be done in core today

1. Core has no completion-gate semantics, and extensions **MUST NOT** add new enum values or new core fields — so the condition can only be expressed as an annotation on an existing state.
2. `INPUT_REQUIRED` already means "agent needs more input to proceed"; reusing it for "done, awaiting your verdict" without a standardized flag is ambiguous to any client that didn't author the convention, and core offers no way to carry the verdict (accept vs. reject-with-reason) back.
3. Core defines no schema for success criteria nor a closed set of rejection reasons, so unstructured `metadata` alone can't deliver interoperability.

And it should be a *published* extension, not private metadata: acceptance gating is a cross-vendor handshake, so without one shared, registered schema every client/agent pair must agree bilaterally — the fragmentation A2A exists to prevent.

## Alternatives considered

- **Core-protocol change** (new `PENDING_ACCEPTANCE` state + `AcceptTask`/`RejectTask` RPCs): rejected — extensions explicitly **MUST NOT** add enum values or core fields, and the need is opt-in, not universal.
- **Ad hoc `input-required` + private `metadata`:** works only inside a single vendor's stack; no cross-vendor discovery or shared vocabulary.
- **Client opens a new task on dissatisfaction** (status quo): loses `contextId` continuity and forces full re-description; no standard feedback channel.

## What I'm asking for

- Community feedback on the design.
- A **Maintainer sponsor** to proceed to experimental status (per governance, I can't create the repo myself).
- Toward graduation I'll provide a reference implementation (a mock agent that parks in `input-required` with the pending flag + a client that sends accept/reject decisions) and interop tests. License: Apache-2.0.

## Open questions

- Is `input-required` the right anchor state, or should pending-acceptance annotate `working`?
- Should `artifactSchema` validation be mandated or merely recommended, and performed by the agent vs. the client?
- Per-task vs. per-context criteria.

---

- [ ] I agree to follow this project's [Code of Conduct](https://github.com/a2aproject/A2A?tab=coc-ov-file#readme)
