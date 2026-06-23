# Proposal: Acceptance Criteria Extension for A2A

- **Author:** Amu Hlongwane (Software Engineering Labs)
- **Status:** Draft proposal (pre-submission)
- **Target:** `a2aproject/A2A` — extension Proposal Phase
- **Specification:** [`../extension/experimental-ext-acceptance/v1/spec.md`](../extension/experimental-ext-acceptance/v1/spec.md)

> This document is the **proposal** — it argues *why* the capability should exist
> and *why* it belongs in an extension. The normative *how* lives entirely in the
> [specification](../extension/experimental-ext-acceptance/v1/spec.md). The
> condensed, paste-ready GitHub issue is [`github-issue.md`](./github-issue.md).

---

## 1. Problem

In core A2A, `COMPLETED` means "the agent stopped working," not "the output meets
the requester's intent." There is no interoperable, cross-vendor way to:

- declare, up front, what "done" looks like for a task;
- gate the `WORKING → COMPLETED` transition on an explicit client or human approval;
- reject an output **with structured, machine-readable feedback** and have the *same*
  task retry — preserving `contextId` and history — instead of opening a brand-new
  task and re-describing the problem.

Today this is solved out-of-band: an orchestrator validates artifacts itself and
spins up a fresh task when dissatisfied, losing task context and conversation
continuity. Because every team invents its own convention, an orchestrator from one
vendor and an agent from another cannot interoperate on approval gating — the exact
fragmentation A2A exists to prevent.

## 2. Proposed solution

A negotiated extension that standardizes completion gating with **no core-protocol
changes**. In outline — full normative detail is in the
[specification](../extension/experimental-ext-acceptance/v1/spec.md):

- **Declaration:** agents advertise support via an `AgentExtension` entry in
  `AgentCapabilities.extensions[]`.
- **Activation:** per-request via the `A2A-Extensions` header; non-activating clients
  see ordinary `WORKING → COMPLETED`.
- **Criteria:** an `AcceptanceCriteria` object (success description, optional JSON
  `artifactSchema`, required artifacts, evaluator, timeout, max-rejections) carried in
  `Task.metadata`.
- **Pending-acceptance:** the existing `input-required` state plus a metadata flag —
  no new `TaskState`.
- **Accept / reject:** ordinary `message/send` calls carrying a `decision` (`accept`,
  or `reject` with a closed-enum `reason` + feedback); the agent performs the
  transition.

## 3. Why an extension, and why not core

The capability is opt-in — most tasks never want a gate — and by A2A's own rules it
cannot go in core: extensions **MUST NOT** add enum values or core fields
([`guides/extensions.md`](../guides/extensions.md)), so a `PENDING_ACCEPTANCE` state
or a `Task.acceptance_criteria` field is disallowed. The condition can only be an
annotation on an existing state, carried in `metadata`. Core also lacks the
semantics to express this on its own:

1. **No completion gate.** `COMPLETED` is terminal and set unilaterally by the agent;
   no core state/field/method means "output produced, awaiting an external decision."
2. **`INPUT_REQUIRED` is overloaded.** It means "agent needs more input to proceed";
   reusing it for "done, awaiting your verdict" without a standard flag is ambiguous
   to any client that did not author the convention, and core cannot carry the
   verdict back.
3. **No shared vocabulary** for success criteria or rejection reasons, so unstructured
   `metadata` alone cannot deliver interoperability.

## 4. Why a *published* extension, not an ad-hoc convention

The sharpest challenge is not "extension vs. core" — it is "why a **registered,
published** extension, when any team can park a task in `input-required` and use
private `metadata` keys?" Because acceptance gating is a **two-party handshake
across a trust or vendor boundary**, the class of thing A2A exists to standardize:

- **Ad hoc only works inside one stack.** A2A's reason to exist is letting an
  orchestrator from team A drive an agent from team B. Crossing that boundary, both
  sides need a *shared* answer to: how criteria are expressed, how the agent signals
  "done, awaiting your decision," how accept vs. reject is sent, and what a rejection
  `reason` means. Without a published schema, every A↔B pair needs a bilateral
  agreement — the N×M fragmentation A2A was built to eliminate.
- **`input-required` alone is ambiguous across vendors.** A client polling a
  third-party agent cannot tell "needs more input" from "finished, awaiting
  approval," and has no standard way to respond with accept vs. reject-with-feedback.
- **It passes A2A's own litmus test.** `streaming` and `push_notifications` are
  first-class precisely because they are cross-vendor handshakes, not internal agent
  details. Acceptance gating is the same shape: high need for inter-party agreement,
  partial adoption.
- **A registry entry buys discovery + conformance.** Declared in the Agent Card and
  activated via header, a client can discover at runtime whether an arbitrary agent
  supports gating and degrade gracefully — something private metadata cannot offer.
  A registered extension also ships one reference schema and implementation, so
  independent implementations interoperate without ever talking to each other.

## 5. Alternatives considered

- **Core-protocol change** (new `PENDING_ACCEPTANCE` state + `AcceptTask`/`RejectTask`
  RPCs): rejected — extensions explicitly **MUST NOT** add enum values or core
  fields, and the need is opt-in, not universal. (This was the author's original
  draft; see [`../archive/original-proposal.md`](../archive/original-proposal.md).)
- **Ad hoc `input-required` + private `metadata`:** works only inside a single
  vendor's stack; provides no cross-vendor discovery or shared vocabulary.
- **Client opens a new task on dissatisfaction** (status quo): loses `contextId`
  continuity and forces full re-description; no standard feedback channel.

## 6. What is being requested

- **Scope:** incubate as `experimental-ext-acceptance` under `a2aproject`.
- **Ask:** community feedback, and a **Maintainer sponsor** to proceed to
  experimental status (per governance, the author cannot create the repo).
- **Commitments toward graduation:** a production-quality reference implementation
  (mock agent that parks in `input-required` with the pending flag + a client that
  sends accept/reject decisions) and interop tests. License: Apache-2.0.

## 7. Open questions

1. Is `input-required` the right anchor state, or should pending-acceptance annotate
   `working`?
2. Should `artifactSchema` validation be mandated or merely recommended, and
   performed by the agent vs. the client?
3. Per-task vs. per-context criteria (a per-context form would warrant a new URI).
