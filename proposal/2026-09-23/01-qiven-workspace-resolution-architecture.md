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

1. repositories publish required and provided contract declarations at exact source revisions;
2. a workspace lock records one validated, immutable revision snapshot, with a WorkspaceGeneration independent of local materialization paths; operation-specific build, tooling, and cognition closures project from that snapshot.

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

A separate minimal control repository is the proposed authority for dependency selection after owner acceptance. It must not become the authority for qiven-context's canonical cognition or for execution RuntimeGeneration. The control repository's own Git revision is attested in receipts outside its lock, avoiding a self-referential hash.

### 1.2 Repository dependency manifest

Every participating repository carries:

~~~text
.qiven/dependencies.json
~~~

The file states provided contracts and required semantic dependency edges, not local paths and not exact workspace peer commits. A selected provider revision must declare a matching provision; syntactic matching is necessary but does not prove ABI or behavioral compatibility. Changes to a provision require provider-owned compatibility evidence and the consumer integration gates that exercise it.

Example for Runtime:

~~~json
{
  "schema": "qiven-dependencies-v1",
  "repository": "qiven-runtime",
  "supported_platforms": ["windows"],
  "provides": [{"contract": "qiven-runtime-api-v1", "targets": ["qiven::runtime"]}],
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
      "platform": "windows",
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
      "platform": "windows",
      "contract": "qiven-toolchain-win-v1"
    }
  ]
}
~~~

A contract identifier is explicit Qiven vocabulary. Version 1 does not require SemVer for internal repositories. The resolver rejects an undeclared provision, a target-contract mismatch, or a change that lacks the contract's prescribed validation; a shared name alone never authorizes a revision upgrade. Third-party package and tool contracts similarly identify their actual provided package/tool surfaces, with their own provenance evidence. The sample declares the initial governed native build as Windows-only: current Runtime CI also dispatches Linux/macOS, but the current qiven-third-party-win SQLite target links advapi32 unconditionally. A workflow matrix is not proof of a supported platform. Other platforms require explicit providers, platform-scoped edges, materialization, and passing gates before being claimed; existing authoritative CI obligations cannot be dropped merely by labeling a platform unsupported.

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
    },
    "qiven-docs": {
      "url": "https://github.com/JasonHuang3D/qiven-docs.git",
      "selection": "only-when-referenced"
    }
  }
}
~~~

Dependency edges remain owned by the repositories that require them; the workspace manifest does not become another architecture document. The example lists a baseline universe, not an assertion that every listed repository enters every build or activation. qiven-docs enters a TCA source closure only when policy explicitly references its deliberation records, consistent with the CA-0 inventory.

### 1.4 Workspace lock

workspace.lock.json records exact selected revisions for the declared snapshot. The following is an illustrative excerpt, not a complete valid lock; a real lock includes every selected node (including Context and Math where selected), a digest of each revision's dependency/provision declaration, and all required platform/variant/slot identities.

Example excerpt:

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

The generation digest is computed over canonical resolution content as specified in §10. It excludes volatile absolute paths and timestamps. Each selected node's lock metadata binds the origin, Git blob identity, and content digest of its dependency/provision declaration (or an explicit leaf classification). For legacy revisions with no .qiven/dependencies.json, WR-1/WR-2 may bind a sealed WR-0 census record in the workspace-control tree to that exact node commit/tree as shadow-only evidence. That origin is explicit in the generation digest and can never count as a repository-owned provision at cutover. A receipt also binds the control repository revision and its manifest/lock blob digests; the control repository is not recursively listed as a node in its own lock.

---

## 2. Bootstrap Boundary

The resolver depends on Devkit while Devkit itself is a node in the graph. That cycle must be broken explicitly.

### 2.1 Bootstrap has one narrow job

qiven-workspace/bootstrap/qiven-bootstrap.py is standard-library-only and performs only:

1. locate exactly one workspace root from an explicit invocation context or an unambiguous marker;
2. parse and validate the lock's bootstrap subset;
3. map the effective Devkit node (base lock or explicit clean candidate overlay) to an explicitly supplied or registered local checkout (a path is a locator, not a selector);
4. verify repository identity, commit/tree, and the required clean state before import;
5. execute only the locked Devkit resolver in preflight mode. Once it validates the complete declaration graph and the Devkit provision, release the Operator command path.

It does not:

- resolve arbitrary dependency graphs;
- interpret repository-specific engineering policy;
- run repository gates;
- select a newer Devkit;
- fall back to an arbitrary sibling Devkit;
- mutate the workspace lock.

This is a compiler-bootstrap-style boundary: a very small stable stage loads the exact resolver implementation before its own graph can be validated. This is the sole bootstrap exception to the resolve-before-execute law; identity and entrypoint integrity are checked before import, and no general Operator task runs until graph validation succeeds.

### 2.2 Repository launchers become pure UX shims

Existing repository-local qiven launchers may remain temporarily, but their only permitted behavior is:

~~~text
find workspace marker
-> invoke workspace bootstrap
-> pass current repository identity
~~~

They carry no Devkit root fallback and no consumer-local Devkit pin.

The architecture rejects shim-plus-pin as dependency resolution; it does not reject a trivial launcher for usability.

### 2.3 Standalone and trust bootstrap

A clean single-repository clone cannot derive a unique authoritative workspace lock without an additional trusted input. CI or an operator supplies an exact workspace-control revision/lock and permitted source acquisition; local launchers may find a unique nearby marker, then validate its identity and report it. The operator- or CI-supplied exact workspace-control revision is the trust root of the whole chain: the bootstrap and the launchers validate identity downward from it and never derive trust upward from ambient state. Preflight may acquire declaration Git objects for the complete locked graph and full worktrees only for the selected operation closure; configure itself remains offline. Ambiguous, missing, or changed workspace identity fails visibly. The bootstrap never treats an arbitrary sibling checkout or environment value as a revision decision. ADR-0046's self-contained gate path remains in force for repositories until this exact standalone/CI proof passes; no managed snapshot is deleted merely because a new launcher exists. Control-repo branch state and product-repo candidate branches are reported separately, and neither changes qiven-context's canonical Git authority.

---

## 3. Resolution and Update Algorithm

Normal build/gate operation validates a committed lock; it never floats to a newer sibling revision. The Devkit resolver runs before CMake configure or TCA source publication.

### Step 1 — Establish the control identity

Load an exact workspace-control revision, workspace.json, and workspace.lock.json. Reject unknown schemas, duplicate JSON keys/node IDs, invalid platform declarations, and ambiguous workspace roots. A valid node for a different platform does not invalidate the whole lock; an operation lacking a supported platform-specific closure fails typed. Record control commit plus manifest and lock blob digests.

### Step 2 — Validate locked selection

For every node in the effective selection (base lock plus explicit candidate overlays), validate stable repository identity, exact commit and tree objects, declared platform/variant/slot, and availability of the repository-owned declaration blob through that tree (or an explicit leaf classification; WR-1/WR-2 have the labeled shadow-only census exception in Step 3). A changed candidate commit/tree supplies its own declaration blob and digest; the base node's declaration is not reused. Materialization maps an already selected node to a path; it does not select a revision. Do not require a checkout of every node for an unrelated operation. Before consuming any projected source, verify its local Git object, checkout HEAD/tree, and required clean state against the lock or the separately identified clean candidate. An unrelated dirty/missing worktree is not an operation failure when the node's locked declaration object is available and it is outside that operation's closure.

### Step 3 — Load declarations from those exact revisions

Read .qiven/dependencies.json as a Git blob reached through each effective source revision's exact commit/tree, verify its recorded blob identity and digest, and construct every required edge and provided contract before materialization. Candidate revisions can add/remove requirements or provisions, so validate every effective incoming edge and reject unknown nodes or contracts before candidate admission. For WR-1/WR-2 only, a legacy node lacking that file may use the sealed WR-0 census declaration stored in the exact workspace-control tree, keyed to its legacy commit/tree and labeled shadow-only. It cannot emit an authoritative graph receipt, prove provider compatibility for a cutover, or be used by WR-7. Normal operation needs the complete graph's exact declaration objects, not full unrelated worktrees. Unknown or undeclared providers, incomplete transitive closure, missing declared targets/packages, and architectural cycles fail typed; an explicit bootstrap edge is not a semantic-layer cycle.

### Step 4 — Validate every consumer edge

For each required edge in every declared platform/variant domain, check provider identity, contract provision, platform/variant/slot, and any required compatibility proof. Validate all incoming edges even when one CMake target already exists. A platform-specific node outside the current operation remains graph-valid; an operation with no matching closed provider set fails PlatformMismatch before configure. A contract string cannot replace provider gates or integration evidence.

### Step 5 — Select an operation projection

The committed snapshot has one base WorkspaceGeneration; an explicit clean revision overlay derives a distinct CandidateWorkspaceGeneration and a separately validated effective full graph. A build, tool invocation, or activation request has an operation execution projection: the closed set of nodes it executes or reads, including required tool/bootstrap nodes, with a versioned edge-filter policy and digest. It cannot omit an input the operation actually reads. TCA has a separate activation source projection containing only cognition sources actually selected by its accepted policy, with exact repository commits/trees, path filters, concrete paths, and per-file digests. Its source projection digest is computed from that content with a domain-separated, canonical encoding that excludes the digest field itself. A Toolchain node used to run the activator, but not selected as a cognition source, belongs to the operation projection only. Unrelated node movement changes neither selected projection.

### Step 6 — Publish a resolution receipt

The full-graph validation receipt enumerates every edge, declaration origin, and validation result in the effective graph and records whether it is base or candidate; a candidate receipt never cites the base graph proof as though it validated new declarations. A receipt containing a WR-1/WR-2 census declaration is labeled shadow-only and cannot back an authoritative operation. An operation receipt references the graph digest and lists the operation projection, its digest, all consumed node commits/trees and declaration digests, and the separate TCA source projection digest when applicable. It also records the control commit, manifest/lock digests, accepted base WorkspaceGeneration, effective candidate generation where applicable, local paths, candidate/main provenance, dirty/override status of consumed nodes, resolver build identity, target repository, and operation class. Absolute paths belong only in local evidence. A receipt is final only after adapter and materialization validation passes.

### Step 7 — Generate and validate adapters

Verify materialized worktrees and then generate deterministic CMake/tool/source adapters under .generated-temp/workspace/<generation>/<projection>/, the approved generated-artifact root. Each adapter embeds the effective generation and operation projection identities. A controlled configure rejects missing/stale adapters and a target whose producer is not the validated node; the final receipt records that check.

### Explicit lock update transaction

Revision movement is a separate proposal: stage all new source commits; read their declarations; validate the complete candidate graph and affected integration gates; create a new lock and generation; compare old/new closures; publish the workspace-control commit only through its governed review. Multi-repository commits cannot land atomically. Stage changed nodes in distinct checkouts or Git worktrees while preserving the old selected worktrees, exact Devkit/bootstrap, and lock objects until the rollback window closes; never switch the sole old checkout and still claim that generation is runnable. After the new source revisions and lock are reachable and verified, cut over atomically at the workspace pointer. Candidate clean revisions may be validated with provenance without pretending that an unmerged branch is canonical main. Failure retains the old runnable generation and produces a typed receipt; it never falls back to a different checkout.

---

## 4. CMake Integration

### 4.1 CMake becomes a materializer, not a resolver

A consuming CMakeLists becomes structurally similar to:

~~~cmake
if(NOT DEFINED QIVEN_RESOLUTION_FILE OR QIVEN_RESOLUTION_FILE STREQUAL "")
    message(FATAL_ERROR "run the workspace preflight and pass its exact adapter")
endif()
include("${QIVEN_RESOLUTION_FILE}")

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

The Devkit preflight supplies QIVEN_RESOLUTION_FILE to the approved CMake configure preset's environment; the preset maps it to the cache variable. The governed entry remains cmake --preset (and the Operator gate); no hand-written cmake -D... invocation is introduced. The adapter binds the exact target candidate as well as resolved dependencies, and the gate rejects a direct configure lacking validated preflight evidence. Build/test presets consume the resulting configured graph.

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
2. checks that the requesting consumer has a validated direct edge and that the adapter's lock/projection identity matches preflight;
3. materializes the selected node once, or verifies that an existing namespaced target was produced by that same selected node;
4. verifies promised public target names, usage requirements, and package targets after materialization;
5. records the producer and materialization in configure evidence.

It never chooses a revision or treats an unrelated pre-existing CMake target as satisfying the edge. For qiven-third-party-win it materializes the selected package subdirectory, preserving the verified Visual Studio project-reference behavior and package provenance; the adapter must not change EXCLUDE_FROM_ALL semantics without a generator-level regression gate.

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

to redirect materialization or select an explicit candidate revision. A path-only relocation of the same locked objects leaves WorkspaceGeneration unchanged. Selecting a different clean commit (including the target repository's own PR head) requires reading its own Git commit/tree and dependency/provision manifest, recomputing the effective complete graph including all changed and incoming edges, and validating it before producing a distinct **CandidateWorkspaceGeneration** by the same byte algorithm. A candidate with a newly required but unselected provider fails; it never impersonates the committed lock's generation or reuses the base graph proof. Dirty worktrees have only a local transient identity, never an authoritative immutable generation; local compile/test preflight evaluates their current declaration bytes as explicitly unsealed evidence, while authoritative validation refuses them. Every adapter and receipt distinguishes accepted base, effective candidate, and source ref. A dirty checkout outside the operation projection does not contaminate its evidence; declaration metadata still comes from exact locked objects.

The resolver labels affected nodes as one of:

~~~text
resolved
candidate-clean
candidate-dirty
~~~

### 5.2 Publication law

- local compile/test MAY use candidate-clean or candidate-dirty overlays;
- a gate claiming exact cross-repository candidate evidence MAY use candidate-clean only under its distinct CandidateWorkspaceGeneration, with commit/tree, branch/ref, control revision, and every selected source recorded; that evidence does not claim canonical-main status;
- publication, acceptance, CI identity, RuntimeGeneration publication, and ActivationGeneration publication reject candidate-dirty dependencies in the consumed closure (including the target) and separately apply their existing candidate-versus-canonical rules;
- no override is invisible.

WR-7 must define how a newly accepted Context revision becomes eligible for new task activations, how the active workspace pointer advances, and when an older lock is reserved for in-flight work or rollback. A pinned but stale Context lock must never silently claim current cognition. This freshness/admission policy is a root governance decision and must preserve existing Context mutation authority; it must not introduce an unapproved owner approval step for routine source movement.

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

Before migration, do not presume Devkit is already one effective implementation across all repositories: qiven-context records an older exact Devkit pin, while C++ repositories still carry managed Operator snapshots. WR-0 records their exact identities and WR-6 reconciles any split before issuing a single-Devkit acceptance claim.

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

ADR-0050 and the CA-0 exit report already require CA-1 to build a complete external source lock from Context, Devkit, Foundation, Runtime, and any additional source actually used by selection, with repository commit/tree, path filter, and per-file digest. CA-1 may implement that bounded accepted design before Workspace Resolution is available. WR-7 is a later selector migration, not permission to replace that lock with one opaque workspace digest.

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

After a shadow comparison and accepted migration, TCA may select a subset of files but takes the revision of each selected repository from the workspace lock. Until then, the accepted CA-1 selector and lock remain authoritative. The activation source projection must contain only selected cognition inputs; a Toolchain, Math, or unrelated third-party movement does not rotate ActivationGeneration merely because the full WorkspaceGeneration or the activator's operation execution projection changed. The parent workspace and operation projection digests are provenance in a receipt envelope, not inputs to the selected source-lock or ActivationGeneration content digest.

### 9.2 TCA source-lock shape

The activation source lock SHALL contain, for each selected repository and file:

- activation-source projection digest (after migration), derived only from the selected source records; the parent WorkspaceGeneration and operation execution projection digest are separately recorded in the provenance envelope, outside the source-lock/ActivationGeneration content hash;
- stable repository node ID, repository identity, exact commit and tree;
- selected path filter, concrete selected path, and per-file content digest;
- capability-surface entry ID where applicable.

The actual commit/tree values remain present, not merely inherited by pointer. The lock is independently verifiable and rebuildable from exact local Git objects. Context's canonical bundle/execution RuntimeGeneration and activation sidecar remain separate. Cache keys retain every input required by the accepted TCA architecture (task, both existing generation IDs, complete selected source lock, policy, consumer profile, renderer build, budget, and live evidence); the workspace-wide or operation execution projection digest alone must not cause unrelated invalidation. Validation still proves that each selected revision belongs to the attested parent WorkspaceGeneration when a migrated receipt is issued.

### 9.3 Capability discovery becomes generation-bound

A capability such as a public Foundation type is meaningful only together with the exact Foundation node in the current generation.

The task bundle can then expose both:

- the semantic capability;
- the exact source generation where it exists.

This is stronger than ambient symbol discovery and cheaper than asking the LLM to reconstruct repository state manually.

---

## 10. Workspace Generation Identity

The digest SHALL be deterministic and free of self-reference. Version 1 defines the byte algorithm before implementation: validate the strict schema, reject duplicate keys and numbers outside the schema's integer vocabulary, encode the manifest and lock *without the generation field* using one specified canonical JSON algorithm (RFC 8785), prefix a domain-separated schema identifier, then SHA-256 those bytes. The lock covers every selected node's stable repository identity, commit/tree, platform/variant/slot, declaration origin, and exact dependency/provision blob identity and digest. The resolver recomputes and compares the stored generation; independent implementations and golden byte vectors must agree. The effective candidate generation includes all changed node commits/trees and declaration origins/blob identities/digests; no candidate is admitted against the base graph receipt. Operation execution and activation source projections have distinct domain-separated digest schemas; golden vectors prove that a tool-only change rotates only the applicable operation projection, never the unchanged activation source projection. The workspace-control commit and its manifest/lock blob digests are receipt provenance, not recursive generation inputs.

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
| RevisionUnavailable | required declaration object or consumed source revision is unavailable |
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

The lock cannot be silently regenerated as a side effect of CMake configure. The accepted qiven-context Git authority and RuntimeGeneration remain distinct from the workspace control repository. Before owner acceptance of a superseding ADR, ADR-0046 and ADR-0050 govern the existing pins and TCA source lock respectively.

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

Version 1 uses distinct local checkouts or Git worktrees for the old and candidate selections during a lock update, retaining the old exact checkout/Devkit and lock for an explicit rollback window. This is the minimum needed to make the old generation runnable through a partial cross-repository landing.

After WorkspaceGeneration is accepted and measured, Qiven MAY add an immutable content-addressed store or managed Git-worktree cache keyed by repository tree/content identity. That optimization is deferred; the explicit side-by-side staging invariant is not.

WorkspaceGeneration already separates logical identity from physical path, so this evolution remains compatible.
