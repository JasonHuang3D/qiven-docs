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

The first accepted milestone is not "all repositories use a sophisticated package manager." It is:

> One machine-readable WorkspaceGeneration proves the same dependency closure that the current build uses, catches conflicts the current build can hide, and becomes the only authority for exact cross-repository revision selection.

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

- TCA activation publication -> exact Context, Devkit, Foundation, Runtime, and capability-source revisions

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

Create a machine-readable inventory of:

- current root variables;
- sibling fallbacks;
- per-consumer pins;
- CMake public targets;
- Operator/template copies;
- Toolchain resolver copies;
- TCA external-source requirements.

Capture the current successful graph at exact repository heads.

**Exit:**

- every active cross-repository edge is represented;
- no unresolved "probably sibling" edge remains;
- the current graph digest can be reproduced twice from the same inputs.

### WR-1 — Workspace schema and bootstrap

Create:

- minimal qiven-workspace repository;
- workspace.json;
- workspace.lock.json;
- qiven-dependencies-v1 schema;
- standard-library bootstrap that loads the exact locked Devkit;
- Devkit resolver command that validates the graph and emits a receipt.

The resolver is read-only with respect to product repositories in this stage.

**Exit:**

- the current workspace resolves successfully;
- an intentionally wrong node SHA fails typed;
- missing or wrong Devkit fails before Operator import;
- WorkspaceGeneration digest is independent of local absolute paths.

### WR-2 — Shadow resolution against the existing build

Keep current root+pin CMake logic active, but run Workspace Resolution first.

For every legacy pin compare:

~~~text
legacy selected revision
workspace selected revision
~~~

A mismatch is a gate failure.

**Exit:**

- all active C++ gates pass under shadow resolution;
- all legacy pins equal the WorkspaceGeneration;
- a synthetic conflict fixture proves resolver conflict detection independently of CMake target order.

### WR-3 — Foundation graph migration

Foundation is the first semantic-layer migration because it currently demonstrates the target-presence/pin-validation defect directly.

Change Runtime, Context Draft, and Math so that:

- their dependency manifests declare the Foundation contract;
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
- all affected gates pass;
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

### WR-7 — TCA source-lock unification

ActivationGeneration publication receives WorkspaceGeneration and derives external repository identity from it.

The TCA source lock still records selected path content digests, but does not choose repository revisions independently.

**Exit:**

- task evidence names both WorkspaceGeneration and ActivationGeneration;
- workspace revision movement deterministically changes or invalidates activation generation;
- no TCA code reads a dirty sibling checkout as authoritative source;
- capability lookup is bound to the exact workspace generation.

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
- one workspace graph update is sufficient to move a compatible shared dependency;
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

Publish ActivationGeneration from the same workspace.

**Pass criterion:**

- TCA source lock references WorkspaceGeneration;
- selected external paths are content-digested;
- repository revisions exactly equal the workspace lock;
- no independent repository revision choice exists in TCA.

### Profile I — Standalone and CI bootstrap

Start from:

- the target repository;
- qiven-workspace bootstrap identity;
- approved access to exact locked source repositories.

**Pass criterion:**

- the workspace materializes the exact dependency closure before configure;
- configure itself remains offline;
- resulting WorkspaceGeneration equals the declared lock.

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

---

## 4. Machine-Readable Evidence

Each authoritative gate after migration should emit or reference a receipt similar to:

~~~json
{
  "schema": "qiven-workspace-resolution-receipt-v1",
  "workspace_generation": "sha256:<digest>",
  "target_repository": "qiven-runtime",
  "target_revision": "<git-oid>",
  "resolver_devkit_revision": "<git-oid>",
  "nodes": [
    {
      "id": "qiven-foundation",
      "commit": "<git-oid>",
      "tree": "<tree-oid>",
      "state": "clean"
    }
  ],
  "legacy_resolution_used": false
}
~~~

Timestamps and absolute local paths may exist in the evidence envelope but do not affect WorkspaceGeneration identity.

---

## 5. Compatibility Window

Old and new mechanisms MAY coexist only during shadow migration.

Rules:

1. Workspace Resolution runs first.
2. Legacy resolution may execute only to prove equality.
3. Any disagreement fails the gate.
4. No new feature depends on a legacy-only resolution capability.
5. Every legacy path has a named removal stage.
6. The compatibility window cannot be extended silently after WR-8 entry.

This prevents dual resolution from becoming permanent architecture.

---

## 6. Rollback

Rollback is stage-local.

Before WR-8, a failed migration of one dependency class may temporarily return that class to legacy resolution while preserving resolver declarations and shadow evidence.

Rollback MUST NOT:

- delete dependency manifests already proved correct;
- weaken exact-revision evidence;
- permit silent fallback after a resolver disagreement;
- claim TCA closure identity if TCA has returned to independent revision resolution.

A failed migration is evidence about implementation, not justification for restoring ambient dependency selection as the architectural endpoint.

---

## 7. Sequencing Against Current Runtime and TCA Work

The workspace program should not silently supersede the accepted TCA roadmap.

Recommended scheduling law:

1. Complete the smallest WR-0 and WR-1 foundation before CA-1 builds permanent multi-repository source-lock machinery.
2. Run WR-2 shadow resolution while unrelated bounded corrective work continues.
3. Require WR-3 Foundation conflict proof before treating workspace dependency identity as authoritative enough to replace ad-hoc source resolution.
4. Land WR-7 before CA-2 claims cognition activation is bound to one exact cross-repository source universe.
5. Do not hold unrelated Runtime functional work behind WR-5 or WR-6 if that work does not depend on those migrations.

This closes the source-lock defect before TCA hardens around it without turning dependency cleanup into an unbounded prerequisite.

---

## 8. Required Governance Follow-Up

If accepted, canonical Qiven governance should create:

1. a root ADR superseding the dependency-resolution portions of ADR-0046 while preserving its infrastructure-layer ownership model;
2. a Devkit architecture document for Workspace Resolver;
3. machine-readable workspace and dependency schemas;
4. a tracked WR-0 through WR-8 migration obligation;
5. a revised cross-repository CMake convention stating that CMake consumes a resolved graph rather than performing resolution;
6. a TCA amendment binding external repository identity to WorkspaceGeneration;
7. a static Devkit gate preventing new governed root+pin/sibling-resolution patterns after migration.

Historical ADRs and incident records remain unchanged. New decisions refer to shim-plus-pin as a superseded stage rather than rewriting history.

---

## 9. Definition of Done

Workspace Resolution becomes Qiven's normal dependency path only when:

- Profiles A-I pass at exact revisions;
- Runtime, Foundation, Draft, and Math no longer rely on consumer-local source pins;
- Toolchain and third-party revision identity come from WorkspaceGeneration;
- normal Devkit execution is generation-bound;
- the current target-presence validation hole has a permanent regression fixture;
- TCA uses WorkspaceGeneration as repository source identity;
- clean local and CI bootstrap can materialize the exact closure;
- no normal-path CMake configure or Python shim performs governed repository discovery;
- generated adapters are deterministic and rebuildable;
- failure diagnostics identify both dependency edge and workspace generation;
- migration evidence shows mechanical pin-ripple work was removed rather than moved into another family of copied files.

The success condition is simple:

> Qiven has one explicit answer to "which repository universe am I building, executing, and reasoning about?", and every subsystem consumes that answer instead of reconstructing its own.
