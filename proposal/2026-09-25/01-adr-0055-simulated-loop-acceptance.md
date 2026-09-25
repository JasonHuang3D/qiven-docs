# ADR-0055 (draft): Simulated-Loop Acceptance — Retiring the Owner-Live GUI H1 Trial for MVP-4

> Status: PROPOSED draft of a canonical ADR for `qiven-context`, opened for cross-LLM deliberation and owner acceptance. Nothing here is law until the owner accepts the ADR and it lands in `qiven-context` through its authority process. This document is the deliberation copy and belongs in `qiven-docs`.
> Companion documents: [ADR-0054 draft — typed-identity law](00-adr-0054-typed-identity-law.md), [H1 trial history and defect evidence](02-h1-trial-history-and-defect-evidence.md).
> Authorship: drafted by GLM-5.3 at the project owner's direction (2026-09-25 UTC).

## 1. Decision requested

1. **Retire the owner-live GUI trial as the MVP-4 acceptance vehicle.** Four consecutive owner-live trials failed, each on an infrastructure defect invisible to the prior validation layer, each costing the owner's hands and roughly a day. The acceptance topology — a human manually driving a GUI agent against a kit — is structurally wrong for a gate whose property is mechanical enforcement.
2. **Adopt the simulated full-loop acceptance rig** defined below as the MVP-4 exit gate: deterministic, scripted, reproducible, and resident in the repository gate — no owner hands required to run or re-run.
3. **Fold the trial-4 corrective items into the wave** (the hello-deadline contract fix, preflight session-start coverage, and the conformance regression cases) rather than landing them as isolated symptom patches. [ADR-0054](00-adr-0054-typed-identity-law.md) owns the general law; this ADR owns the acceptance vehicle.

## 2. Background

MVP-4 is the Qiven runtime's mediation slice: in a real agent session, every Bash/Write/Edit tool call crosses a PreToolUse hook into a governed RuntimeHost, which classifies the target and returns allow or a typed deny; the harness must enforce the deny with no fallthrough. The exit gate's "real H1" row was defined as an owner-run live trial with a packaged kit: the owner enables the hook config, boots the host, starts a fresh agent session, and pastes six probe prompts. See the [evidence document](02-h1-trial-history-and-defect-evidence.md) for all four failures: a mis-provisioned hook payload assumption (trial 1), a connection-model defect plus an installation-record gap (trial 2), a kit environment defect (trial 3), and a protocol contract split that made every probe fail with an unrelated denial code while the preflight stayed green (trial 4).

The structural diagnosis: the live trial tests EVERYTHING AT ONCE — the kit, the config, the harness, the host, the hook, the owner's environment — with a sample size of six probes, no determinism, and a single human as the only driver. Each failure then requires a full diagnosis cycle to even locate the layer. Meanwhile the property being accepted ("deny has no fallthrough in a real harness loop") is almost entirely mechanical and is best proved mechanically.

The owner's direction (2026-09-25 UTC): build a rig that reproduces the REAL invocation loop — user input → harness → LLM → tool call → hook → Qiven runtime verdict → harness enforcement → back to the LLM — with a mock LLM, and drive a hundred real experiments through it. A pinned local clone of the harness source exists in the workspace, and a recent feasibility PoC against that harness built the mock-provider machinery this rig reuses (a dev-mode harness instance with a local mock wire endpoint, plus a portable build environment).

## 3. The acceptance definition

> **SA-1 — Two layers, both required.**
>
> **Layer 1 — payload-level scenario suite (deterministic, gate-resident).** A driver invokes the REAL hook binary with the REAL host over the real pipe, using synthesized event payloads whose schemas are taken from the pinned harness source and captured live payloads. Scenario taxonomy (each axis independently varied; the product exceeds 100 named scenarios): path forms (Windows-absolute in both separators, MSYS-style, relative, Unicode, trailing separators, `..` traversal, mixed, empty, oversize) × tools (Bash/Write/Edit) × targets (exact governed hit, governed-prefix directory, outside the governed root, conservative text-reference) × session states (unregistered, registered, degraded, duplicate start) × host states (live, no listener, draining, secret skew, version skew, timeout, replay). The suite runs in the repository gate forever and is the regression spine.
>
> **Layer 2 — real-harness full loop.** The actual agent CLI, built from a pinned revision of the harness source clone, with its provider pointed at a LOCAL MOCK LLM endpoint that returns scripted responses — including tool calls replaying the six H1 probe steps. "No real LLM" is achieved by provider configuration, not by excising code from the harness. The real hook config mediates the loop against the real host. The rig asserts the complete cycle: session registration fires, each tool call crosses the hook, verdicts match expectations, and a deny BLOCKS the tool in the real harness (the exit-code contract is enforced by the real harness code, not simulated).
>
> **SA-2 — MVP-4 exit = both layers green at the exact candidate head.** An owner-live trial becomes an OPTIONAL smoke test, never a gate. Re-adjudication of the MVP-4 exit on this basis supersedes the previous "real owner H1" row; the typed handoff law is unchanged (this ADR changes WHAT the acceptance evidence is, not who owns acceptance).
>
> **SA-3 — Fidelity duties.** Payload schemas and hook-invocation contracts are derived from the pinned harness revision and cross-checked against captured real payloads; the pin (revision, build digest) is recorded with every run, and harness drift invalidates Layer 2 coverage claims until requalified (version binding, not "latest"). The mock provider's response contract must cover the tool-call shapes the real harness parser accepts — where the parser is permissive, the mock exercises the permissive forms deliberately.
>
> **SA-4 — Honest scope.** The rig proves the mediation mechanics: invocation wiring, payload contracts, verdict computation, and enforcement-through-the-real-harness. It does NOT prove anything about real-LLM behavior (irrelevant to this gate), the human-visible GUI experience, or properties of the installed Desktop application beyond the pinned CLI loop. Those remain separate claims with separate vehicles.
>
> **SA-5 — Folded corrections.** The trial-4 root cause fix (single deadline ceiling policy consulted by hello and event frames alike — the concrete instance of ADR-0054 TI-2), preflight coverage of the session-registration path (its blind spot in trial 4), and the conformance case "handshake frame carrying an event-grade deadline" land in this wave.

## 4. Relation to standing decisions and proposals

- **ADR-0054 (companion):** the rig tests through the typed boundary that ADR-0054 introduces; its scenario taxonomy is deliberately hostile to stringly-typed path handling (the path-form axis exists to make that class regress loudly).
- **PR #3 (harness-mediation proposal, still open):** that program's model-invocation claims need exactly this kind of controlled harness instance with a mock provider; the mock-wire machinery is shared infrastructure, and PR #3's version-binding discipline (HM-5) is adopted here for the rig's harness pin. No part of this ADR prejudges PR #3's acceptance.
- **ADR-0050/0052 programs:** the CA program's dependency order is unchanged except that the MVP-4 exit leg is re-pointed at the rig; MVP-5 remains frozen behind CA-2 and the MVP-4 exit.
- **Typed handoffs (ADR-0036):** unchanged. This ADR changes the evidence vehicle, not authority.

## 5. Consequences and honest risks

- **What improves:** determinism (every failure reproduces), coverage (100+ scenarios vs 6), cost (zero owner hands; failures diagnosed from artifacts, not from a human's memory of a GUI), and permanence (the suite becomes the regression spine for all future mediation work).
- **Cost:** the rig itself is engineering — a driver, a scenario corpus, a mock provider, and a pinned harness build in the gate environment. The reusable PoC assets (portable toolchain, mock endpoint, harness fork branch with notes) reduce this substantially.
- **Risk: fidelity theater.** A rig that drifts from real harness behavior proves nothing. SA-3's pin-and-invalidate rule plus periodic re-capture of real payloads are the countermeasure; reviewers should demand the cross-check evidence, not just green runs.
- **Risk: the pinned harness revision ages.** The pin must be a recorded, deliberate bump — never a floating dependency — and each bump re-runs the full corpus.
- **Alternative rejected:** "fix trial 4 and run a fifth owner-live attempt" — rejected because the failure mode is the TOPOLOGY, not the specific defect; a fifth trial re-rolls the same dice at the same cost.

## 6. Open questions for deliberation

1. **Layer 2 breadth:** is the CLI loop sufficient, or does the acceptance also require the dev-mode Desktop application path (the PoC exercised it) given the installed-binary packaging differences the PR #3 review documented?
2. **Mock provider depth:** how faithfully must the mock model the provider API surface (streaming, tool-call deltas, multi-attempt retries) before Layer 2's enforcement claims are sound?
3. **Where Layer 2 runs:** repository gate on every run, nightly cadence, or acceptance-time only? (Layer 1 is uncontroversially gate-resident.)
4. **Scenario count and shape:** is the taxonomy's product the right acceptance unit (all named scenarios green), or is there a minimal mandatory core plus randomized generation beyond it?
5. **Pin policy:** which events force a harness-pin requalification (upstream releases, fork branch updates, toolchain changes)?
6. **Does this rig become the standing acceptance vehicle for LATER gates too** (e.g., the mediated record path arriving with MVP-5), and should the ADR say so now or defer that generalization?
