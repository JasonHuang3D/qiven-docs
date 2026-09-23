# Qiven Workspace Resolution Migration and Acceptance

## Removing Shim-and-Pin Dependency Resolution Without Freezing MVP Progress

> **Status:** Proposed delivery and acceptance plan
> **Date:** 2026-09-23 UTC
> **Program:** Qiven Workspace Dependency Resolution
> **Architecture:** 01-qiven-workspace-resolution-architecture.md
> **Related accepted program:** Task Cognition Activation (TCA)

## 0. Delivery Principle

The migration SHALL remove dependency-resolution attention debt without creating a second unbounded infrastructure detour.

The program therefore migrates one dependency class at a time behind a dual-resolution proof. Substantive Runtime/TCA work should stop only at explicit entry/exit gates defined below, not for an open-ended workspace rewrite.

The first accepted workspace milestone is shadow validation: a machine-readable WorkspaceGeneration reproduces the closure of the current build and detects conflicts the current build can hide. Exclusive authority for revision selection comes later, after governed migration, standalone/CI proof, and TCA source-lock equivalence. ADR-0046 and ADR-0050 remain in force meanwhile.

---

## 1. Scope Inventory

The first implementation batch SHALL inventory every active resolver-shaped mechanism.

### First-party source edges

- Runtime -> Foundation
- Runtime -> Context Draft
- Context Draft -> Foundation
- Math -> Foundation

### Tooling edges

- Context -> Devkit Operator
- managed C++ repositories -> Devkit Operator or managed snapshot
- Devkit-managed repositories -> Toolchain

### Third-party edges

- Runtime -> qiven-third-party-win -> SQLite

### Cognition edges

- TCA activation publication -> exact Context, Devkit, Foundation, Runtime, and capability-source revisions with path filters and per-file digests (CA-1's accepted source-lock contract)

Each edge is classified as:

~~~text
semantic dependency
tool dependency
build-only dependency
test-only dependency
cognition-source dependency
bootstrap dependency
~~~

No new shim-plus-pin resolver is added after the inventory begins unless recorded as an explicit emergency compatibility exception with a removal trigger.

---

## 2. Delivery Stages

### WR-0 — Resolver census and sealed baseline

Reuse the already completed CA-0 source inventory and 2026-09-24 exit report as inputs; do not reopen CA-0 or charge the accepted CA-1 batch for this program. Create a machine-readable inventory of:

- current root variables;
- sibling fallbacks;
- per-consumer pins;
- CMake public targets;
- Operator/template copies;
- Toolchain resolver copies;
- TCA external-source requirements.

Capture the current successful graph at exact revisions, recording both selected commits and their relationship to canonical main or candidate branches. Identify provision contracts, operation-specific closure, and the control-repo bootstrap input.

**Exit:**

- every active cross-repository edge and its provided contract is represented;
- no unresolved "probably sibling" edge remains;
- the current graph digest can be reproduced twice from the same inputs;
- the CA-1 source inventory and its bounded batch schedule are not changed.

### WR-1 — Workspace schema and bootstrap

Create:

- minimal qiven-workspace repository;
- workspace.json;
- workspace.lock.json;
- qiven-dependencies-v1 schema;
- standard-library bootstrap that loads the exact locked Devkit;
- Devkit resolver command that validates the graph and emits a receipt.

The resolver is read-only with respect to product repositories in this stage. The workspace-control repository is still a proposal until its authority and review path are accepted; initial shadow receipts identify its exact candidate revision.

**Exit:**

- the current workspace resolves successfully;
- an intentionally wrong node SHA fails typed;
- missing or wrong Devkit fails before Operator import;
- WorkspaceGeneration digest is independent of local absolute paths;
- canonicalization golden vectors agree across independent implementations, including duplicate-key and generation-self-hash failures;
- a clean single-repository/CI invocation has a specified exact workspace-control input and fails visibly if it is absent.

### WR-2 — Shadow resolution against the existing build

Keep current root+pin CMake logic active, but run Workspace Resolution first.

For every legacy pin and for each source selected by the accepted CA-1 lock (once available), compare:

~~~text
legacy or CA-1 selected repository identity, commit/tree, and source-content digests
workspace candidate projection's corresponding identity, commit/tree, and digests
~~~

A mismatch is a gate failure.

**Exit:**

- all active C++ gates pass under shadow resolution;
- all legacy pins equal the corresponding selected workspace nodes; if the CA-1 source lock exists, every selected source path/digest is equal in the shadow projection;
- a synthetic conflict fixture proves resolver conflict detection independently of CMake target order.

### WR-3 — Foundation graph migration

Foundation is the first semantic-layer migration because it currently demonstrates the target-presence/pin-validation defect directly.

Change Runtime, Context Draft, and Math so that:

- their dependency manifests declare the Foundation requirement, while the exact selected Foundation revision declares a provided contract with provider-owned and consumer integration evidence;
- WorkspaceGeneration selects the exact Foundation revision once;
- CMake receives the resolved Foundation root;
- consumer CMakeLists contain no Foundation sibling discovery;
- consumer CMakeLists contain no Foundation Git SHA check;
- Foundation is materialized once per build graph.

Add a sealed fixture:

~~~text
runtime accepts Foundation contract A
draft requires incompatible Foundation contract B
~~~

Resolution must fail before configure even if Runtime would otherwise create qiven::foundation first.

**Exit:**

- no active consumer-local Foundation SHA remains;
- all affected gates pass in top-level and nested Visual Studio builds, and no unrelated target can spoof qiven::foundation;
- moving a compatible Foundation revision no longer requires mechanical consumer re-pin commits.

### WR-4 — Context Draft migration

Move Runtime -> Context Draft to the same model.

Context Draft remains independently versioned. Only location and exact revision selection move to WorkspaceGeneration.

**Exit:**

- Runtime CMake no longer resolves QIVEN_DRAFT_ROOT;
- no Runtime-local Draft SHA remains;
- clean bootstrap reproduces the exact Runtime/Draft/Foundation closure.

### WR-5 — Toolchain and third-party singleton migration

**Toolchain**

- remove duplicated sibling-root logic from copied per-repo toolchain.py as the normal path;
- obtain Toolchain root from WorkspaceGeneration;
- retain toolchain.json as the authoritative inventory of executables and versions inside the selected Toolchain node.

**Third-party**

- select qiven-third-party-win once in the workspace lock;
- preserve package provenance and package CMake targets;
- remove per-consumer singleton revision selection.

**Exit:**

- Toolchain or third-party workspace location changes require no consumer source edit;
- no governed dependency resolves from PATH/system package discovery;
- configure remains offline.

### WR-6 — Devkit Operator migration

Replace shim-plus-pin implementation discovery with exact workspace bootstrap.

During the compatibility window, a repository-local launcher may remain only if it:

- identifies the current repository/workspace;
- invokes workspace bootstrap;
- contains no Devkit path fallback;
- contains no consumer-local Devkit pin.

Managed Operator copies are retired only after the bootstrap path works in both local development and CI.

**Exit:**

- the Context-specific QIVEN_DEVKIT_ROOT -> sibling -> import path is no longer normal;
- every Operator invocation reports WorkspaceGeneration and exact Devkit node;
- wrong local Devkit revision fails before Operator code executes;
- template version drift can no longer silently select an older Operator implementation.

### WR-7 — TCA source-selector migration

CA-1 first delivers its already accepted exact external source lock and ActivationGeneration sidecar under ADR-0050. WR-7 then runs a shadow comparison of the complete selected Context/Devkit/Foundation/Runtime closure, including every path filter, commit/tree, and per-file digest. Only an owner-accepted governance amendment and passing equivalence evidence may switch TCA's repository selector to WorkspaceGeneration. The TCA source lock remains independently verifiable and self-contained; RuntimeGeneration remains separate.

**Exit:**

- task evidence names the parent WorkspaceGeneration as provenance, selected workspace projection, complete TCA source lock, execution RuntimeGeneration, and ActivationGeneration;
- movement of a selected source input changes or invalidates ActivationGeneration, while movement outside the selected closure does not;
- old-generation rebuild and rollback work from its exact source lock without depending on a mutable current workspace pointer;
- no TCA code reads a dirty sibling checkout as authoritative source;
- capability lookup binds to exact selected source revisions and digests;
- CA-1/CA-2 evidence and the existing bounded-batch stall trigger remain valid.

### WR-8 — Legacy removal and enforcement

Remove or prohibit normal-path:

- CMake sibling dependency discovery;
- consumer-local exact SHA selection for workspace peers;
- Devkit sibling implementation fallback;
- Toolchain sibling fallback inside copied resolver modules;
- managed copies whose only purpose is carrying dependency resolver logic.

Devkit gates add static checks for forbidden resolver patterns with narrowly documented bootstrap exceptions.

**Exit:**

- a new repository cannot accidentally reintroduce the old architecture;
- one governed workspace lock update selects a shared dependency only after selected provider compatibility and affected consumer integration gates pass;
- active repository gates and TCA acceptance profiles pass.

---

## 3. Acceptance Profiles

### Profile A — Graph completeness

Given the active workspace:

- every required cross-repository edge is declared;
- every declared node resolves;
- no hidden sibling fallback is needed;
- no governed CMake target appears without a resolved node.

**Pass criterion:** 100% of active governed edges appear in the resolution receipt.

### Profile B — Conflict detection

Construct a sealed graph with incompatible requirements on one shared dependency.

**Pass criterion:**

- resolver returns DependencyConflict;
- failure occurs before CMake configure;
- reversing consumer or target materialization order does not change the result.

This is the permanent regression test for the current if(NOT TARGET qiven::foundation) failure class.

### Profile C — Layout independence

Materialize the same locked revisions under a different workspace directory layout.

**Pass criterion:**

- WorkspaceGeneration digest is identical;
- builds/tests remain equivalent;
- no repository source edit is required;
- absolute path differences appear only in local evidence.

### Profile D — Ambient-state rejection

Test:

- unrelated sibling repository at wrong revision;
- misleading environment override;
- globally installed package with matching name;
- stale generated CMake adapter.

**Pass criterion:** none can silently change the selected node.

### Profile E — Re-pin elimination

Move Foundation to a new accepted compatible revision.

**Pass criterion:**

- one workspace lock transaction selects the new revision;
- product code changes only where semantics require them;
- consumer CMake files receive no mechanical SHA edits;
- no pin-ripple-only PR is required.

### Profile F — Build singleton

Build Runtime with Context Draft and Foundation.

**Pass criterion:**

- one Foundation source node is materialized;
- every consumer links the same qiven::foundation target;
- resolver evidence proves every incoming Foundation edge was validated before materialization.

### Profile G — Devkit identity

Invoke Qiven from at least Context, Runtime, and Foundation.

**Pass criterion:**

- every invocation reports the same locked Devkit node for one WorkspaceGeneration;
- replacing a sibling Devkit checkout with another revision cannot cause silent execution;
- bootstrap failure is explicit and typed.

### Profile H — TCA closure identity

After CA-1 exists, compare its accepted source lock against a workspace projection; after WR-7, publish ActivationGeneration from that projection.

**Pass criterion:**

- the self-contained TCA source lock lists repository commits/trees, selected path filters, concrete paths, and per-file digests, plus the projection digest after WR-7; parent WorkspaceGeneration appears in a separate provenance envelope;
- every selected repository revision exactly equals its workspace node after WR-7;
- an unrelated workspace node movement leaves the selected source-lock digest and ActivationGeneration unchanged;
- selected source changes invalidate the correct receipts while RuntimeGeneration remains independent;
- the old activation generation rebuilds from its exact pinned sources.

### Profile I — Standalone and CI bootstrap

Start from:

- the target repository;
- an explicitly supplied exact qiven-workspace control revision and lock (a lone product clone is insufficient);
- approved access to exact locked source repositories.

**Pass criterion:**

- the workspace materializes the exact dependency closure before configure;
- configure itself remains offline;
- resulting WorkspaceGeneration equals the declared lock;
- a missing or ambiguous workspace identity fails without changing the legacy self-contained gate until the replacement is accepted.

### Profile J — Cognitive burden reduction

This profile measures process effect and does not replace Profiles A-I or the accepted TCA utility protocol.

For a sealed cross-repository engineering task measure before and after migration:

- number of files read solely to reconstruct dependency location/version;
- number of mechanical peer-pin edits;
- number of distinct resolver implementations;
- task input tokens spent on dependency-topology explanation;
- dependency-resolution mistakes found in review.

**Pass target:**

- normal task cognition receives one bounded workspace graph projection instead of several consumer-specific resolver implementations;
- no mechanical peer-repin edits are required;
- Profiles A-I show no correctness regression.

Broader claims about LLM engineering quality remain governed by the accepted Cognitive Effectiveness Acceptance protocol.

### Profile K — Contract soundness and transaction recovery

Change a provider's declared surface without changing the consumer's requirement string, and stage a multi-repository update with one new source commit temporarily unavailable.

**Pass criterion:**

- missing or incompatible provider provision, absent target, and failing integration gate all deny the update typed before it becomes authoritative;
- the old generation remains runnable through partial landings and failed lock publication;
- a clean unmerged candidate receives a distinct CandidateWorkspaceGeneration and cannot be mistaken for canonical-main acceptance;
- the new generation is selected only after its entire transitive closure is available and validated.

---

## 4. Machine-Readable Evidence

Each authoritative gate after migration should emit or reference a receipt similar to:

~~~json
{
  "schema": "qiven-workspace-resolution-receipt-v1",
  "workspace_control_revision": "<git-oid>",
  "workspace_manifest_digest": "sha256:<digest>",
  "workspace_lock_digest": "sha256:<digest>",
  "workspace_generation": "sha256:<accepted-base-digest>",
  "effective_candidate_generation": null,
  "operation_projection_digest": "sha256:<digest>",
  "target_repository": "qiven-runtime",
  "target_revision": "<git-oid>",
  "target_revision_provenance": "canonical-main-or-candidate-ref",
  "resolver_devkit_revision": "<git-oid>",
  "nodes": [
    {
      "id": "qiven-foundation",
      "commit": "<git-oid>",
      "tree": "<tree-oid>",
      "declaration_digest": "sha256:<digest>",
      "state": "clean"
    }
  ],
  "legacy_resolution_used": false
}
~~~

Timestamps and absolute local paths may exist in the evidence envelope but do not affect WorkspaceGeneration identity.

---

## 5. Compatibility Window

Existing ADR-0046 paths remain normal until each class reaches its cutover; ADR-0050's CA-1 source lock remains authoritative until WR-7 proves equivalence and receives governance acceptance. Within a class's WR-2 dual-run window:

1. Workspace shadow validation runs before legacy build resolution.
2. Legacy resolution executes to prove equality and still supplies the actual build until cutover.
3. Any disagreement fails that class's migration gate; it does not make an otherwise valid CA-1 source-lock result false.
4. No new feature depends on a legacy-only capability.
5. Every legacy path has a named removal stage.
6. After cutover, legacy resolution is disabled except a documented stage-local rollback under §6; the compatibility window cannot be silently extended after WR-8.

This prevents dual resolution from becoming permanent architecture.

---

## 6. Rollback

Rollback is stage-local.

Before WR-8, a failed migration of one dependency class may temporarily return that class to legacy resolution while preserving resolver declarations and shadow evidence.

Rollback MUST NOT:

- delete dependency manifests already proved correct;
- weaken exact-revision evidence;
- permit silent fallback after a resolver disagreement;
- claim *workspace-unified* TCA selection while the accepted independent CA-1 source lock is in force; that lock retains its own exact closure claim.

A failed migration is evidence about implementation, not justification for restoring ambient dependency selection as the architectural endpoint.

---

## 7. Sequencing Against Current Runtime and TCA Work

This proposal does not unilaterally amend ADR-0050 or its CA-0 exit report. As of qiven-context main at 3ceac425be5d23cb1a1f7c47e078a0be49e20866, CA-0 is complete; the real MVP-4 H1 rerun and RR-0 implementation remain prerequisites for CA-1; substantive MVP-5 remains frozen until CA-2. CA-1 has a declared scope, resource ceiling (at most three Runtime PRs, one Context policy-instance PR, one Devkit schema PR), Profile A/B exit, and a stall trigger.

Scheduling law:

1. WR-0 and an initial WR-1 shadow prototype MAY proceed in parallel with MVP-4 H1 and RR-0, reusing CA-0 inventory. They are not new CA-1 entry gates.
2. CA-1 SHALL implement the accepted exact multi-repository source lock and immutable ActivationGeneration sidecar within its original bounded batch. It cannot wait indefinitely for a new workspace repository or trade its complete path digests for an opaque generation pointer.
3. WR-2 compares that source lock and current build pins with the candidate workspace projection when both exist. A mismatch blocks workspace migration, not the independently valid accepted CA-1 path.
4. WR-3 through WR-6 move build and tooling classes only after their own shadow/standalone/CI proofs; they do not silently extend CA-1 or postpone CA-2.
5. WR-7 changes TCA's repository selector only after the full source-lock equivalence proof and an owner-accepted governance amendment. It preserves the separate RuntimeGeneration and selective ActivationGeneration invalidation.
6. If a proposed workspace change actually requires altering ADR-0050's sequence or CA-1 ceiling, record the conflict and obtain root governance re-deliberation before implementing that change.

This keeps the workspace architecture an explicit follow-on improvement without converting an ongoing acceptance batch into an unbounded infrastructure rewrite.

---

## 8. Required Governance Follow-Up

If accepted, canonical Qiven governance should create:

1. an owner-accepted root ADR superseding the dependency-resolution portions of ADR-0046 while preserving its infrastructure-layer ownership model, and explicitly reconciling any change to ADR-0050;
2. a Devkit architecture document for Workspace Resolver;
3. machine-readable workspace and dependency schemas;
4. a tracked WR-0 through WR-8 migration obligation;
5. a revised cross-repository CMake convention stating that CMake consumes a resolved graph rather than performing resolution;
6. a TCA amendment, after WR-7 shadow equivalence and before WR-7 cutover, binding selected external repository identity to WorkspaceGeneration while preserving ADR-0050's complete source-lock fields and cache invalidation scope;
7. a static Devkit gate preventing new governed root+pin/sibling-resolution patterns after migration.

Historical ADRs and incident records remain unchanged. New decisions refer to shim-plus-pin as a superseded stage rather than rewriting history.

---

## 9. Definition of Done

Workspace Resolution becomes Qiven's normal dependency path only when:

- Profiles A-I and K pass at exact revisions (J is a measured process-effect profile);
- Runtime, Foundation, Draft, and Math no longer rely on consumer-local source pins;
- Toolchain and third-party revision identity come from WorkspaceGeneration;
- normal Devkit execution is generation-bound;
- the current target-presence validation hole has a permanent regression fixture;
- TCA uses a qualified WorkspaceGeneration projection as revision selector while publishing its full, self-contained external source lock;
- clean local and CI bootstrap can materialize the exact closure;
- no normal-path CMake configure or Python shim performs governed repository discovery;
- generated adapters are deterministic and rebuildable;
- failure diagnostics identify the dependency edge, selected operation projection, and workspace generation;
- migration evidence shows mechanical pin-ripple work was removed rather than moved into another family of copied files.

The success condition is simple:

> Qiven has one explicit answer to "which repository universe am I building, executing, and reasoning about?", and every subsystem consumes that answer instead of reconstructing its own.
