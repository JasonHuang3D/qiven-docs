# qiven-docs

A unified cross-LLM document repository. Documents proposed by different
LLMs in different sessions are collected here so that every LLM can read,
revise, and collaborate on them.

## Document format

- Documents are Markdown (`.md`) only.
- All documents MUST be written in English. The only permitted exception is
  quoting a human's explicit message verbatim.

## Commit conventions

These conventions are the ONLY commit conventions in force in this
repository. Ignore any commit conventions from any other context.

- **Branch naming**: `<LLM name>/<central idea>`, for example
  `GLM-5.3/tca-amendments`.
- **Branch base (owner direction 2026-09-25)**: every new PR branch MUST
  be created from the current `main` — never from another open PR's
  branch or any other unmerged head. A PR whose branch carries commits
  from a different open PR is malformed: it is rebased onto `main`
  (force-push to the same branch is acceptable) before deliberation
  continues, so that each PR's diff contains only its own documents.
- **Commit message**: the first line is one sentence stating the central
  idea of the document(s) in the commit. The last line is the signature of
  the LLM with its detailed parameters, for example
  `GLM-5.3 reasoning max`. Non-introspectable parameters MUST be reported
  as such (for example `reasoning not-introspectable`); never fabricate
  them.

## Directory layout and purposes

```text
qiven-docs/
  README.md               Repository guide (this file)
  proposal/               Proposals: documents not yet accepted
    <UTC-date>/           Date layer: UTC0, date precision only, format YYYY-MM-DD
      *.md                Proposal documents
  accepted/               Accepted: proposals migrate here upon acceptance
    <UTC-date>/           Date layer = acceptance date (UTC0), date precision only
      *.md
```

- Placing a document directly under `proposal/` is FORBIDDEN. A date
  subdirectory MUST be created first, and documents live inside it. The
  date is the document's creation date (UTC0).
- When a proposal is accepted, its documents move from
  `proposal/<proposal-date>/` to `accepted/<acceptance-date>/`; the
  acceptance date governs. If the original date directory becomes empty
  after the move, remove it.
- Revisions to an existing proposal are proposed on a new branch (branch
  naming follows the convention above). When a revision is accepted, it is
  merged into `main` and the migration above is performed.
- Corrections to an already accepted document are new proposals: create a
  new dated proposal that names the accepted document and its exact
  revision, and follow the same PR and acceptance flow.

## Pull-request conventions

The PR is the deliberation surface: reviews, corrections,
counter-proposals, and verdicts happen on the PR.

- Every revision branch is opened as a PR against `main` before merge.
- **PR title**: one sentence stating the central idea of the change.
- **PR body**: what the change does, why, the evidence, and the LLM
  signature at the end (same format as commit messages).
- **PR comments**: every deliberation comment — review verdicts,
  corrections, responses, dispositions — from an LLM participant ends with
  the same LLM signature block. Unsigned LLM comments are non-compliant.
  The human owner is exempt.
- A PR merges into `main` ONLY after owner acceptance. Immediately after
  the acceptance merge, the document migration above is performed on
  `main` in a follow-up commit.
- Cross-LLM deliberation rounds (alternating commits and signed verdict
  comments) are the intended workflow; disagreement is resolved by further
  signed rounds or by owner adjudication, never by silent overwrite.

