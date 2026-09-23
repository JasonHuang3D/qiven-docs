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

A contract identifier is explicit Qiven vocabulary. Version 1 does not require SemVer for internal repositories. The resolver rejects an undeclared provision, a target-contract mismatch, or a change that lacks the contract's prescribed validation; a shared name alone never authorizes a revision upgrade. Third-party package and tool contracts similarly identify their actual provided package/tool surfaces, with their own provenance evidence.

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

The generation digest is computed over canonical resolution content as specified in §10. It excludes volatile absolute paths and timestamps. A receipt also binds the control repository revision and its manifest/lock blob digests; the control repository is not recursively listed as a node in its own lock.

---

## 2. Bootstrap Boundary

The resolver depends on Devkit while Devkit itself is a node in the graph. That cycle must be broken explicitly.

### 2.1 Bootstrap has one narrow job

qiven-workspace/bootstrap/qiven-bootstrap.py is standard-library-only and performs only:

1. locate exactly one workspace root from an explicit invocation context or an unambiguous marker;
2. parse and validate the lock's bootstrap subset;
3. map the locked Devkit node to an explicitly supplied or registered local checkout (a path is a locator, not a selector);
4. verify repository identity, commit/tree, and the required clean state before import;
5. execute the locked Devkit resolver/operator.

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

### 2.3 Standalone and trust bootstrap

A clean single-repository clone cannot derive a unique authoritative workspace lock without an additional trusted input. CI or an operator supplies an exact workspace-control revision/lock and permitted source acquisition; local launchers may find a unique nearby marker, then validate its identity and report it. Ambiguous, missing, or changed workspace identity fails visibly. The bootstrap never treats an arbitrary sibling checkout or environment value as a revision decision. ADR-0046's self-contained gate path remains in force for repositories until this exact standalone/CI proof passes; no managed snapshot is deleted merely because a new launcher exists. Control-repo branch state and product-repo candidate branches are reported separately, and neither changes qiven-context's canonical Git authority.

---

## 3. Resolution and Update Algorithm

Normal build/gate operation validates a committed lock; it never floats to a newer sibling revision. The Devkit resolver runs before CMake configure or TCA source publication.

### Step 1 — Establish the control identity

Load an exact workspace-control revision, workspace.json, and workspace.lock.json. Reject unknown schemas, duplicate JSON keys/node IDs, unsupported platforms, and ambiguous workspace roots. Record control commit plus manifest and lock blob digests.

### Step 2 — Validate locked selection

For every selected node, validate stable repository identity, exact commit and tree, declared platform/variant/slot, and local availability. Materialization maps an already selected node to a path; it does not select a revision. A missing declaration is legal only for an explicit leaf class. The Git object and worktree must agree with the locked commit/tree; authoritative paths reject dirty state.

### Step 3 — Load declarations from those exact revisions

Read .qiven/dependencies.json from each locked source revision, verify its recorded digest, and construct every required edge and provided contract before materialization. Unknown or undeclared providers, incomplete transitive closure, missing targets/packages, and architectural cycles fail typed; an explicit bootstrap edge is not a semantic-layer cycle.

### Step 4 — Validate every consumer edge

For each required edge, check provider identity, contract provision, platform/variant/slot, and any required compatibility proof. Validate all incoming edges even when one CMake target already exists. A contract string cannot replace provider gates or integration evidence.

### Step 5 — Select an operation projection

The committed snapshot has one WorkspaceGeneration. The build, tool invocation, or activation request records the nodes reachable through its declared operation-relevant edge kinds, plus any explicit cognition sources, and a projection digest. The edge-filter policy is versioned and fail-closed; it cannot omit an input the operation actually reads. A change outside an operation's selected inputs does not by itself invalidate its derivative. The TCA projection additionally records selected path filters and content digests per the accepted source-lock contract.

### Step 6 — Publish a resolution receipt

The receipt includes the control commit, manifest/lock digests, accepted base WorkspaceGeneration, effective candidate generation where applicable, operation projection and digest, all selected node commits/trees and declaration digests, local paths, candidate/main provenance, dirty/override status, resolver build identity, target repository, and operation class. Absolute paths belong only in local evidence.

### Step 7 — Generate and validate adapters

Generate deterministic CMake/tool/source adapters under .generated-temp/workspace/<generation>/<projection>/, the approved generated-artifact root. Each adapter embeds the generation and projection identities. A configure-time guard rejects missing/stale adapters and a target whose producer is not the validated node.

### Explicit lock update transaction

Revision movement is a separate proposal: stage all new source commits; read their declarations; validate the complete candidate graph and affected integration gates; create a new lock and generation; compare old/new closures; publish the workspace-control commit only through its governed review. Multi-repository commits cannot land atomically, so the old accepted generation remains runnable until the new lock and required source revisions are reachable and verified. Candidate clean revisions may be validated with provenance without pretending that an unmerged branch is canonical main. Failure retains the old generation and produces a typed receipt; it never falls back to a different checkout.

---

## 4. CMake Integration

### 4.1 CMake becomes a materializer, not a resolver

A consuming CMakeLists becomes structurally similar to:

~~~cmake
if(NOT DEFINED QIVEN_RESOLUTION_FILE)
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

to redirect materialization or select an explicit candidate revision. A path-only relocation of the same locked objects leaves WorkspaceGeneration unchanged. Selecting a different clean commit produces a distinct **CandidateWorkspaceGeneration** computed over the effective complete closure by the same byte algorithm; it never impersonates the committed lock's generation. Dirty worktrees have only a local transient identity, never an authoritative immutable generation. Every adapter and receipt distinguishes accepted base, effective candidate, and source ref.

The resolver labels affected nodes as one of:

~~~text
resolved
candidate-clean
candidate-dirty
~~~

### 5.2 Publication law

- local compile/test MAY use candidate-clean or candidate-dirty overlays;
- a gate claiming exact cross-repository candidate evidence MAY use candidate-clean only under its distinct CandidateWorkspaceGeneration, with commit/tree, branch/ref, control revision, and every selected source recorded; that evidence does not claim canonical-main status;
- publication, acceptance, CI identity, RuntimeGeneration publication, and ActivationGeneration publication reject candidate-dirty dependencies and separately apply their existing candidate-versus-canonical rules;
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

After a shadow comparison and accepted migration, TCA may select a subset of files but takes the revision of each selected repository from the workspace lock. Until then, the accepted CA-1 selector and lock remain authoritative. The activation source projection must contain only selected inputs; a Toolchain, Math, or unrelated third-party movement does not rotate ActivationGeneration merely because the full WorkspaceGeneration changed. The parent workspace digest is provenance in a receipt envelope, not an input to the selected source-lock or ActivationGeneration content digest.

### 9.2 TCA source-lock shape

The activation source lock SHALL contain, for each selected repository and file:

- operation-projection digest (after migration); the parent WorkspaceGeneration is separately recorded in the provenance envelope, outside the source-lock/ActivationGeneration content hash;
- stable repository node ID, repository identity, exact commit and tree;
- selected path filter, concrete selected path, and per-file content digest;
- capability-surface entry ID where applicable.

The actual commit/tree values remain present, not merely inherited by pointer. The lock is independently verifiable and rebuildable from exact local Git objects. Context's canonical bundle/execution RuntimeGeneration and activation sidecar remain separate. Cache keys use the complete selected source-lock content digest and accepted policy/task inputs; a workspace-wide digest alone must not cause unrelated invalidation. Validation still proves that each selected revision belongs to the attested parent WorkspaceGeneration when a migrated receipt is issued.

### 9.3 Capability discovery becomes generation-bound

A capability such as a public Foundation type is meaningful only together with the exact Foundation node in the current generation.

The task bundle can then expose both:

- the semantic capability;
- the exact source generation where it exists.

This is stronger than ambient symbol discovery and cheaper than asking the LLM to reconstruct repository state manually.

---

## 10. Workspace Generation Identity

The digest SHALL be deterministic and free of self-reference. Version 1 defines the byte algorithm before implementation: validate the strict schema, reject duplicate keys and numbers outside the schema's integer vocabulary, encode the manifest and lock *without the generation field* using one specified canonical JSON algorithm (RFC 8785), prefix a domain-separated schema identifier, then SHA-256 those bytes. The lock covers every selected node's stable repository identity, commit/tree, platform/variant/slot, and exact dependency/provision manifest digest. The resolver recomputes and compares the stored generation; independent implementations and golden byte vectors must agree. The workspace-control commit and its manifest/lock blob digests are receipt provenance, not recursive generation inputs.

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

After WorkspaceGeneration is accepted and measured, Qiven MAY add an immutable local store or Git-worktree cache keyed by repository tree/content identity.

That would allow multiple generations to coexist without mutating one shared checkout.

It is intentionally deferred from v1 because the present defect can be removed while preserving today's editable sibling repositories.

WorkspaceGeneration already separates logical identity from physical path, so this evolution remains compatible.
