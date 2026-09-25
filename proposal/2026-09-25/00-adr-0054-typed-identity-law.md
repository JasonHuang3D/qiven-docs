# ADR-0054 (draft): Typed-Identity Law for Qiven Native Code — Revoking the "Land First, Revisit Later" Semantics

> Status: PROPOSED draft of a canonical ADR for `qiven-context`, opened for cross-LLM deliberation and owner acceptance. Nothing here is law until the owner accepts the ADR and it lands in `qiven-context` through its authority process. This document is the deliberation copy and belongs in `qiven-docs`.
> Companion documents: [ADR-0055 draft — simulated-loop acceptance](01-adr-0055-simulated-loop-acceptance.md), [H1 trial history and defect evidence](02-h1-trial-history-and-defect-evidence.md).
> Authorship: drafted by GLM-5.3 at the project owner's direction (2026-09-25 UTC), after the fourth failed MVP-4 owner-live H1 trial; see the evidence document for the incident record.

## 1. Decision requested

1. **Revoke the "land first, revisit later" semantics for Qiven native (C++) code.** The recurring justification pattern — "implement the call site now with hand-rolled string handling; move the abstraction upstream later" — is no longer an accepted engineering position anywhere in Qiven native code. Each such deferral must now satisfy the constitutional falsifiability clause in full (observable failure signals, a falsification path, a revisit trigger, a credible migration path) AND a named owner-accepted exception; there is no blanket deferral class anymore.
2. **Adopt the typed-identity law** below (TI-1 through TI-7): identity values that cross a layer, process, or wire boundary — filesystem paths first, protocol scalars alongside — are owned by typed primitives in the semantically-correct layer, every existing call site is swept to them, and a mechanical gate makes the raw pattern class unconstructible going forward.
3. **Sequence downstream semantics work behind the wave** (TI-6): no new downstream feature work in `qiven-runtime` semantics starts until the typed-identity wave lands. Build-infrastructure migrations are not blocked (scope defined in TI-6).

## 2. Background and evidence (summary)

Qiven is a multi-repository native system (`qiven-foundation` → shared layers including `qiven-runtime` → products). The full evidence record is the [companion document](02-h1-trial-history-and-defect-evidence.md); the load-bearing facts:

- **The path-separator defect class is chronic.** Filesystem paths have been handled as raw `std::string` across the runtime, adapters, and tooling. Windows-absolute, forward-slash, and MSYS-style (`/d/...`) forms meet hand-written normalization and comparison at each call site. The recorded incidents include a CWD-derived default that broke an owner-live kit, an installation-record image check that compares path strings character-by-character without canonicalization, a hook-path normalizer that exists precisely because every input form arrives, and repeated template/config path-form drift. The project owner's assessment — "hundreds of compile-time and runtime errors" across the project's sessions — is directionally consistent with the record even where individual incidents were caught before landing.
- **The class is not confined to paths.** The fourth MVP-4 H1 trial failed uniformly because a protocol scalar (a deadline in milliseconds) was validated by one code path while a different code path chose its value, split across the hello and event frames of one connection. One side gave the session-start event a 9750 ms budget; the other capped every non-event frame at 5000 ms. Every probe then failed with an unrelated-looking denial code. Root cause is a one-line contract split; the failure cost an owner-live acceptance trial. This is the same defect class — an untyped identity value whose policy is defined in more than one place.
- **The mechanism is structural, not a competence story.** Each authoring session minimizes its local diff; upstreaming an abstraction costs a bigger diff, another repository, and a pin ripple, so the locally-cheapest patch wins every time. Two hundred locally-cheapest patches are two hundred independent guesses at path semantics. The only durable countermeasure is mechanical: make the defect class unconstructible, so that the locally-cheapest patch IS the typed one.

## 3. Proposed normative text (the law)

> **TI-1 — Typed canonical primitives in the owning layer.** Qiven introduces `qiven::fs::Path` in `qiven-foundation` as the owning type for filesystem identity in native code. Construction is explicit about input form (Windows-absolute in either separator convention, MSYS-style POSIX mapping, relative); the internal representation is canonical and singular; comparison follows an explicit platform policy (case-folding where and only where the filesystem folds); membership tests (`is_under`, `relativize`) are member operations; output conversions (`as_windows`, `as_posix`, `as_native`) are explicit named functions. The type does not implicitly convert to or from raw strings.
>
> **TI-2 — Wire and protocol scalars are typed with one policy table.** Every scalar that crosses a process or wire boundary and carries a validity policy (deadlines, budgets, limits, version stamps) is a named type whose policy is defined in exactly one place and consulted by every frame, handshake, and event that carries it. The MVP-4 hello-deadline split is the governing incident: a ceiling table that classifies frames by kind must be the single source consulted by both the client that chooses budgets and the server that validates them.
>
> **TI-3 — Primitive-with-consumer rule (anti-speculative-completion).** No primitive lands without its real consumer in the same wave, and no consumer hand-rolls its own once the primitive exists. This law revokes land-first deferral WITHOUT licensing speculative framework completion: the wave is scoped by enumerated call sites, not by imagination. Constitution §15 falsifiability continues to apply to any deferral that remains.
>
> **TI-4 — Mechanical ban.** A repository gate task fails on raw path-string comparison and normalization patterns in Qiven native code (defined by an audited pattern list reviewed like any other gate). New code cannot reintroduce the class; the ban list itself is versioned and amendable through normal review, never silently widened or narrowed.
>
> **TI-5 — Call-site sweep.** The existing hand-written sites (the hook-path normalizer, the installation-record image comparison, the governed-scope checks, configuration and profile path handling, and their kin) are migrated to the typed primitives as part of the wave, not deferred. A sweep that leaves known sites behind fails its own acceptance.
>
> **TI-6 — Sequencing scope.** Downstream SEMANTICS work (runtime feature batches, the CA program's implementation legs, MVP-5) does not start until the wave lands at its acceptance bar. Build-infrastructure migrations (workspace dependency-resolution steps, toolchain migrations, pin reconciliation) are NOT blocked: they are disjoint from the defect class and independently scheduled. This clause is the owner's sequencing direction recorded as law.
>
> **TI-7 — Mirror decisions for tooling languages.** Qiven's Python operator/tooling layer carries the same stringly-typed identity values. The present ADR adopts the principle repository-wide but scopes the FIRST wave to native code; the Python mirror (typed path helpers in the devkit operator layer, or an explicit accepted exception) is a named follow-up decision, not a silent skip.

## 4. What remains in force

- ADR-0046 layering (toolchain → devkit → all repositories) and the software-engineering philosophy: a correct downward dependency is a feature; semantic ownership decides placement. `qiven::fs::Path` belongs in Foundation because path identity is intrinsically lower than every consumer — this ADR is that philosophy applied, not a new model.
- All existing ADRs; this ADR adds a law, it does not amend governance, handoffs, or program structure.
- The constitution's simplicity clause (§15): deferrals are still POSSIBLE — they just cannot be the unexamined default anymore, and each must carry its falsifiability record plus a named owner-accepted exception.

## 5. Consequences and honest risks

- **Cost:** the sweep touches every path-handling site in the runtime and adapters, plus Foundation's new surface with property tests over the input-form equivalence classes. Pin ripples ride the normal managed-template cadence.
- **Risk of over-typing:** the classic failure mode of "finish the foundation first" programs is building the wrong abstraction without consumer pressure. TI-3 is the countermeasure and reviewers should police it: a primitive with no enumerated consumer in this wave is a defect of this ADR, not a success of it.
- **Risk of ban-gate theater:** a pattern ban that fires on idiomatic non-path string code taxes every batch. The ban list starts narrow (comparison and normalization of things declared/used as paths) and widens only on evidence.
- **Alternative rejected:** "be more careful in future sessions" — rejected because the failure mechanism is structural local optimization; discipline-only fixes have a measured recurrence record in this project.

## 6. Open questions for deliberation

1. **Representation:** should `Path` wrap `std::filesystem::path` (native facet, Windows semantics) or own a normalized UTF-8 representation with explicit conversion at the OS boundary? Trade-offs around Unicode, long paths, and compare policy.
2. **MSYS mapping placement:** is `/d/...` interpretation a property of the type (`Path::from_msys()`) or a boundary-only adapter concern (the hook adapter converts before constructing)? The former centralizes; the latter keeps Foundation ignorant of a shell convention.
3. **Case-folding granularity:** per-volume detection vs a platform-wide policy; interaction with `is_under` on case-sensitive directories inside a case-insensitive volume.
4. **Ban-gate shape:** clang-tidy custom check vs gate-script pattern scan vs both; where the ban list lives and its amendment process.
5. **Scope of "wire scalars":** deadline/budget/limit types in the runtime IPC layer only, or a Foundation-level scalar-policy framework? (The evidence supports the former; the latter smells like the speculative completion TI-3 forbids.)
6. **Python mirror timing:** same wave, next wave, or explicit exception with a falsifiable trigger?
7. **Does the law extend to configuration surfaces** (YAML/JSON path fields parsed by native code), requiring typed parsing at the loader boundary?
