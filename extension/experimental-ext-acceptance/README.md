# Acceptance Criteria Extension

> **Experimental:** This extension currently has experimental status — breaking
> changes are possible and community feedback is much appreciated. To learn more
> about the extension lifecycle, see
> [A2A Extension and Protocol Binding Governance](https://a2a-protocol.org/latest/topics/extension-and-binding-governance/).

This repository contains the specification for the **Acceptance Criteria Extension**
for the Agent2Agent (A2A) protocol.

## Purpose

In core A2A, `COMPLETED` means "the agent stopped working," not "the output meets the
requester's intent." This extension adds a negotiated **acceptance gate**: a client
declares success criteria up front, the agent parks finished output in a
**pending-acceptance** condition, and an authorized client or human **accepts**
(→ `COMPLETED`) or **rejects** with structured feedback (→ `WORKING`, same task),
preserving `contextId` and history instead of opening a new task.

It is layered entirely through the A2A extension mechanism — **no core-protocol
changes**: no new `TaskState` value, no new `Task`/`AgentCapabilities` fields, and no
new core RPCs.

## Specification

The full specification (v1 Draft) is at [`./v1/spec.md`](./v1/spec.md).

## Sample Implementation

A reference implementation lives under [`./sample`](./sample) _(work in progress —
required before graduation from experimental to official status)._

## Status & provenance

- **Status:** Experimental (proposed). A Maintainer sponsor and an
  `experimental-ext-acceptance` repository under `a2aproject` are prerequisites for
  formal experimental status; until then the GitHub tree URL is the provisional
  extension URI.
- This extension supersedes an earlier core-change proposal. Rationale,
  alternatives, and the mapping from that original draft live in the proposal:
  [`../../proposal/proposal.md`](../../proposal/proposal.md).

## License

Licensed under the [Apache License 2.0](./LICENSE).
