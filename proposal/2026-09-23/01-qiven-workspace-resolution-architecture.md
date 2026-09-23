# Qiven Workspace Resolution Architecture

## One Dependency Graph for Build, Tooling, and Cognition

> **Status:** Proposed architecture
> **Date:** 2026-09-23 UTC
> **Program:** Qiven Workspace Dependency Resolution
> **Primary implementation owner:** qiven-devkit
> **Workspace state owner:** proposed minimal qiven-workspace repository
> **Consumers:** all active Qiven repositories and Task Cognition Activation
> **Governance:** 00-qiven-workspace-dependency-resolution-program.md

## 0. Architectural Decision

Qiven SHALL introduce a two-level dependency system:

1. repositories publish dependency declarations describing semantic requirements;
2. a workspace resolver produces one immutable WorkspaceGeneration selecting exact repository revisions and local materialization identities for the complete graph.

CMake, Python tools, Devkit Operator entry points, and TCA consume that resolved generation. They do not each discover repositories independently.

The first implementation deliberately reuses local Git checkouts. A content-addressed artifact store may be added later, but it is not required to remove the current architectural defect.

---

## 1. Components

### 1.1 qiven-workspace

A new deliberately small repository is proposed as the workspace control-plane root:

~~~text
qiven-workspace/
  README.md
  workspace.json
  workspace.lock.json
  bootstrap/
    qiven-bootstrap.py
    qiven.cmd
~~~

The repository carries dependency-control data and bootstrap only. It owns no product semantics.

A neutral repository is required because no current product repository can own a graph that also contains itself without creating an ownership/bootstrap ambiguity.

### 1.2 Repository dependency manifest

Every participating repository carries:

~~~text
.qiven/dependencies.json
~~~

The file states semantic dependency edges, not local paths and not exact workspace peer commits.

Example for Runtime:

~~~json
{
  "schema": "qiven-dependencies-v1",
  "repository": "qiven-runtime",
  "dependencies": [
    {
      "id": "qiven-foundation",
      "kind": "first-party-source",
      "contract": "qiven-foundation-api-v1"
    },
    {
      "id": "qiven-context-draft",
      "kind": "first-party-source",
      "contract": "qiven-context-draft-v4"
    },
    {
      "id": "qiven-third-party-win",
      "kind": "third-party-singleton",
      "packages": ["sqlite3"]
    },
    {
      "id": "qiven-devkit",
      "kind": "tooling",
      "contract": "qiven-devkit-operator-v2"
    },
    {
      "id": "qiven-toolchain-win",
      "kind": "toolchain",
      "contract": "qiven-toolchain-win-v1"
    }
  ]
}
~~~

A contract identifier is explicit Qiven vocabulary. Version 1 does not require SemVer for internal repositories.

### 1.3 Workspace manifest

workspace.json defines the allowed node universe and stable repository identity.

Example:

~~~json
{
  "schema": "qiven-workspace-v1",
  "workspace_id": "qiven",
  "repositories": {
    "qiven-context": {
      "url": "https://github.com/JasonHuang3D/qiven-context.git"
    },
    "qiven-runtime": {
      "url": "https://github.com/JasonHuang3D/qiven-runtime.git"
    },
    "qiven-foundation": {
      "url": "https://github.com/JasonHuang3D/qiven-foundation.git"
    },
    "qiven-context-draft": {
      "url": "https://github.com/JasonHuang3D/qiven-context-draft.git"
    },
    "qiven-math": {
      "url": "https://github.com/JasonHuang3D/qiven-math.git"
    },
    "qiven-devkit": {
      "url": "https://github.com/JasonHuang3D/qiven-devkit.git"
    },
    "qiven-toolchain-win": {
      "url": "https://github.com/JasonHuang3D/qiven-toolchain-win.git",
      "platform": "windows"
    },
    "qiven-third-party-win": {
      "url": "https://github.com/JasonHuang3D/qiven-third-party-win.git",
      "platform": "windows"
    }
  }
}
~~~

Dependency edges remain owned by the repositories that require them; the workspace manifest does not become another architecture document.

### 1.4 Workspace lock

workspace.lock.json records the exact resolved closure.

Example:

~~~json
{
  "schema": "qiven-workspace-lock-v1",
  "workspace_id": "qiven",
  "generation": "sha256:<digest>",
  "nodes": {
    "qiven-foundation": {
      "commit": "<40-char-git-oid>",
      "tree": "<git-tree-oid>"
    },
    "qiven-context-draft": {
      "commit": "<40-char-git-oid>",
      "tree": "<git-tree-oid>"
    },
    "qiven-runtime": {
      "commit": "<40-char-git-oid>",
      "tree": "<git-tree-oid>"
    },
    "qiven-devkit": {
      "commit": "<40-char-git-oid>",
      "tree": "<git-tree-oid>"
    },
    "qiven-toolchain-win": {
      "commit": "<40-char-git-oid>",
      "tree": "<git-tree-oid>",
      "platform": "windows"
    },
    "qiven-third-party-win": {
      "commit": "<40-char-git-oid>",
      "tree": "<git-tree-oid>",
      "platform": "windows"
    }
  }
}
~~~

The generation digest is computed over canonical sorted resolution content. It excludes volatile absolute paths and timestamps.

---

## 2. Bootstrap Boundary

The resolver depends on Devkit while Devkit itself is a node in the graph. That cycle must be broken explicitly.

### 2.1 Bootstrap has one narrow job

qiven-workspace/bootstrap/qiven-bootstrap.py is standard-library-only and performs only:

1. locate the workspace root from explicit invocation context or a workspace marker;
2. parse and validate the lock's bootstrap subset;
3. locate the exact locked Devkit checkout;
4. verify its Git identity;
5. execute the Devkit resolver/operator.

It does not:

- resolve arbitrary dependency graphs;
- interpret repository-specific engineering policy;
- run repository gates;
- select a newer Devkit;
- fall back to an arbitrary sibling Devkit;
- mutate the workspace lock.

This is a compiler-bootstrap-style boundary: a very small stable stage loads the real implementation at an exact identity.

### 2.2 Repository launchers become pure UX shims

Existing repository-local qiven launchers may remain temporarily, but their only permitted behavior is:

~~~text
find workspace marker
-> invoke workspace bootstrap
-> pass current repository identity
~~~

They carry no Devkit root fallback and no consumer-local Devkit pin.

The architecture rejects shim-plus-pin as dependency resolution; it does not reject a trivial launcher for usability.

---

## 3. Resolution Algorithm

The Devkit resolver executes before CMake configure and before TCA source publication.

### Step 1 — Establish workspace identity

Load workspace.json and reject unknown schema, duplicate node, invalid repository identity, or unsupported platform.

### Step 2 — Load repository declarations

For every selected workspace node, read the exact revision's .qiven/dependencies.json.

A missing declaration is legal only for an explicitly declared leaf class such as a pure toolchain artifact repository.

### Step 3 — Build the complete directed graph

Construct every edge before selecting materialization.

Architectural cycles among semantic layers are errors unless an explicit cycle contract exists. The Workspace bootstrap to Devkit handoff is a bootstrap boundary, not a semantic dependency cycle.

### Step 4 — Validate compatibility constraints

Every incoming edge to a shared node is checked before any consumer builds.

Example:

~~~text
runtime -> foundation-api-v1
draft   -> foundation-api-v1
math    -> foundation-api-v1
~~~

can share one Foundation node.

If one consumer requires an incompatible contract, resolution fails even if CMake could technically produce a target.

### Step 5 — Select exact revisions

The workspace lock supplies the exact commits for normal build/gate operation. Revision movement is an explicit lock transaction, never a configure-time float.

### Step 6 — Validate local materialization

Map each locked node to a local checkout.

Version 1 MAY use the existing sibling workspace layout as one materialization mapping, but sibling placement is no longer the source of truth.

Validation checks:

- repository identity;
- exact commit;
- exact tree;
- clean state where authoritative operation requires it;
- platform and variant.

### Step 7 — Publish a resolution receipt

The receipt includes:

- workspace generation digest;
- manifest digest;
- lock digest;
- each node commit and tree;
- local materialization path;
- dirty or override state;
- resolver build identity;
- target repository;
- operation class.

Absolute paths belong in the receipt, not in WorkspaceGeneration identity.

### Step 8 — Generate consumer adapters

The resolver produces deterministic derivatives under Qiven's approved generated-artifact location.

Suggested forms:

~~~text
.generated-temp/workspace/<generation>/qiven-deps.cmake
.generated-temp/workspace/<generation>/tool-paths.json
.generated-temp/workspace/<generation>/source-closure.json
~~~

All derive from the same lock.

---

## 4. CMake Integration

### 4.1 CMake becomes a materializer, not a resolver

A consuming CMakeLists becomes structurally similar to:

~~~cmake
include("<resolved qiven-deps.cmake>")

qiven_workspace_require(qiven-foundation)
qiven_workspace_require(qiven-context-draft)
qiven_workspace_require(qiven-third-party-win PACKAGE sqlite3)

target_link_libraries(
    qiven-runtime
    PUBLIC
        qiven::foundation
        qiven::context_draft
    PRIVATE
        qiven::tp::sqlite3
)
~~~

There is no consumer-local:

- QIVEN_FOUNDATION_ROOT sibling discovery;
- Git rev-parse dependency selection;
- Foundation SHA literal;
- governed package search;
- network acquisition.

### 4.2 qiven_workspace_require

The generated adapter already knows the selected root and identity.

It only:

1. verifies the requested dependency exists in the validated resolution;
2. materializes the node once if its target is absent;
3. verifies expected public target names after materialization;
4. records the materialization in configure evidence.

It never chooses a revision.

### 4.3 The target-presence bug is structurally removed

Dependency constraints are checked during resolution before CMake runs.

Therefore target presence may safely mean only:

~~~text
the already-resolved node is materialized
~~~

It can no longer mean:

~~~text
skip this consumer's dependency validation
~~~

### 4.4 Source co-development remains supported

Qiven does not need to convert Foundation or Context Draft into prebuilt binary packages to obtain correct dependency semantics.

The generated adapter can still use add_subdirectory with an exact resolved source checkout.

That preserves:

- IDE source visibility;
- local source debugging;
- target usage requirements;
- offline configure;
- current source-first development.

The architectural change is moving resolution out of CMake, not forbidding source composition.

---

## 5. Development Overlays

A central lock must not make multi-repository development unusable.

### 5.1 Explicit local overlay

A developer MAY supply an untracked local file such as:

~~~text
.qiven-workspace.local.json
~~~

to redirect materialization or select an explicit candidate revision.

The resolver labels affected nodes as one of:

~~~text
resolved
candidate-clean
candidate-dirty
~~~

### 5.2 Publication law

- local compile/test MAY use candidate-clean or candidate-dirty overlays;
- a gate claiming exact cross-repository evidence MAY use candidate-clean when commit/tree are recorded;
- publication, acceptance, CI identity, RuntimeGeneration publication, and ActivationGeneration publication reject candidate-dirty dependencies;
- no override is invisible.

A later sealed-candidate-tree mechanism may relax this only through a separately accepted design.

---

## 6. Toolchain Integration

qiven-toolchain-win becomes a resolved workspace node.

Its internal toolchain.json remains authoritative for the versions and relative paths of CMake, CTest, clang-format, and other pinned executables.

The responsibility split is:

~~~text
Workspace resolver -> which qiven-toolchain-win revision?
Toolchain manifest  -> which executables inside that revision?
~~~

This removes duplicated toolchain_root sibling discovery from consumers without conflating workspace revision identity with tool-version inventory.

---

## 7. Third-Party Singleton Integration

qiven-third-party-win remains the canonical third-party repository and keeps all package provenance rules.

What changes is exact singleton revision selection.

Today multiple consumers can carry their own singleton SHA. After migration, WorkspaceGeneration selects the exact qiven-third-party-win revision once.

Package provenance still owns:

- upstream source/version identity;
- archive and file digests;
- patches;
- licensing;
- per-package target definitions.

Workspace Resolution does not replace third-party provenance.

---

## 8. Devkit Resolution

The current Context shim separates two facts:

~~~text
where Devkit is found
which Devkit revision the consumer expects
~~~

The new path binds them before execution:

~~~text
WorkspaceGeneration
     |
     +--> exact qiven-devkit commit/tree
              |
              +--> local materialization mapping
                       |
                       +--> bootstrap identity check
                                |
                                +--> execute Operator
~~~

A consumer cannot import a different sibling Devkit from the one named by WorkspaceGeneration.

Managed Operator snapshots may remain during migration. The target endpoint is one resolved Devkit implementation per WorkspaceGeneration.

---

## 9. TCA Integration

### 9.1 WorkspaceGeneration is the repository-identity parent

The accepted TCA architecture already requires exact external repositories and ActivationGeneration.

The revised composition is:

~~~text
WorkspaceGeneration
  exact repository commits/trees
        |
        +--> activation source projection
               exact selected paths + content digests
                    |
                    +--> ActivationGeneration
~~~

TCA may select a subset of files, but it does not independently decide which Runtime, Foundation, Devkit, or other repository revision is current.

### 9.2 TCA source-lock shape

The activation source lock should reference:

- WorkspaceGeneration digest;
- selected repository node ID;
- selected path;
- path content digest;
- capability-surface entry ID where applicable.

Repository revision identity is inherited from WorkspaceGeneration.

### 9.3 Capability discovery becomes generation-bound

A capability such as a public Foundation type is meaningful only together with the exact Foundation node in the current generation.

The task bundle can then expose both:

- the semantic capability;
- the exact source generation where it exists.

This is stronger than ambient symbol discovery and cheaper than asking the LLM to reconstruct repository state manually.

---

## 10. Workspace Generation Identity

The digest SHALL be deterministic.

Conceptually:

~~~text
sha256(
    workspace-schema-version
    + canonical-workspace-manifest
    + sorted(
        node-id,
        repository-identity,
        commit-oid,
        tree-oid,
        platform,
        variant,
        dependency-contract-digest
      )
)
~~~

Excluded:

- absolute local paths;
- timestamps;
- usernames;
- temporary directories;
- environment-variable spellings;
- receipt IDs.

Two machines resolving the same closure produce the same WorkspaceGeneration even when their checkout paths differ.

---

## 11. Failure Taxonomy

At minimum:

| Failure | Meaning |
|---|---|
| WorkspaceNotFound | no explicit workspace root or marker |
| UnknownNode | dependency references undeclared repository |
| MissingDeclaration | required dependency metadata absent |
| DependencyConflict | incoming contracts cannot share one resolved node |
| RevisionUnavailable | locked commit is not locally/materially available |
| RevisionMismatch | checkout HEAD differs from lock |
| TreeMismatch | tree identity differs from lock |
| DirtyDependency | authoritative operation sees unsealed changes |
| PlatformMismatch | node cannot satisfy current platform/variant |
| TargetContractMismatch | materialized CMake project lacks promised target |
| ToolContractMismatch | tooling node lacks promised executable/API |
| StaleGeneratedResolution | generated adapter does not match the lock |
| BootstrapDevkitMismatch | bootstrap found Devkit different from the lock |

Diagnostics name both the consumer edge and conflicting node.

---

## 12. Security and Hermeticity

The architecture preserves Qiven's no-network-at-configure law.

Normal configure/build:

- reads the exact lock;
- validates local materialization;
- performs no GitHub fetch;
- performs no package-manager discovery;
- performs no PATH-based governed dependency selection.

Acquisition and update remain explicit workspace operations and are separately governed by Devkit/Operator custody.

---

## 13. Authority Split

The following split is normative:

~~~text
repository dependency manifests    canonical declarations
workspace.json                     canonical node universe
workspace.lock.json                canonical resolved generation
generated CMake/tool/source files  rebuildable derivatives
local overlay                      non-canonical development input
resolution receipt                 evidence
~~~

The lock cannot be silently regenerated as a side effect of CMake configure.

---

## 14. Why Ambient CMake find_package Is Not the Resolver

The problem is not merely locating qiven::foundation.

Qiven must prove:

- every consumer accepted one compatible Foundation contract;
- one exact revision was selected;
- the whole cross-repository closure is coherent;
- TCA uses the same source universe;
- publication evidence names that closure.

Those are workspace-resolution responsibilities.

A generated CMake package surface may later be useful as a materialization mechanism, but ambient package search cannot be the authority.

---

## 15. Future Content-Addressed Store

After WorkspaceGeneration is accepted and measured, Qiven MAY add an immutable local store or Git-worktree cache keyed by repository tree/content identity.

That would allow multiple generations to coexist without mutating one shared checkout.

It is intentionally deferred from v1 because the present defect can be removed while preserving today's editable sibling repositories.

WorkspaceGeneration already separates logical identity from physical path, so this evolution remains compatible.
