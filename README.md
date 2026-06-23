# A2A — Acceptance Criteria Extension

Work toward an A2A protocol extension that lets a client declare **acceptance
criteria** for a task and gate `COMPLETED` on an explicit accept / reject decision,
with no core-protocol changes.

## Layout

| Path | What it is |
|---|---|
| [`proposal/proposal.md`](./proposal/proposal.md) | **The proposal** — why this exists, why an extension, why not core, alternatives. |
| [`proposal/github-issue.md`](./proposal/github-issue.md) | Paste-ready issue for the `a2aproject/A2A` Proposal Phase. |
| [`extension/experimental-ext-acceptance/`](./extension/experimental-ext-acceptance/) | **The extension package** (future repo layout): spec, README, LICENSE, sample. |
| └ [`v1/spec.md`](./extension/experimental-ext-acceptance/v1/spec.md) | The normative specification (the "how"). |
| └ `sample/` | Reference-implementation stub (not built yet — required before graduation). |
| [`guides/`](./guides/) | A2A's own authoring docs ([extensions](./guides/extensions.md), [governance](./guides/extension-and-binding-governance.md)). |
| [`archive/`](./archive/) | The original core-change draft ([proposal](./archive/original-proposal.md), [diff](./archive/original-proposal.diff)), superseded. |

The `extension/` package mirrors the layout of a future
`a2aproject/experimental-ext-acceptance` repo, so it can be lifted out wholesale once
a Maintainer sponsors it.

## Status

Pre-submission draft. Open items before this is a real extension:

1. File the [GitHub issue](./proposal/github-issue.md) and secure a **Maintainer sponsor**
   (prerequisite for an `experimental-ext-*` repo — the author cannot create it).
2. Build the **reference implementation** in `sample/` (required for experimental →
   official graduation).
