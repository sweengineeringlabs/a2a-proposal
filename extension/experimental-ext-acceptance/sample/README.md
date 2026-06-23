# Sample Implementation (work in progress)

This directory will hold the reference implementation of the Acceptance Criteria
Extension, demonstrating an end-to-end accept/reject flow. A production-quality
reference implementation is **required** to graduate the extension from experimental
to official status (per A2A governance).

## Planned scope

- **Sample agent** — produces an artifact, then (when the extension is activated and
  `evaluator` is `client`/`human`) parks the task in `input-required` with the
  `{EXT}/pendingAcceptance` flag instead of going straight to `COMPLETED`.
- **Sample client** — activates the extension via the `A2A-Extensions` header,
  attaches `AcceptanceCriteria`, then sends an accept or a reject (with `reason` +
  feedback) via `message/send` and observes the resulting transition.
- **Interop test** — asserts: (1) a non-activating client sees ordinary
  `WORKING → COMPLETED`; (2) accept → `COMPLETED`; (3) reject(retry) → `WORKING`
  with feedback in history; (4) reject past `maxRejections` → `FAILED`;
  (5) unauthorized accept is refused (spec §11).

## Status

Not yet implemented. See the spec at [`../v1/spec.md`](../v1/spec.md).
