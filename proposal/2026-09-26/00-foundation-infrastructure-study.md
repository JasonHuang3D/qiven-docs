# Foundation infrastructure scope and mature adoption: decision study

> Status: EXPLORATORY PROPOSAL in qiven-docs PR #4, reflecting the owner's direction to examine mature infrastructure adoption. This is not ADR-0056, an accepted decision, a mandate to vendor, or a change to the MVP-4 gate. The earlier PR revision presented this direction as four owner rulings; the owner clarified that it was an idea for investigation, not a settled architecture.
> Related: [typed path and IPC boundary proposal](../2026-09-25/00-adr-0054-typed-identity-law.md), [Foundation audit](../2026-09-25/03-foundation-architecture-and-museum-transition.md), [ZCode vehicle study](01-zcode-harness-vehicle-study.md).

## Question and current boundary

Could a broader, deliberate infrastructure layer built on mature implementations remove recurring local reinvention and improve the C++ dependency graph? This is a valid design hypothesis, especially for the path defect investigated in PR #4. It does not establish that filesystem, network, threading, and memory in *all their aspects* belong to one Foundation repository. The accepted [engineering philosophy](https://github.com/JasonHuang3D/qiven-context/blob/main/collaboration/software-engineering-philosophy.md) §§4–8 explicitly leaves some shared capabilities above Foundation (Networking and Runtime are named examples), requires a concrete consumer and coherent adjacent contract, and rejects speculative framework completion. [ADR-0024](https://github.com/JasonHuang3D/qiven-context/blob/main/decisions/ADR-0024.md) changes the consumer-count rule; it does not assign all future infrastructure to Foundation.

The first concrete decision remains the path/IPC boundary in the companion proposal. A `qiven::fs` public contract could be a Foundation responsibility; path-based admission still needs OS-backed object identity and policy in the appropriate Runtime boundary. An IPC transaction deadline belongs to its protocol owner, even if lower clock/time primitives are supplied by Foundation. Do not make that repair contingent on a survey of unrelated domains.

## Evidence to collect before proposing an ADR

| Capability and observed consumer | Ownership question | Candidate implementations to evaluate |
| --- | --- | --- |
| Lexical paths, explicit encoding and conversion for Runtime | Which stable representations belong in `qiven::fs`? Which operations require handles and OS-specific checks at the caller? | Constrained standard-library/OS primitives behind a Qiven contract; established libraries as dependencies; a selectively vendored component where justified. |
| IPC and network transport | Which semantics are transport-specific to Runtime or Networking, and which are genuinely foundational? | Existing OS/standard facilities and maintained libraries against the actual protocol and deployment target. |
| Synchronization and threading | Which real consumer requires more than existing standard/OS primitives? | Existing facilities and targeted libraries; evaluate cost, cancellation, resource lifetime and platform behavior. |
| Allocation and memory | Which measured allocation/lifetime failure is unsolved by current Foundation facilities? | Current allocator/lifetime contracts, standard/OS facilities and targeted libraries under measured load. |

For each candidate record source revision, license and redistribution terms, maintenance/security response, platform/compiler coverage, binary and transitive dependency cost, failure/exception/allocator behavior, API/ABI implications, performance *at the relevant workload*, and independent regression results. Commercial use and popularity may help discover candidates; they do not prove contract fit. Check qiven-foundation's own distribution/licensing posture before copying code. Porting or vendoring a whole library is not the default: a direct dependency or a thin owned adapter may preserve updates and upstream fixes better. A targeted implementation remains admissible when a coherent contract cannot be supplied by an external primitive at acceptable cost; the specific case needs evidence, not a blanket exception from a universal rule.

## Bounded next experiment and sequencing

1. Write the `qiven::fs` contract against actual Runtime call sites, path forms and OS-backed authorization requirements. Separate public semantics from representation and implementation.
2. Compare at least a constrained standard/OS-backed implementation and a suitable maintained library on those same fixtures. If vendoring is considered, include license/provenance, update burden, integration tests and source mapping; decide only after this comparison.
3. Integrate the chosen *bounded* path capability with its first real consumer at an exact source graph; exercise Windows path/admission edge cases and the existing relevant Foundation/Runtime gates.
4. Reassess other domains only with their own consumers and observed gaps. Record a separate ownership decision per domain; do not infer it from a filesystem result.

The existing MVP-4/H1 defects and safety boundaries remain visible and testable during this study. No proposal here suspends, weakens or postpones a full test gate until an unspecified whole-repository refactor completes. If a changed implementation makes the current gate infeasible, record the exact blocker and seek a specific canonical amendment; keep runnable, scoped regressions active.

## Decision output

A later ADR may choose a concrete owner and acquisition method for a named capability if the comparison warrants it. It must include rejected candidates and falsifiers, the tested source graph, license/provenance, migration scope and a bounded rollback. Until then, “complete infrastructure layer,” “wholesale port,” “all aspects,” and “refactor every C++ repository before the MVP-4 gate” remain hypotheses under test, not owner instructions.
