# ADR-0056 (draft): Foundation as the Complete C++ Infrastructure Layer, Built by Mature Adoption

> Status: PROPOSED for owner adjudication and canonical landing in `qiven-context` (owner direction 2026-09-26). This deliberation copy creates no authority by itself.
> Companion: [ADR-0054](../2026-09-25/00-adr-0054-typed-identity-law.md), [ADR-0055](../2026-09-25/01-adr-0055-simulated-loop-acceptance.md), [ADR-0057](01-adr-0057-zcode-cleanroom-transplant.md), [Foundation architecture audit](../2026-09-25/03-foundation-architecture-and-museum-transition.md).

## 1. Decision

Foundation is the single owner of the **complete C++ infrastructure layer** for the workspace: **filesystem, network, threading, and memory — all aspects**. The layer is built by **adopting the best mature cross-platform implementations, not by self-authored rewrites**: an open survey per domain selects candidates against performance, commercial adoption, and stability; the chosen code is ported into Foundation wholesale (照搬), with qiven-owned work limited to **adaptive format, types, naming, and reconciliation with Foundation's existing symbols**.

This extends two accepted rules to whole-layer scale: ADR-0024's correct-lower-ownership (the infrastructure layer is the lowest qiven-owned semantic boundary, so its completeness is a Foundation responsibility, not a per-product one) and the engineering philosophy §7's preference for mature lower primitives over local reinvention. The `qiven::fs` module decision in ADR-0054 BC-2 is the filesystem face of this layer: its deficiency list is the argument against consuming `std::filesystem` raw, **not** an instruction to hand-roll a replacement — the module's implementation is expected to come from the adopted library, adapted to the module contract. Self-authored code inside the infrastructure layer requires explicit owner adjudication per case.

## 2. Adoption process (per domain: filesystem, network, threading, memory)

1. **Survey.** Web-wide search for mature cross-platform candidates — libraries with real commercial deployment, active maintenance, and a performance track record. Record the candidate set and the evaluation matrix per domain.
2. **Selection.** Criteria: performance; breadth of commercial adoption; stability and maturity; portability (Windows/Linux/macOS at minimum); dependency weight; license compatibility; explicit failure model. Record the per-domain choice **and the rejected alternatives with reasons** (negative knowledge, constitution §7).
3. **Porting.** Vendor the selected code into Foundation at a pinned upstream revision, then apply the adaptation layer only: format (Foundation formatting rules), types (the qiven type surface), naming (the Devkit naming law), and **symbol reconciliation** — one recorded mapping between existing Foundation symbols and incoming symbols, with no silent duplication or drift.
4. **Gates.** Each porting wave runs Foundation's full gate at its exact head, and the adopted library's own test suite runs green inside the Foundation tree before the wave closes (the suite is evidence, not decoration).
5. **Consumer migration.** Downstream repositories consume the Foundation layer; local implementations of infrastructure-layer capabilities are swept to it — ADR-0054's sweep law for path handling is the first instance; equivalent sweeps for network/threading/memory follow as their consumers migrate.

## 3. Sequencing law (owner direction 2026-09-26)

The whole-repository C++ refactor **depends on the new Foundation**: ADR-0054's path/IPC wave, the Runtime sweep, and the other C++ repositories' migrations proceed on top of the landed infrastructure layer (filesystem first — it is the MVP-4 blocker), and the program does not return to the **MVP-4 full test gate** until the refactor on the new Foundation is complete. Adoption waves and consumer refactoring may interleave by domain (fs adoption → Runtime path sweep → network/threading/memory adoption → their consumers), but no consumer hand-rolls an infrastructure capability the landed layer already owns.

## 4. Honest limits and falsifiers

- Adoption is not uncritical wrapping: if the best candidate fails Foundation's portability, failure, dependency, or cost laws for a domain, re-survey; if no mature candidate exists for a narrow gap, a bounded self-authored primitive needs owner adjudication and its own falsifier.
- The adaptation layer must stay thin. If format/type/naming adaptation demonstrably costs more than the value of the qiven-native surface, bring the measured fact to owner adjudication rather than deepening the fork.
- License and provenance compliance is a hard gate: every vendored revision is recorded with its upstream identity and license; anything unresolvable blocks the wave.
- Falsify this decision if the adopted stack cannot meet measured performance or stability needs at MVP-4 scale, or if vendoring maintenance cost demonstrably exceeds a self-authored alternative for a domain — return measured facts to adjudication, not preferences.

## 5. Amendment footprint

On acceptance: amend the Foundation architecture entry points per the [03 audit](../2026-09-25/03-foundation-architecture-and-museum-transition.md)'s replacement contract (now describing the complete infrastructure layer and the adoption law); amend ADR-0054 BC-2's implementation-source note; keep all vendored code inside `qiven-foundation` (no new third-party repositories; the workspace dependency graph gains no edges); and supersede the sequencing wording in the program obligation `OBL-20260926T234500Z-B4C5D6` with §3 above.
