# ADR-0057 (draft): ZCode CLI Coupling Audit and Clean-Room Transplant — the Provably-Sandboxed H1 Vehicle

> Status: PROPOSED for owner adjudication and canonical landing in `qiven-context` (owner direction 2026-09-26). This deliberation copy creates no authority by itself.
> Companion: [ADR-0055](../2026-09-25/01-adr-0055-simulated-loop-acceptance.md) (the acceptance law this vehicle serves), [ADR-0056](00-adr-0056-foundation-complete-infrastructure-layer.md) (the Foundation layer the new repository consumes), [H1 evidence audit](../2026-09-25/02-h1-trial-history-and-defect-evidence.md).

## 1. Decision and problem

MVP-4 H1 needs harness-side execution evidence without the contamination class: [ADR-0055](../2026-09-25/01-adr-0055-simulated-loop-acceptance.md) §1/§3 ban real-harness execution on the owner machine while its side-effect surface is unmapped — and the 2026-09-25 PoC proved what "unmapped" costs (official-install state contamination, browser reinstall; `MEM-20260925T190500Z-C0A1B2`). The harness's actual surface — how much filesystem read/write, how many databases, how many network transactions the ZCode CLI performs — is currently **unmapped** (owner finding, 2026-09-26). This ADR defines how that surface gets mapped and how a provably-sandboxed executable harness vehicle comes to exist.

**Step 1 — coupling audit first.** Against the pinned ZCode source revision, inventory the CLI's side-effect surface at function granularity: every filesystem read/write, every database open, every network endpoint, every process and environment touch — the map the PoC session never made. The audit is a recorded artifact with per-call-site classification and a summary verdict: **coupled** or **uncoupled** (whether a RAM-and-mock-network-only execution configuration exists in the source as it stands).

**Step 2a — coupled case → clean-room transplant (搬运法).** Port the CLI function-by-function into a new, clean qiven repository. Each ported unit or cluster compiles in the new tree, its side-effect dependencies are replaced by in-RAM substitutes or the mock LLM server's network interface, and a **per-step gate** proves the growing tree's dependency contract. The transplant continues until the rewritten CLI **depends only on RAM and the mock LLM server's network**, and that CLI completes the MVP-4 H1 steps. The new repository consumes Foundation's infrastructure layer ([ADR-0056](00-adr-0056-foundation-complete-infrastructure-layer.md)) for whatever primitives it needs — it does not vendor its own.

**Step 2b — uncoupled case → condemnation record.** If the audit proves the CLI source could already run depending only on RAM and the mock LLM server's network, then the PoC session's failure was not forced by the codebase, and its sin is recorded as one degree greater: **immediately** preserve in qiven-context, as a permanent scar record for every future LLM: the complete PoC notes from the fork's branch (`QIVEN-POC-NOTES`), and the full retained session transcript of that session, labeled as the historical cautionary record — isolation theater on a live machine; read the coupling map before ever claiming isolation. In this branch, whether even a custody-bounded build of the audited CLI runs on this machine — versus proceeding with the transplant regardless — is an explicit owner decision; the default remains the transplant vehicle, whose contract is proven by construction.

## 2. Transplant mechanics (coupled case)

- A new clean repository (owner-chosen name; e.g. a qiven harness-vehicle repository), consuming Foundation per ADR-0056, with no other dependencies.
- Port in function-level units. Each step records: the upstream function/commit it came from (provenance mapping), the compile result, a unit gate, and the dependency-contract check — no filesystem writes, no real database, no network except the mock LLM endpoint, bounded memory. Every step runs under bounded process custody (ADR-0048).
- The mock LLM server is a local, scripted endpoint (no real provider, no public network).
- **Fidelity duty:** the transplant is a port of upstream behavior, not a reimagining. Divergences discovered during porting are recorded as deltas with their cause; any delta that would change an H1 step's semantics is escalated, not smoothed over.
- The MVP-4 H1 steps run against the transplanted CLI exactly as [ADR-0055](../2026-09-25/01-adr-0055-simulated-loop-acceptance.md) specifies them.

## 3. Relation to ADR-0055

ADR-0055 Layer 2's pinned-source analysis and captured-payload replay feed the coupling audit and the transplant's provenance mapping. Once the transplant CLI's dependency contract is gate-proven, it becomes Layer 2's **executable leg**, and MVP-4 H1 execution evidence comes from it — never from an official or dev-mode build on the owner machine (that ban stands unchanged). The read-only carrier binding (ADR-0055 §3) still records the installed Desktop build identity; the transplanted CLI is labeled as the qiven-owned vehicle, not the official carrier, and the claim ledger keeps the two distinct.

## 4. Falsifiers

A transplanted unit that secretly retains a filesystem/database/network dependency fails its step gate. A divergence between upstream behavior and the port that changes an H1 step's semantics blocks acceptance. A contradiction between the audit verdict and the transplant process (audit says uncoupled, the transplant still finds coupling — or the reverse) invalidates the audit and re-opens the branch decision with owner adjudication. A coupling-audit claim that cannot be tied to named call sites is not evidence.

## 5. Amendment footprint

On acceptance: supersede the program obligation `OBL-20260926T234500Z-B4C5D6`'s rig description (its "real ZCode CLI + local mock LLM" wording dates from the pre-ban direction and predates both the execution ban and this vehicle); add the executable-leg reference to ADR-0055 §2; record the new repository in the workspace inventory; and, in the uncoupled branch, land the condemnation record as a qiven-context memory record with the PoC notes and session transcript bound to it. The ZCode fork repository remains an evidence artifact; its branch is the source of the PoC notes if that record is ordered.
