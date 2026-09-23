# TCA Proposals — Amendments Manifest

> Date (UTC0): 2026-09-23 · Branch: `GLM-5.3/tca-amendments`
> Amends: the four TCA proposals in this directory (00-03, Jason's originals of 2026-09-23)
> Authoritative hunk view: `git diff main...GLM-5.3/tca-amendments`

Initial review verdict: direction approved with ten amendments (A1-A3
blocking, B1-B4 material, C1-C3 administrative). Pass 2 accepted A1 and B2,
accepted the intent of the other amendments, and corrected six boundary
errors. Pass 3 accepted those corrections and correctly identified a conflict
with the canonical human-handoff contract. Pass 4 made that contract change
explicit and removed the remaining R3 classification ambiguity. Pass 5
corrected deterministic-output, receipt timing, source-closure, budget-error,
and observable-boundary defects exposed by checking the proposal against the
current Context and Runtime contracts. The resulting PR
branch is the authority for proposed wording; no review self-accepts the
documents.

## Amendment-to-landing mapping

| ID | Amendment | Lands in |
|---|---|---|
| A1 | Restore the ADR-0024 semantic-ownership admission criterion; the "at least two appropriate consumers" threshold is the superseded ADR-0007 rule and is explicitly not restored | 03 §5.1 |
| A2 | Treat the repeated byte-packing problem as RR-0, a separate supporting representation-remediation batch and scheduling prerequisite, not as proof or scope of CA-1. Any new Foundation primitive requires its own ADR-0024 admission record; Foundation owns generic storage/bounds/ownership and scalar-endian mechanics, while each Runtime format retains framing and prefix-width semantics | 03 §4 RR-0, 03 §6.3, 03 §6.4, 01 §4.3 |
| A3 | Profile C uses one pre-sealed, condition-blind normalizer. Judgment-bearing descriptor fields are mechanically derived or empty; there is no post-sealing alternate descriptor method, and selectors requiring curator judgment earn no Profile C utility credit | 02 §7.1 |
| B1 | Fresh-review isolation is owner-free after an explicit ADR-0036/Human Handoff Boundary amendment: routine R2 may use a disclosed harness-created fresh session; every R3 and milestone/release trial uses independently orchestrated sealed review. Verified orchestration transport ceases to be inherently H1, while owner-only trust anchors, external crossings, designated adjudication, existing K4/K5 owner-named gates, and H2-H4 remain governed as stated | 02 §8.2, 03 §4 CA-2 Work, 03 §10 Root ADR |
| B2 | New fixture F-08 (fail-closed mediator availability: lifecycle, pre-flight self-check before enabling the gate, diagnosable disjoint denial taxonomy, real multi-frame contract, comment-lies classified as defects); MVP-4 lane gains the connection-model decision, the denial-split, and kit pre-flight | 02 §4.2, 03 §5.1 |
| B3 | Transition discipline: CA-0 declares one bounded CA-1 implementation batch; a failed exit attempt or batch close without Profile A/B stops feature work for review. Session/calendar counts are planning telemetry, not an acceptance deadline. Expiring, revision-pinned program packs remain permitted during the interim | 03 §8.5, 00 §11.3 |
| B4 | R2/R3 designs declare boundary kinds in machine-readable form; Devkit compares the exact set to the receipt and mechanically invalidates growth. The independent reviewer challenges semantic completeness — whether the design omitted a real boundary — rather than doing enforceable set arithmetic manually | 01 §12.3, 01 §13.3-13.4, 02 §8.4 |
| C1 | After CA-0 atomically introduces the selector schema and bootstraps existing protected records, the qiven-context gate rejects new or materially revised protected records lacking valid selectors | 01 §4.1, 03 §6.1 |
| C2 | One qiven-context acceptance commit records the ADR and coherent MVP-4 state; separately validated Runtime commits are bound by exact revision. No atomic cross-repository transaction is claimed | 03 §10 Acceptance transaction |
| C3 | Canonical landing homes are retained; selector schema precedes enforcement, each landing carries rule-level metadata or a validated companion manifest, and accepted deliberation copies move under `accepted/<date>/` in this repository | 03 §10 Canonical landing homes |

## Follow-up correction rationale

| ID | Corrected failure mode |
|---|---|
| R1 | Prevented an incident-local byte API and consolidation effort from becoming part of the causal proof that task activation works |
| R2 | Kept the owner out of routine prompt transport and preserved H1-H4 as authority/environment boundaries rather than cognitive-review machinery |
| R3 | Replaced session/calendar pressure with a bounded engineering batch and evidence-based stop condition |
| R4 | Moved exact boundary-set comparison into Devkit and left only semantic completeness to independent cognition |
| R5 | Removed the selector-schema bootstrap cycle: metadata is bootstrapped before the rejection gate activates |
| R6 | Removed impossible cross-repository atomicity and bound repository-local commits by exact revision instead |
| R7 | Corrected the claim that owner-free orchestration merely reinterprets the handoff contract: ADR-0036 explicitly classifies fresh-session launch/relay as H1, so the root ADR must narrow that text honestly |
| R8 | Removed the undefined `R3 governance-adjacent` split: every R3 cognitive review now uses orchestrated isolation; governance mutation separately retains H2 plus root-principal acceptance |
| R9 | Moved issuance time and random receipt IDs outside byte-identical canonical bundle output; Profile A compares deterministic selection bytes and independently verifies receipt bindings |
| R10 | Separated pre-design activation receipt, observed bundle delivery, later design-evidence binding, and falsification receipt; only selector-bearing changes require reactivation, while every changed design/candidate digest requires renewed binding/review |
| R11 | Added an exact external repository source lock and a separate ActivationGeneration sidecar, so Devkit/Foundation/Runtime source changes cannot reuse a stale Context-only index key or mutate an immutable execution bundle |
| R12 | Aligned protected budget overflow with the architecture's `BudgetInsufficient` machine result; re-deliberation is a subsequent decision |
| R13 | Scoped before-phase and no-bypass claims to observed harness entrypoints and controlled publication gates; after-the-fact receipt issuance cannot prove earlier cognition delivery |

## Evidence boundary

The MVP-4 incident-specific assertions in B2 remain proposed requirements until
the accepting Context transaction binds each assertion to an exact H1 artifact,
accepted incident record, or implementation revision. This proposal repository
does not elevate a reviewer's implementation observation into canonical project
truth merely by repeating it.

## Pass-3 finding and pass-4 disposition (2026-09-23 UTC)

GLM-5.3 accepted all six corrections R1-R6 (R1, R4, and R5 each fixed a
real pass-1 defect) and correctly identified one remaining governance gap:
owner-free fresh-review transport conflicts with the canonical handoff
contract unless the accepting ADR changes that contract explicitly.

Pass 4 accepts the gap but corrects two details:

- This is not a change to a merely historical reading. ADR-0036 and
  `human-handoff-boundary.md` expressly place fresh-session launch, relay,
  and fresh-consumer acceptance roles inside H1. The root ADR must narrow
  that classification for mechanically verified approved orchestration.
- `R3 governance-adjacent` was undefined. Every R3 cognitive review now
  receives orchestrated isolation. A governance mutation additionally
  retains H2 plus root-principal acceptance. Existing owner-named K4/K5
  trust-anchor gates and H2-H4 claims remain unchanged.

## Pass-5 contract check (2026-09-23 UTC)

The current Production MVP binds an execution `RuntimeGeneration` to its existing immutable Context bundle, profile, and Runtime build; it does not bind external Devkit or Foundation revisions. The amended TCA therefore publishes a separate activation generation bound to an exact multi-repository source lock. Acceptance distinguishes deterministic selection bytes from issuance metadata and checks the activation-to-design/delivery timeline explicitly. These are proposed corrections, not claims that the corresponding implementation or an ADR-0036 amendment has landed.
