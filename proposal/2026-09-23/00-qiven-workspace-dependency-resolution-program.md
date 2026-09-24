# Qiven Workspace Dependency Resolution Program

## From Ambient Sibling State to an Explicit Workspace Closure

> **Status:** Proposed governance and architecture program
> **Date:** 2026-09-23 UTC
> **Decision owner:** Qiven root governance principal
> **Applies to:** every active qiven-* repository, Devkit, Toolchain, the third-party singleton, and Runtime Task Cognition Activation
> **Companions:** 01-qiven-workspace-resolution-architecture.md and 02-qiven-workspace-resolution-migration-and-acceptance.md

## 0. Executive Decision

Qiven SHALL stop treating repository-relative paths, environment-root overrides, CMake target presence, and per-consumer exact-SHA literals as the dependency resolver for the multi-repository workspace.

Those mechanisms were useful while the workspace was small. They are now a distributed control plane whose state is reconstructed repeatedly by humans, LLMs, CMake, Python shims, and governance documents.

Qiven SHALL introduce one explicit **Workspace Resolution** layer with these properties:

1. Every participating first-party repository, tool repository, and governed third-party singleton is a node in one machine-readable dependency graph; the workspace-control repository supplies the root and is attested separately.
2. One immutable **WorkspaceGeneration** selects the exact revision and content identity of every node in a declared workspace snapshot; operation-specific closures are explicit projections of that snapshot.
3. Dependency constraints are validated before any governed CMake target, activation source, or general tool implementation is materialized. The minimal bootstrap may load only the exact identity-checked Devkit resolver to perform that validation; it cannot run an Operator task before the graph passes.
4. Repositories declare semantic dependency edges and compatibility contracts, but do not each carry an independent exact SHA for the same workspace peer.
5. CMake consumes an already-resolved graph and does not perform repository discovery.
6. A thin launcher shim MAY remain as a user-interface entry point, but a shim MUST NOT select dependency versions or discover governed implementations.
7. Devkit, Toolchain, Foundation, Runtime, Context Draft, Math, and qiven-third-party-win resolve through the same workspace closure.
8. Once qualified, TCA takes repository selection from WorkspaceGeneration while retaining its independently verifiable, complete external-source lock with commit/tree and selected-path content digests. The accepted CA-1 lock remains authoritative until that migration is accepted.
9. Publication and authoritative gates on migrated paths reject unresolved, conflicting, stale, or dirty consumed dependency closure; new task cognition also checks the accepted active-lock freshness policy before claiming current Context state.
10. Process singleton rules such as RuntimeHost remain separate from dependency singleton rules.

The immediate goal is not a new package ecosystem. It is to turn a hidden, attention-dependent multi-repository topology into explicit machine state.

---

## 1. Current Evidence

This proposal is based on the current repository contracts inspected on 2026-09-23/24 UTC.

### 1.1 Runtime already contains a hand-written repository resolver

The current qiven-runtime/CMakeLists.txt independently:

- resolves qiven-foundation from QIVEN_FOUNDATION_ROOT or a sibling ../qiven-foundation checkout;
- validates that checkout against an exact Foundation Git SHA;
- resolves qiven-context-draft using the same root/sibling pattern;
- validates an exact Context Draft SHA;
- resolves qiven-third-party-win using the same root/sibling pattern;
- validates an exact third-party singleton SHA;
- injects those source trees through add_subdirectory;
- obtains public symbols by relying on the targets thereby introduced.

Repository location, dependency selection, graph validation, target materialization, and version enforcement are therefore interleaved inside the consumer's build description.

### 1.2 The pattern is intentionally replicated

The current Devkit convention docs/conventions/cross-repo-cmake.md defines the workspace-wide pattern as:

~~~text
root override -> sibling fallback -> exact Git SHA check -> add_subdirectory
~~~

qiven-math and qiven-context-draft each repeat Foundation root resolution and an exact Foundation SHA check in their own CMakeLists.

qiven-runtime/tools/toolchain.py and qiven-foundation/tools/toolchain.py duplicate Toolchain root discovery from an environment override or sibling layout.

qiven-context/tools/qiven.py resolves Devkit from QIVEN_DEVKIT_ROOT or a sibling qiven-devkit checkout while .qiven/operator.json separately records a devkit_pin.

These are not isolated conveniences. Together they are the workspace's current dependency system.

### 1.3 Current CMake "singleton" behavior can hide a dependency conflict

Both current qiven-context-draft and qiven-math gate Foundation resolution and pin validation behind the equivalent of:

~~~cmake
if(NOT TARGET qiven::foundation)
    # resolve Foundation
    # validate this consumer's Foundation pin
    add_subdirectory(...)
endif()
~~~

When Runtime adds Foundation first and then adds Context Draft, qiven::foundation already exists. Context Draft therefore skips not only duplicate materialization, but also its own dependency constraint validation.

Today the inspected Runtime and Context Draft Foundation pins agree. The architecture does not guarantee that property. A future graph can contain two incompatible declarations while configure succeeds depending on target creation order.

Target existence is being used for two different meanings:

~~~text
"this target has already been materialized"
and
"this dependency requirement has been satisfied"
~~~

Those meanings are not equivalent.

### 1.4 Per-consumer pins create repository ripple as normal operation

The current law requires every cross-repository source consumer to carry an exact SHA and re-pin after an upstream movement.

Recent workspace work already demonstrates the consequence: Foundation and Context Draft movements caused Runtime re-pin work, while managed Operator snapshots required separate sibling synchronization.

Exactness is correct. Replicating the exact revision decision into every consumer is the defect.

### 1.5 Accepted TCA already needs the missing abstraction

ADR-0050 and the accepted TCA architecture require an exact multi-repository source lock covering Context, Devkit, Foundation, Runtime, and other sources actually used by selection. The lock records repository commit/tree, selected paths, and per-file content digests; a separate ActivationGeneration is built from that closure. CA-0 has already landed in qiven-context (PR #128 and its 2026-09-24 exit report), including a bounded CA-1 batch estimate. MVP-4's real H1 and RR-0 still gate CA-1.

If engineering builds continue to resolve repositories independently through CMake/shims while TCA constructs another exact source graph, Qiven will have two different multi-repository resolution systems:

~~~text
engineering build resolver -> sibling roots + per-consumer pins
cognition source resolver   -> TCA external source lock
~~~

That would preserve the same class of attention and consistency failure that TCA is intended to reduce.

---

## 2. Problem Classification

The current failure is larger than "shim + pin is ugly."

### 2.1 Topology is ambient

The workspace is inferred from directory placement and environment variables. The real dependency graph is not a first-class object.

### 2.2 Version selection is distributed

The selected revision of one shared dependency may appear in multiple CMakeLists, JSON manifests, templates, or operating documents.

### 2.3 Resolution and materialization are conflated

CMake decides both where a dependency is and whether its target already exists. Python shims decide both where an implementation is and which implementation will execute.

### 2.4 Compatibility is expressed as commit identity

A consumer often says "I require commit X" where its actual semantic requirement is closer to "I require Foundation public contract epoch 1 with capability Y."

Exact commit identity is still required for reproducibility, but it belongs to the resolved workspace generation. It should not substitute for the dependency contract.

### 2.5 LLM cognition pays the control-plane cost

The current participant repeatedly reconstructs:

- which checkout supplies one target;
- which SHA each consumer expects;
- whether a managed file is a copy, shim, or canonical implementation;
- whether changing a lower layer requires one or several re-pin commits;
- whether symbol absence means not implemented, wrong revision, dependency not added, wrong root, or target already existed;
- which external revisions TCA should bind.

This competes directly with domain reasoning and with the accepted TCA context budget.

---

## 3. Normative Model

Qiven SHALL distinguish four concerns.

### 3.1 Dependency declaration

A repository states what semantic layer or capability it requires, and each selected provider revision states which contracts it provides. An edge is satisfied only when the locked provider actually declares the required contract and any required integration gate passes; a matching string in two consumer manifests is not evidence of compatibility.

Examples:

~~~text
qiven-runtime requires qiven-foundation contract epoch 1
qiven-runtime requires qiven-context-draft frozen semantic contract v4
qiven-runtime requires sqlite3 package contract profile qiven-v1
~~~

This declaration is source-controlled with the repository. Existing legacy commits that predate the manifest MAY be described by a sealed, exact-revision-bound WR-0 census record only during WR-1/WR-2 shadow evaluation. Such a record is not a provider-authored contract and cannot authorize a cutover, an authoritative graph receipt, or TCA selector migration; it expires when that node's repository-owned declaration lands.

### 3.2 Workspace resolution

One resolver validates a selected set of exact repository revisions against every declaration in the workspace snapshot. An explicit lock transaction proposes revision movement; normal build and gate operations validate the complete effective declaration graph from exact Git objects (the base lock plus any explicit clean candidate overlay), then materialize and check worktrees only for the operation's closed projection. A candidate overlay must revalidate changed and incoming edges from its own revision, never reuse the base graph proof. The output is a WorkspaceGeneration. Build, tooling, and cognition each select a closed projection from this one validated universe; an unrelated node's unavailable or dirty checkout does not invalidate a consumer's operation when its locked declaration metadata is available and the node is outside that projection. Lock updates still require complete candidate-graph validation.

### 3.3 Materialization

The selected node is made available to a build, tool, or activation engine.

Examples include adding a CMake source tree once, locating an executable inside Toolchain, loading the exact Devkit Operator, or copying TCA source files into an ActivationGeneration.

Materialization does not re-resolve the dependency.

### 3.4 Process/service lifetime

A resident process such as RuntimeHost may separately be one-per-workspace or one-per-profile.

That is not package resolution and SHALL NOT be implemented by the dependency resolver.

---

## 4. Singleton Law

"Singleton" is overloaded today. The program adopts three precise meanings.

### 4.1 Resolution singleton

For one WorkspaceGeneration there is exactly one selected provider for each tuple within a declared resolution domain:

~~~text
(package-id, platform, variant, semantic-slot)
~~~

Two dependency declarations that cannot accept the same node make resolution fail. The manifest MUST name the domain and slot explicitly; it must not accidentally merge incompatible platforms, variants, or independently deployable processes.

### 4.2 Build singleton

Within one generated CMake build graph, a resolved source node is materialized once and every consumer links its namespaced target.

CMake target presence may prevent duplicate materialization only after the resolver has already proven every dependency constraint.

### 4.3 Runtime singleton

A process singleton such as one RuntimeHost for an active workspace/profile remains a Runtime concern with its own mutex, lease, IPC, and recovery rules.

These singleton classes MUST NOT be conflated.

---

## 5. Governance Laws

### WG-1 — Resolve before configure

No governed CMake configure may discover or choose a first-party repository revision. It receives a validated resolution.

### WG-2 — One revision decision per workspace node

Exact workspace revision selection is recorded once in WorkspaceGeneration, not repeated in every consumer.

### WG-3 — No ambient success

A sibling directory, PATH entry, CMAKE_PREFIX_PATH, Python import path, or environment variable cannot silently satisfy a governed dependency. Overrides are explicit development inputs and appear in the resolution receipt.

### WG-4 — Dependency validation precedes target existence

Every dependency edge is validated before any CMake target is created or general Operator task runs. The identity-checked resolver bootstrap is the sole pre-validation exception. if(TARGET ...) may deduplicate materialization; it cannot waive a constraint.

### WG-5 — Thin shims are allowed; resolver shims are not

A tiny launcher transferring control to the already-resolved Devkit is a UX mechanism.

A launcher that searches for Devkit, selects a checkout, interprets a pin, or falls back to sibling layout is dependency resolution and belongs in the workspace resolver.

### WG-6 — Dirty state is never authoritative dependency truth

Development overlays may point at dirty worktrees for local iteration, but authoritative gates and publication require clean exact commits or a separately sealed candidate-tree identity for every node they actually consume, including the target repository. An unrelated unmaterialized worktree does not make an operation dirty.

### WG-7 — One graph feeds engineering and cognition after a qualified migration

ADR-0050's CA-1 source lock is a current accepted obligation and SHALL proceed on its bounded schedule. Before replacing its revision selection, a workspace adapter MUST reproduce the complete selected repository set and every commit/tree, path filter, and per-file digest in shadow mode. Once accepted, TCA SHALL select those repository revisions from WorkspaceGeneration, then produce a self-contained external-source lock with the inherited commit/tree and selected-path digests. It MUST NOT choose a conflicting revision, weaken digest checks, or read a dirty checkout. Only changes to inputs of that selected source closure (or other accepted activation inputs) invalidate ActivationGeneration; a movement of an unrelated workspace node does not. A build/tool execution projection digest is receipt provenance, not an input to the TCA source-lock or ActivationGeneration hash; the source projection digest is computed solely from selected cognition-source inputs.

### WG-8 — Capability discovery is generation-bound

A capability surface, public header, architecture reference, or tool contract is interpreted against one exact WorkspaceGeneration.

"No symbol found" without generation identity is not sufficient evidence.

### WG-9 — Graph conflicts fail visibly

Conflicting compatibility requirements, missing nodes, revision mismatch, platform mismatch, duplicate semantic slots, stale generated material, and candidate declarations that introduce unresolved edges are typed failures.

### WG-10 — Cross-repository publication names the generation

Any acceptance evidence depending on more than one Qiven repository records its exact selected closure and candidate/main provenance. After migration it also names WorkspaceGeneration. A digest alone never substitutes for the auditable member revisions and source-content evidence required by that gate.

---

## 6. Relationship to Existing Decisions

### 6.1 ADR-0046 is refined rather than discarded

ADR-0046 correctly established the toolchain -> devkit -> all repositories infrastructure model and correctly used shim-plus-pin as movement away from vendored copies. Its own revisit text anticipated a unified workspace manifest.

This proposal declares the next step:

~~~text
vendored copies
    ->
shim + pin
    ->
explicit workspace resolver + generation lock
~~~

Shim-plus-pin remains valid migration history but ceases to be the target architecture.

### 6.2 TCA remains the cognition architecture

TCA decides which cognition applies to a task. Workspace Resolution selects the revision universe; each consumer projects only the nodes and paths it uses. ADR-0050's execution RuntimeGeneration remains separate. The selected TCA source lock must be self-contained so that the activation sidecar can be audited and rebuilt even if the workspace control repository is unavailable later.

~~~text
WorkspaceGeneration -> validated build/tool closure
                   -> selected TCA source lock (commits, trees, paths, digests)
                            -> ActivationGeneration -> task-specific cognition
RuntimeGeneration ------------------------------------^ (separate execution identity)
~~~

### 6.3 Devkit remains engineering-process owner

Devkit should own resolver implementation, schemas, validation rules, generated CMake integration, and operator UX.

The selected workspace graph itself is proposed to live in a dedicated minimal workspace control repository. That repository is a candidate control-plane owner, not an already approved canonical authority. Its revision and lock provenance must be bound to each receipt; merely moving the file does not solve bootstrap or governance.

---

### 6.4 Acceptance and migration authority

This PR is a proposal, not an amendment to accepted ADR-0046 or ADR-0050 by itself. Replacing the Devkit cross-repository pin law and changing TCA's repository selector require an owner-accepted root ADR and corresponding canonical changes. CA-0 is complete; the accepted CA-1 batch (at most three Runtime PRs, one Context policy-instance PR, and one Devkit schema PR, with its stall trigger) is not silently expanded or delayed by WR-0/WR-1. Workspace work can run alongside it. A formal re-deliberation is required before changing that schedule or claiming a new CA-2 prerequisite.

---

## 7. Non-Goals

Version 1 does not:

- create a public package registry;
- require semantic-version releases for every internal commit;
- require prebuilt Foundation or Runtime packages;
- replace CMake as the native build generator;
- require an immediate content-addressed store;
- merge Qiven into a monorepo;
- permit network fetch during configure;
- turn TCA into a build system;
- make Devkit a daemon;
- infer API compatibility automatically from Git history.

The first milestone is explicit closure and deterministic validation.

---

## 8. Expected Cognitive Effect

After migration, an engineering participant should not need to inspect several CMakeLists and Python shims just to answer:

~~~text
Which Foundation?
Which Devkit?
Which Context Draft?
Which third-party singleton?
Which exact source set does TCA mean?
Did another repo silently skip its pin because the target already existed?
~~~

The machine-readable answer becomes a WorkspaceGeneration plus a resolved graph and typed diagnostics.

That is the dependency equivalent of TCA's accepted principle: exact machine state should replace repeated reconstruction from ambient repository prose.

---

## 9. Program Exit

This program is complete only when:

1. all active cross-repository dependencies are declared in machine-readable repository manifests;
2. one workspace manifest and lock resolve the complete active graph;
3. CMake no longer performs sibling-path or Git-SHA dependency resolution;
4. Devkit and Toolchain no longer depend on consumer-specific root guessing as their normal path;
5. graph conflicts are caught before configure;
6. a shared dependency revision is selected once rather than copied into every consumer;
7. Runtime, Draft, Math, Foundation, Devkit, Toolchain, and the third-party singleton pass migration acceptance;
8. after a separate, evidenced migration, TCA selects the same relevant repository revisions while preserving its full source lock and separate ActivationGeneration;
9. authoritative gates bind receipts to the workspace generation;
10. the legacy shim-plus-pin path is deleted or retained only as an explicitly time-bounded compatibility path;
11. staged lock updates retain an executable old-generation materialization and exact rollback inputs until cutover evidence closes.

The end state is not fewer rules. It is that topology and dependency identity become mechanical facts, leaving human and LLM attention for semantics that cannot be mechanized.
