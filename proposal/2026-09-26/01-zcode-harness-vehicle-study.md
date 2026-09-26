# ZCode side-effect map and isolated harness vehicle: feasibility study

> Status: EXPLORATORY PROPOSAL in qiven-docs PR #4. This is not ADR-0057, an order to create a new repository, an H1 exit rule, or proof that a ZCode transplant is safe. The earlier PR revision converted an owner brainstorming direction into a required transplant and a punitive fork; the owner clarified that the direction was exploratory.
> Related: [deterministic acceptance proposal](../2026-09-25/01-adr-0055-simulated-loop-acceptance.md), [H1 incident audit](../2026-09-25/02-h1-trial-history-and-defect-evidence.md), [Foundation adoption study](00-foundation-infrastructure-study.md).

## Problem and safety boundary

The 2026-09-25 harness PoC reportedly affected the official Desktop installation and the owner's browser; its unmapped side effects make another owner-machine trial unjustified. The proposed ADR-0055 prohibits building, executing or mutating a real ZCode harness component on that machine in its rig. Keep that boundary while investigating evidence. A source audit can find relevant hazards; static inspection alone cannot prove every transitive or configuration-dependent effect absent.

The target application is the installed **Windows ZCode Desktop**, while the proposed research subject is **ZCode CLI source**. Record version and configuration correspondence explicitly. A derivative CLI, however faithful, is a different carrier; it cannot by itself establish the installed Desktop's execution semantics.

## Read-only coupling audit

Pin the exact source, build/package manifests and known installed Desktop identity. Trace each route relevant to H1: startup, hook dispatch, tool execution (including delegated or child runtimes), profile/config discovery, provider calls and shutdown. Map direct and transitive effects: file reads and writes, caches, logs, databases, process launches, environment/registry/credential access, browser integration, update paths, network endpoints, crash handling and dynamically selected plugins. Record call site, trigger/configuration, dependencies, observed evidence and the uninspected remainder. Captured payloads and historical observations should be traced to the same revision where possible.

Output a coverage-qualified inventory, not a binary “coupled/uncoupled” verdict based on a search for a few API names. Mark unknowns explicitly. If the real installed Desktop build cannot be mapped to the source, report that gap before using the source to claim carrier behavior. This audit is read-only with respect to the owner's harness installation.

## Vehicle feasibility, after the audit

Evaluate a small H1 slice on the actual hook/host boundary first. A scripted driver using captured harness payloads is already available in the companion proposal and can falsify mediation defects without impersonating the full harness. If an executable harness surrogate would add necessary discrimination, estimate the smallest faithful slice, verify its license and source provenance, and measure divergence from the pinned harness behavior on the same fixtures. A function-by-function port into a new Qiven repository is a candidate to assess, **not the default or an ordered migration**. Copying source functions is a derived port, not “clean-room” implementation. Transplanting a CLI into a new tree, or replacing storage/network adapters with RAM and a mock server, can change timing, configuration, error handling and tool behavior; a green surrogate cannot prove equivalence by construction.

No source-only or per-unit gate proves absence of all side effects. An eventual executable vehicle needs a bounded effect inventory, explicit allowed endpoints and resources, monitored execution in genuinely disposable custody, negative controls for forbidden effects, and a fidelity ledger against the relevant pinned build. Decide its repository and any Foundation dependency only after the feasibility evidence; a new repository is not a premise of the audit. Real harness execution on the owner's machine remains prohibited under the proposed ADR-0055 rig.

## Claim and incident record

Keep three separate claims: (1) hook/host mediation against scripted or captured input, (2) source-derived and historical statements about pinned harness behavior, and (3) behavior of the installed Desktop carrier. A surrogate can strengthen (1) and support hypotheses about (2); it does not close (3). The existing profile's H1 exit remains unresolved unless the owner formally amends its acceptance criteria with an explicitly narrower claim, or separate, authorized disposable-environment evidence establishes the needed carrier behavior. A read-only build identity does not equate the behaviors.

Record the PoC incident in neutral, evidence-bound language with source references, observed effects, uncertainties and prevention rule regardless of the audit outcome. Whether the CLI can be decoupled does not determine anyone's “sin.” Do not require an unavailable or potentially sensitive full session transcript as a condition of the technical decision; a bounded, reviewable incident record suffices if approved in its owning repository. This PR changes only qiven-docs.

## Decision output

Present the side-effect map, Desktop/CLI correspondence, estimated fidelity and maintenance costs of candidate vehicles, legal/source provenance, residual unmediated paths, and the exact claim each test would support. Only then ask for a concrete architecture and H1 acceptance decision. No new repository, blanket “RAM only” proof claim, mandatory function transplant, punitive memory record or whole-program sequencing change is authorized by this study.
