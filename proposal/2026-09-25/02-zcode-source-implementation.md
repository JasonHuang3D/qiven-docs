# ZCode Source Adapter: Patch Plan and Interface Details

> Status: PROPOSED implementation design, pinned inspection date 2026-09-25 UTC. Code fragments below are deliberately illustrative TypeScript, not a patch claimed to compile on the inspected revision. The source inventory and tests must precede production use.

## 1. Inspected baseline and confirmed seams

At inspected upstream ZCode commit [`29628c9`](https://github.com/zai-org/ZCode/commit/29628c9acdb81b703bbd4080c207a0e7ce5e276e); the exact source files are linked in the table below:

| Source | Confirmed behavior | Implication |
| --- | --- | --- |
| [`packages/adapters/src/model/runner.ts`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/adapters/src/model/runner.ts#L138-L327) | `createModel` closes over `resolved`; `executor.generateText` and `executor.streamText` call `toLegacyRequest` and then the corresponding `*WithResolved` path | Logical invocation preflight can be inserted here for this adapter's calls |
| [`model.ts`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/adapters/src/model/model.ts#L17-L92) | Executor `generateText` returns `Promise<ModelResult>`; `streamText` returns `AsyncIterable<ModelEvent>` synchronously; `prepareRequest` validates/binds model options | The stream wrapper must itself return `AsyncIterable`, e.g. an async generator; no `async (request) => Promise<AsyncIterable>` |
| [`runner-generate.ts`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/adapters/src/model/runner-generate.ts#L91-L196) | Per-attempt request may change messages; `resolveModelForAttempt` refreshes auth; options are built before `runtime.generateText(options)` | A preflight only in `executor` cannot attest to each physical attempt |
| [`runner-stream.ts`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/adapters/src/model/runner-stream.ts#L119-L313) | Per-attempt linked abort, refreshed model, final options and `runtime.streamText(options)` | Gate each physical stream attempt before any event escapes |
| [`runner-options.ts`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/adapters/src/model/runner-options.ts#L36-L155) | Converts messages/tools into AI SDK representation; handles provider options, output shape and attribution headers | Compare actual projected messages/tools at the final send seam, not just pre-projection text |
| [`runner-runtime-headers.ts`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/adapters/src/model/runner-runtime-headers.ts) | Auth refresh can occur on every attempt | Keep auth resolution in ZCode and omit credentials from Qiven IPC |
| [`runner-runtime.ts`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/adapters/src/model/runner-runtime.ts) | AI SDK runtime methods have different return contracts (`generateText` promise; `streamText` immediate result) | Put awaited authorization in the runner before calling either method; do not make runtime's streaming method async |

These are confirmed **model-adapter** paths, not yet a proof that every model invocation or tool action uses this adapter. No verified tool-dispatch source location is claimed here. Finding and qualifying it is an implementation entry gate. The intended `BeforeModelInvocation` call is mandatory for every logical invocation in the qualified Desktop Agent, even when Qiven revalidates an existing bundle or admits a narrowly named observed-only class. The per-attempt guard remains separate; neither a first-task-only hook nor a locally cached receipt may stand in for later logical calls.

### 1.1 Native seams and live observations at the pinned revision (2026-09-25 source review)

A full source review of the pinned commit plus direct observation of the owner machine's installed build found five facts that change the implementation plan:

1. **A native `ModelRequestAdmission` port already exists and is factory-bound** ([contracts](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/contracts/src/model/index.ts#L57-L90), [adapters](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/adapters/src/model/request-admission.ts)). Both runners call `admitAttempt` before **every physical attempt** and release on completion; the port is bound once at the model-handle factory, and the upstream comments record that this binding was their own fix for exactly the auxiliary-call-site bypass gap: every model handle a runtime hands out — turn steps, tool-internal calls, compaction, the title sidecar — carries it, with no per-call exit. Subagent child runtimes **inherit** the parent's port by construction ([subagent.ts](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/core/src/runtime/methods/subagent.ts)). Its input today is only provider/model identity and its only implementation is an observe-only concurrency governor. Crucially, [generate](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/adapters/src/model/runner-generate.ts#L123-L196) and [stream](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/adapters/src/model/runner-stream.ts#L238-L310) both acquire this ticket **before** model/auth refresh and construction of final options. It is the native capacity/status lifecycle, not a final-content gate. For the qualified profile, extend its ticket with a mandatory, awaited Qiven final-permit phase after options are built; retain the existing acquire/release and status plumbing. Do not add the final projection digest to an earlier `acquire` input that cannot yet know it, or create a second capacity queue.
2. **The census is larger than the file list above.** Confirmed additional model call sites at the pinned revision: `target-completion-verification.ts` (goal-state-affecting — not observed-only-eligible without proof), `workspace-generate-text.ts` (stream and generate), `memory-agent-loop.ts` (driven by `project-memory-agent.ts`), `read-session-context.ts` (a tool that runs a model over session context), and `webfetch-processing.ts` (WebFetch processes pages with a model call). All flow through runtime model handles into the adapter; a repository-wide search found **no AI SDK imports outside the adapters package**, so the executor seam is a strong provider-facing candidate; source-import search alone does not establish complete runtime coverage of dynamic clients or alternate process paths.
3. **`PreToolUse` input replacement is implemented at the pinned revision**: hook output parsing accepts `updatedInput` ([hooks/output.ts](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/core/src/hooks/output.ts#L125-L127)) and the tool executor substitutes it before execution ([call-runner.ts](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/core/src/tool/executor/call-runner.ts#L269-L272)); `PermissionRequest` hooks can also modify input. This resolves the README-versus-ADR-0051-probe discrepancy source-side: the capability exists at `29628c9`, so the earlier live probe must be re-run against the deployed 3.14.3 build with a schema-conforming hook output (probable causes of the earlier negative: an older deployed build, or an output shape the parser silently dropped).
4. **Subagents are in-process child runtimes**, not subprocesses: the Agent tool constructs `new AgentRuntime(...)` inside the agent process with `subagentContext` (agent prompt and user instructions; Project Context deliberately not inherited). AsyncLocalStorage propagation therefore reaches subagent model calls without a process boundary; the separately authenticated cross-process envelope is required only for the Desktop-Host↔Agent boundary and plugin-host children.
5. **The installed build is now observed** (previously the named residual): the owner machine's installed Desktop is version 3.14.3.7762 — equal to upstream tag `v3.14.3`, which is the pinned commit `29628c9` (version-level identity). Live process capture confirms the spawn chain exactly: Electron main plus a `zcode.cjs app-server --stdio --surface desktop` agent per session and an agent-child `__zcode-plugin-host` process serving MCP plugins; the installed `resources/glm/zcode.cjs` SHA-256 is `b1df2ef3e5bd76c4af3ecb296bc003a10d3f13191a26610bd0ba940feadad529`, and `resources/glm/packages` carries the official plugin set. This is a static observation of one machine's installed state, not a governed-trial observation of a patched build.

Two risk-register facts from the same review: the Desktop update feed ships as an inert generic placeholder pointing at localhost ([electron-builder.config.js](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/packages/desktop/electron-builder.config.js#L759-L768); the installed `app-update.yml` matches), so official 3.14.3 builds do not auto-update from a remote feed — record the feed as a preflight input anyway, since future upstream releases or self-built configs can change it; and official Coding Plan requests are routed through the ZCode platform gateway by pure URL rewrite with account credentials passed through and entitlement validated platform-side ([official-coding-plan-gateway.ts](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/adapters/src/model/official-coding-plan-gateway.ts)) — no client-signature gate is visible on that path, so provider acceptance of a rebuilt isolated-profile client is a policy (terms-of-service) question for the owner, not a mechanical block, and stays undecided here.

## 2. Repository changes by responsibility

| Repository | Candidate delta | Guardrail |
| --- | --- | --- |
| `qiven-context` | Owner-accepted ADR/program amendment after deliberation; if admitted, update control policy/coverage inventory and obligation | Do not edit an accepted ADR's historical decision or mark this proposal accepted by a docs PR |
| `qiven-runtime` | Native `BeforeModelInvocation`/attempt-decision operations over CA-1 core, a capability-scoped connector identity and durable events | Existing Host IPC is not a drop-in Agent privilege: server-side per-client verb checks must reject owner `Shutdown`/`Mutation` from this identity; no broad owner secret in Desktop JS |
| `qiven-devkit` | Connector contract conformance, generation/coverage verification, task/role schema mapping, independent negative trials and publication gate bindings | Never treat a bundle issuance as proof of delivery or a tool status as proof of control |
| ZCode source overlay | Small version-checked patch set: all-logical-invocation adapter call, physical attempt guard at both runner call sites, final tool dispatch guard, provider-side effect census, build ID and observation writer | Rebase reproducibly on the pinned upstream; preserve provider binding, option validation, auth, cancellation, telemetry, background execution and release packaging; never turn a Host allow into a raw ZCode mutation |
| `qiven-docs` | This proposal and signed cross-LLM/owner deliberation | Merge and accepted migration only under owner acceptance |

### 2.1 Proposed local TypeScript contract

```ts
// qiven-harness-adapter.ts -- proposed names and types, not upstream API.
type TaskScope =
  | Readonly<{
      kind: "bound";
      taskId: string;
      phaseEpoch: string;
      role: "jason-worker" | "jason-brother" | "jason-extended-cognition";
      observedBoundaryId: string;
      delegatedBriefSha256?: string;
    }>
  | Readonly<{ kind: "unbound"; reason: "pre_task" | "background" | "system" }>;

type InputBoundary =
  | Readonly<{ kind: "ordinary-task"; policyId: string; allowedSourceSetSha256: string }>
  | Readonly<{
      kind: "clean-worker-review";
      policyId: string;
      allowedSourceSetSha256: string;
      sealedBriefSha256: string;
      reviewArtifactHeads: readonly string[];
    }>
  | Readonly<{
      kind: "sealed-trial";
      policyId: string;
      allowedSourceSetSha256: string;
      assignmentId: string;
    }>
  | Readonly<{
      kind: "freshness-canary";
      policyId: string;
      allowedSourceSetSha256: string;
      canaryNonceSha256: string;
    }>;

type InvocationKey = Readonly<{
  sessionId: string;
  callId: string;
  actorKind: "main" | "subagent" | "workflow_child" | "system" | "tool";
  scope: TaskScope;
  inputBoundary: InputBoundary; // established by trusted ingress/delegation, never model text
}>;

type BoundInvocationKey = InvocationKey & Readonly<{
  scope: Extract<TaskScope, { kind: "bound" }>;
}>;

type PreparedInvocation = Readonly<{
  key: BoundInvocationKey;
  receiptId: string;
  bundleSha256: string;
  activationGeneration: string;
  sourceLockSha256: string;
  policySha256: string;
  consumerProfile: string;
  rendererBuild: string;
  rendererSha256: string;
  budgetBinding: string;
  liveEvidenceBinding: string;
  inputBoundarySha256: string;
  insertedSegmentId: string;
  expiresAt?: string;
}>;

type ModelInvocationDecision =
  | { kind: "governed"; prepared: PreparedInvocation; renderedSegment?: string }
  | { kind: "observed-only"; key: InvocationKey; policyId: string; scopeDigest: string }
  | { kind: "deny"; key: InvocationKey; reason: string };

interface QivenMediationPort {
  beforeModelInvocation(input: {
    invocation: InvocationKey;
    task: ObservedAndClaimedTask | UnboundTaskEvidence;
    callClass: ModelCallClass;
    providerId: string;
    modelId: string;
    projectedRequest: PublicRequestProjection;
    abortSignal?: AbortSignal;
  }): Promise<ModelInvocationDecision>;
  authorizePhysicalAttempt(input: {
    decision: Exclude<ModelInvocationDecision, { kind: "deny" }>;
    attempt: number;
    finalProjectionDigest: string;
    insertedSegmentProof?: SegmentProof;
    modelIdentity: { providerId: string; modelId: string };
    abortSignal?: AbortSignal;
  }): Promise<AttemptPermit>;
  decideAction(input: ObservedAction): Promise<ActionDecision>;
  recordDispatch(input: AttemptDispatchObservation): Promise<void>;
}
```

`PublicRequestProjection` excludes `accountAccess`, headers, bearer tokens, auth callbacks and sensitive debug bodies. The adapter compares actual projected segment bytes locally and sends bounded proofs/digests; full prompts are captured only in the separately controlled trial channel. The connector must be versioned, authenticated and bounded. Current Runtime Host IPC has an owner-only DACL and installed-image allowlist with a same-user DPAPI/HMAC secret, but no caller-specific verb authorization; any authenticated client can request `Shutdown`. Do not add the Electron Agent to that broad client set or expose its reusable owner secret to JS. Introduce a narrow native connector or separate endpoint and enforce per-client/per-operation rights at the Host, with negative `Shutdown` and `Mutation` probes. The existing one-shot `qiven-adapter-bridge` is hook-mapping state tooling, not this Host RPC. Per-verb checks prevent a broad Agent credential but cannot isolate hostile code already running as the same Windows user with access to the DPAPI secret or allowed owner executables. Record that trust limit; stronger hostile-process claims require OS identity isolation and separate credentials. Exact C++ operation names and transport schema must be reconciled against CA-1 implementation before coding; no invented `qiven cognition` subcommand is represented as currently shipped.

### 2.1.1 Origin and call-class propagation

The inspected [ZCode `ModelInvocationContext`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/contracts/src/model/invocation-context.ts#L15-L21) uses Node `AsyncLocalStorage`; its [`runWithModelInvocationContext`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/contracts/src/model/invocation-context.ts#L72-L102) also re-enters that context on an async iterator's `next/return/throw`. [`runtime-model.ts`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/core/src/runtime/methods/runtime-model.ts#L44-L92) wraps model handles and merges call context, and [`runner.ts`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/adapters/src/model/runner.ts#L138-L194) reads it in `toLegacyRequest`. These are usable **in-process propagation mechanics**, not existing Qiven identity or authorization. Today's [`ModelApiCallObservation`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/contracts/src/telemetry/index.ts#L114-L146) and `querySource` fallback name telemetry operations/actors; they do not bind the observed task ingress, phase epoch or Jason role. In particular, `actorKind: "subagent"` is not proof of which Jason role was delegated, while a title, compaction or tool-internal model call may have no bound Jason task at all.

Extend the core-owned invocation context with a private, typed `qivenOrigin` record containing `sessionId`, a generated per-logical-call ID, trusted call-site/class ID, actor kind, and a `TaskScope` matching the union above. For `bound`, obtain task and phase from the observed ingress/Runtime task ledger, role and delegator/brief binding from the actual main or subagent creation path; for `unbound`, record why no task exists. This field is set by vetted core call sites, never copied from `ModelRequest`, `metadata`, model output, `agentName`, or an arbitrary `querySource`. Propagate it via `withModelInvocationContext` into the adapter, but strip it into a private side channel at `toLegacyRequest`: the current implementation spreads the remaining invocation context into the legacy request, so simply adding `qivenOrigin` to the type would leak a control field into provider option construction, telemetry or debug paths. Keep model-facing `AiSdkModelTextRequest` and public diagnostics free of this field and never let a model-handle factory snapshot mutable task state at creation. Verify the context survives lazy stream consumption; capture the origin at call time and recheck phase/source freshness at first iteration and every send. `AsyncLocalStorage` cannot cross the Desktop Host-to-Agent or any plugin-host **process** boundary; carry an authenticated, scoped ingress envelope where those paths are in scope and validate it against Runtime facts on the receiving side. At this pinned revision, subagents are in-process child runtimes: the async store can propagate to them, but create a fresh, independently checked child origin from the actual delegation brief, child session and role rather than inheriting the parent's task/role as authority. The trusted connector identity proves which build/process is asking, not that a supplied task/role field is true.

The input boundary is not inferred from the role string. A fresh-position worker review receives a sealed, scoped input policy established by the orchestrator at delegation: brief, named artifact heads and applicable engineering law only. The [v29 clean-input contract](https://github.com/JasonHuang3D/qiven-context/blob/5d6653db54d1d41c913ea6a0729524a8b7b235de/views/workflows/subagent-delegation.md#L205-L242) excludes session/checkpoint narratives, workflow logs and process-related state/memory/obligation records, including loop-state commit narration. An ordinary author's TCA bundle (which may contain state and obligations) cannot be copied into this reviewer merely because a digest matches. Resolve the admitted source set in the Runtime selector, bind its policy and digest to the full cache key, and inspect generated renderer bytes before insertion. If an applicable protected engineering invariant exists only in a forbidden process record, block the review and raise the conflict for owner adjudication rather than drop the rule or widen the corpus silently. An ADR-0050 §14.1 independent falsification reviewer, a Profile C consumer and an R3/CA-5 reviewer use their separately sealed inputs; they do not inherit the clean-worker policy by role name alone. A freshness canary runs with the **same loaded adapter and policy configuration** as the imminent worker, using a declared no-project-cognition input boundary; it produces a triage result, not governed design evidence. Both positive required-rule visibility and negative forbidden-source absence must be measured in the final request. The adapter cannot itself stop a worker from reading Git history or local files; retain the workflow's ACK disclosure unless a separately qualified read boundary is built.

Reconcile the two channels that compose a worker's model-visible cognition: the brief's scoped engineering-law reading list ([ADR-0053 §5a](https://github.com/JasonHuang3D/qiven-context/blob/5d6653db54d1d41c913ea6a0729524a8b7b235de/decisions/ADR-0053.md)) names sources the worker may read itself, while mediation injects protected segments into its requests. Resolve both against the admitted source set so a rule arrives once: digest-bind any brief excerpt of a protected rule to its pinned source, deduplicate injected segments against brief-carried text by stable segment ID or source digest, and treat an excerpt-versus-segment wording mismatch as a typed block rather than a double delivery.

The initial call-site census must include at least [main `model.ts` calls](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/core/src/runtime/methods/model.ts#L80-L143), [compaction stream-to-generate fallback](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/core/src/runtime/methods/compact-summary-model-request.ts#L285-L330), [title generation](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/core/src/runtime/methods/title-generation-sidecar.ts#L130-L156), [project-memory agent](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/core/src/runtime/helpers/project-memory-agent.ts#L48-L77), and [tool-internal WebSearch](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/core/src/tool/handlers/websearch.ts). The compaction helper [copies a fixed set of fields into a new async context](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/core/src/runtime/methods/compact-summary-model-request.ts#L24-L36) [for both stream and generate](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/core/src/runtime/methods/compact-summary-model-request.ts#L285-L331); simply adding a field to an outer context would lose it. Extend the request type and explicit copy with `qivenOrigin`, and assign a distinct Qiven logical call ID to both the initial compaction stream and its non-stream fallback (which [already receives a distinct telemetry ID](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/apps/zcode-cli/packages/core/src/runtime/methods/compact-summary-model-request.ts#L74-L86)). Likewise, a tool-internal call that creates a new context must propagate its vetted tool/task origin explicitly. Each replacement call needs its **own** Qiven decision and current input check; carrying the prior context or delivery record across the replacement fails. Validate ADR-0053 actor/role combinations at the runtime boundary, including a brother subagent's full boot and the extended-cognition main-only rule. Add remaining callers, alternate SDK paths and actual subagent spawn routes to the executable census rather than extrapolating from these examples. An absent/contradictory origin is `UnqualifiedOrigin`: still record the encountered call, but deny a governed dispatch. An unbound call is not automatically exempt: Qiven may admit a named observed-only class only after proving its output cannot influence governed design/action state. Compaction summaries and tool outputs that feed later prompts fail that test unless separately contained.

### 2.2 Logical seam in `runner.ts`

The user's sketch is directionally correct for this seam. The following shape handles both executor return types and avoids passing sensitive `resolved` to Qiven. `beforeModelInvocation` runs on **every** logical call, including a later call that reuses a bundle; a denial cannot enter either runner:

```ts
executor: {
  generateText: async (request) => {
    const invocation = captureHarnessInvocation(); // reads private qivenOrigin or blocks
    const legacy = toLegacyRequest(request); // strips origin from provider fields
    const decision = await qiven.beforeModelInvocation({
      invocation,
      task: observeTaskEnvelope(invocation),
      callClass: classifyModelCall(invocation),
      providerId: resolved.providerId,
      modelId: resolved.modelId,
      projectedRequest: projectPublicRequest(legacy),
      abortSignal: request.abortSignal,
    });
    if (decision.kind === "deny") throw new QivenMediationError(decision.reason);
    const mediated = applyQualifiedDecision(legacy, decision);
    // Only a typed message addition is allowed; model/options/tools/auth are unchanged.
    return this.generateTextWithResolved(
      mediated, resolved, resolveForRequest(mediated, request.options), decision,
    );
  },
  streamText: (request) => {
    const invocation = captureHarnessInvocation(); // capture before lazy iteration
    const legacy = toLegacyRequest(request); // strips origin from provider fields
    const task = observeTaskEnvelope(invocation);
    const callClass = classifyModelCall(invocation);
    return (async function* () {
      const decision = await qiven.beforeModelInvocation({
        invocation,
        task,
        callClass,
        providerId: resolved.providerId,
        modelId: resolved.modelId,
        projectedRequest: projectPublicRequest(legacy),
        abortSignal: request.abortSignal,
      });
      if (decision.kind === "deny") throw new QivenMediationError(decision.reason);
      const mediated = applyQualifiedDecision(legacy, decision);
      yield* runPreparedStream(
        mediated, resolved, resolveForRequest(mediated, request.options), decision,
      );
    })();
  },
}
```

In real code, preserve `this` by using a bound/local `runPreparedStream`, and use the repo's concrete message types. A lazily iterated stream captures its invocation/role context at call time, **before** the caller consumes it; the generator revalidates phase and source freshness at execution time. `applyQualifiedDecision` preserves the exact authorized protected segment if already present in projected history, otherwise inserts it through a typed helper with explicit priority/ordering and idempotence. It must do so for every design-capable call, including a later call after compaction or a delegated subagent with fresh input. A reused activation means the complete TCA §17.3 cache binding was revalidated; it never permits receipt-only input or reuse of an earlier delivery event. A shorter segment requires a new Runtime-approved policy/renderer/budget binding and receipt retaining all applicable protected items. An observed-only class is admitted only under a named policy proving that call cannot influence governed design/action state. A new `messages` array is derived without mutating canonical conversation history. The returned object passes request validation rules; model options are bound from the original `request.options`. If mediation changes anything other than an allowed message addition, reject it with a typed protocol error. The `callClass` and task scope are taken from the core-owned `qivenOrigin` established in §2.1.1, then reconciled by Qiven against task/role and call-site evidence. Do not silently fall back from a missing origin to telemetry `modelCall` or `querySource`; `UnqualifiedOrigin` is a typed block for governed calls. A model-supplied label cannot create an exemption.

### 2.3 Physical attempt seam in both runners

Both runners already call `admitAttempt` with the factory-bound `modelRequestAdmission` port (or a synchronous `tryAcquire` fast path) **before** `resolveModelForAttempt` and `create*Options`; they keep its ticket until a terminal event or `finally`. This port controls capacity/attempt status. At acquisition time it has only provider/model identity, so extending its `acquire` input with `finalProjectionDigest` would bind a value that does not exist yet. Preserve the one native capacity ticket and require a **two-phase ticket** for every call in the qualified profile: first acquire/queue as upstream does; then, once refreshed model/auth and final options exist, invoke an awaited `ticket.authorizeFinalAttempt` that delegates to `QivenMediationPort.authorizePhysicalAttempt` for this invocation and attempt. Place that second phase before `model_request_started` and the SDK call, with the same attempt ID and linked abort semantics. There is no second capacity queue. Require the guarded ticket even when `tryAcquire` succeeds; the upstream no-port path returns a silent no-op, so absence of the port or final-permit method must be `UnqualifiedAdapter` and block every classified send in a claimed qualified session (including observed-only), not silently admit it. A Qiven denial is a nonretryable local failure, never attributed to provider connectivity. Release the acquired ticket in the existing `finally` on authorization failure, preserving upstream queue, cancel and status behavior.

Pass the per-call `decision` to `runGenerateText` and `runStreamText` (or an equivalent context object). After `createGenerateTextOptions` / `createStreamTextOptions`, and before `input.runtime.generateText(options)` / `.streamText(options)`, do the following for **every attempt**:

```ts
// Check exact protected bytes and priority in this attempt's final projection,
// even when the segment was already present in projected history.
const proof = verifyLocalSegmentAndHash(options.messages, options.tools, decision);
const permit = await requireGuardedTicket(admission, decision).authorizeFinalAttempt({
  decision,
  attempt,
  modelIdentity: { providerId: resolved.providerId, modelId: resolved.modelId },
  finalProjectionDigest: proof.digest,
  insertedSegmentProof: proof.segment,
  abortSignal: attemptRequest.abortSignal,
});
if (!permit.matchesFinalProjection) throw new QivenMediationError("DeliveryMismatch");
// The guarded ticket delegates to qiven.authorizePhysicalAttempt for this same attempt.
// Publish authorization and placement verification here, not delivery.
// Only then invoke the existing input.runtime.generateText(options) or
// input.runtime.streamText(options). Do not await runtime.streamText itself.
```

`model_request_started` currently appears after options construction and before send. Make its semantics precise: authorization is not network acceptance; emit a separate authorized/placement event or move the started record after successful runtime call setup and preserve the existing accounting contract. Even successful runtime call setup is not independently proved provider acceptance: a separately qualified transport observation or real captured trial is needed before issuing `InvocationDeliveryEvent`. Route a Qiven pre-dispatch failure to a **nonretryable local failure class** that cannot be mistaken for provider throttling, stream idle timeouts, or a retryable auth error. Do not let existing generic catch/retry paths silently make an unauthorized next attempt. Keep request ID fresh for a new physical attempt; refresh auth once per existing ZCode rules. Signature-repair retry may legitimately alter earlier reasoning messages, so compare the registered injected segment in the final projection rather than requiring the whole request byte hash to equal the initial logical request. Record the whole final projection digest for correlation. Census the final provider options for hosted tools or server-side effects: if a provider could perform an in-scope effect upon receiving this request, the local post-output and tool-dispatch gates are too late. Disable that capability in the qualified profile or require a separately accepted pre-send mediation path; prove the option/serialized body used for the actual attempt.

`projectRequestHistory` occurs in `*WithResolved` before the runners; `toAiSdkMessages` in `runner-options.ts` can change the provider-facing shape. The ticket's final-permit phase therefore verifies **presence, position and equality of the injected segment after both transformations**, or a valid named observed-only disposition, not merely the preflight output. For a clean-input or sealed-trial profile, the same final-request boundary must also reject any forbidden source content or provenance carried by existing history, automatic context, hook text or tool results; matching the inserted protected segment is insufficient. Bind the admitted-source-set digest and policy revision to the permit, and preserve sealed full-input capture in the controlled trial channel for the absence claim. If the adapter cannot establish that negative claim, the review remains unqualified. A model provider or AI SDK may further serialize messages; capture a mock HTTP body and at least one live model-visible task trial for the qualified profile. If final serialization changes the segment, move the check to the actual send point and requalify; do not issue a false delivery event. For a claimed governed design path, the attempt's qualified delivery observation must precede release of any design output, including streamed output. Bound any buffering while the transport observation is pending; missing or late observation blocks/cancels the governed attempt, and a later response cannot backdate delivery. Only an explicitly observational shadow run may retain unqualified output outside acceptance.

Implement that release gate at the per-attempt runner boundary across **all** model-origin output paths. In the pinned `runner-stream.ts`, the ordinary visible-event loop, direct tool-call commit on a validation error, retry-safe pending events and the tool-call assembler flush can each `yield` separately. Buffer each path's model-derived events, including tool calls and completion/error events that can change downstream design or action state; publish visible milestones and mark output committed only after qualified delivery for that same attempt. An unverified timeout cancels and cleans up the iterator, discards buffered output and returns a typed local `DeliveryUnverified` result. A retry gets a new attempt and cannot inherit the previous attempt's delivery. In `runner-generate.ts`, the provider promise may resolve before the delivery observer: hold the normalized text, reasoning and tool calls, success status and return until the matched observation arrives. The observer must bind the attempt ID, final serialized request digest, protected-segment/priority proof and actual transport event; response headers or a first chunk by themselves do not prove exact input. Bound the gate's memory and wait time, and treat an SDK path without the required observer as unqualified.

### 2.4 Tool action seam, to be located by census

Instrument the actual dispatcher that has the final structured tool name, arguments, execution target and permission result. It must run before the side effect and cover main/subagent tools. Model output is a proposal; classify trusted mechanism facts separately from claimed intent. The following is illustrative, not a shipped Host API:

```ts
const proposed = observeToolRequest(finalToolName, validatedArgs, target, invocationRef);
const qivenDecision = await qiven.decideAction(proposed);
if (!qivenDecision.allow) return typedDenial("CognitionDenied");
const permission = await resolveZCodePermission(proposed);
if (!permission.allow) return typedDenial("PermissionDenied");
const finalAction = observeFinalDispatchArguments();
if (!stillBound(qivenDecision, finalAction, currentCognitionRevision(),
                acceptedDeploymentProfileRevision())) {
  return typedDenial("StaleActionDecision");
}
if (finalAction.kind === "governed-local-mutation") {
  // Host validates Qiven's server-side decision, its current lease/fence,
  // and the exact final action; it executes inside its broker.
  return hostBroker.executeWithCurrentAdmission(finalAction, qivenDecision);
}
if (finalAction.kind === "qualified-read-only") {
  return executeWithOutcomeObservation(finalAction, existingExecutor);
}
return typedDenial("UnqualifiedActionPath");
```

ADR-0038 §§8-9/14 require Qiven allow to bind the exact final action, pinned cognition/policy, satisfying evidence and accepted DeploymentProfile revision. A changed argument, target, policy, evidence or profile invalidates that allow; the Host independently rechecks lease/fencing when it executes. For a governed JasonPC mutation, Host approval followed by `existingExecutor` is a bypass: the Host broker must verify the Qiven decision using trusted server-side state (not an Agent-supplied Boolean) and own execution, journaling and indeterminate-outcome reconciliation. If permission handling changes the final action, both Qiven and harness permission must be reacquired for that new proposal. Before MVP-5 the current Host denies governed raw writes, so this branch remains deny-only. External consequential effects need a separately qualified authority route; they do not silently fall through the read-only branch. A hook denial or `PostToolUse` context cannot replace the complete action guard without profile qualification. Preserve `run_in_background`, completion notification, `TaskStop` tree termination, oversized-output persistence and ADR-0048 custody. Ensure public tool boundaries, MCP delegation, shell and filesystem writes have distinct census rows; if an auxiliary executor bypasses the central dispatcher, instrument it or exclude its operation class honestly. Provider-hosted effects occur at model send rather than local dispatch, so account for them in the physical-attempt census even if this source revision has none enabled.

Where a workspace hook router already mediates the same tool boundary (the ADR-0051 law carrier in qiven-devkit), the effective action decision is the **conjunction** of adapter decision, router decision and harness permission, and each denial names the boundary that produced it with its own typed reason and provenance tag. An ambiguous denial that does not say which gate spoke is a defect under the owner's [no-ambiguous-denials rule](https://github.com/JasonHuang3D/qiven-context/blob/5d6653db54d1d41c913ea6a0729524a8b7b235de/collaboration/long-command-registry.md); the adapter must not swallow, re-label or coalesce router denials into its own, and a router-mandated re-call form (for example the background re-call instruction) must reach the model as the router phrased it.

## 3. Windows Desktop build, startup and reproducibility

The owner's primary surface is **ZCode Windows Desktop**, not standalone ZCode CLI. The inspected upstream source has a concrete Desktop chain:

| Stage | Confirmed source and behavior | Verification target |
| --- | --- | --- |
| Build | [`scripts/build-desktop-agent-cli.mjs`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/scripts/build-desktop-agent-cli.mjs#L83-L99) builds the Agent and stages it for local Desktop dev; [`prepare-agent-node-bundle.mjs`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/packages/desktop/scripts/prepare-agent-node-bundle.mjs#L231-L295) performs the packaged staging | Changed adapter source is present in generated `apps/zcode-cli/packages/cli/dist/zcode.cjs` |
| Stage | [`stage-agent-bundle.mjs`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/packages/desktop/scripts/stage-agent-bundle.mjs) copies that file to `packages/desktop/bundled-agents/win32-<arch>/glm/zcode.cjs` and writes `.node-bundle-meta.json` | Source, staged bytes and meta match; old staged content is removed |
| Package | [`electron-builder.config.js`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/packages/desktop/electron-builder.config.js#L625-L633) maps the platform's staged `glm` directory to the app's `resources/glm` | Built Windows package contains the expected `resources/glm/zcode.cjs` digest |
| Resolve | [`providerRuntimeResolver.ts`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/packages/services/src/runtime-tools/providerRuntimeResolver.ts) looks for a packaged `resources/glm/zcode.cjs` using `process.resourcesPath`; its JS-bundle resolver is distinct from the native binary resolver and does not read `GLM_BINARY_PATH` | Actual loaded JS file path, not a CLI on `PATH` or a native `.exe` |
| Spawn | [`zcodeAgentProcessManager.ts`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/packages/services/src/zcode-agent/zcodeAgentProcessManager.ts#L353-L463) starts the Agent with the app's Electron Node runtime, `ELECTRON_RUN_AS_NODE=1`, and `zcode.cjs app-server --stdio` | Host-to-Agent command/args, loaded digest, session and invocation events |

Command precedence matters. In the inspected resolver an explicit `ZCODE_AGENT_SERVER_COMMAND` overrides the default; a monorepo development entry may win next; then the packaged Electron JS bundle; only then a native binary fallback. Desktop's `GLM_BINARY_PATH` native-binary handling is **not** a reliable override for the packaged JS Agent. Also distinguish development bytecode mode (`ZCODE_DESKTOP_AGENT_BYTECODE=1`) from the normal JS bundle. Therefore a successful `node apps/.../zcode.cjs --help`, a changed global CLI, or a copied file next to the source tree proves nothing about an already running installed Desktop session. The source establishes the packaging route; the installed build and runtime path on the owner machine were **statically observed in the 2026-09-25 source review** (version 3.14.3.7762 matching the pinned tag, the spawn command and process tree matching this table, agent bundle digest recorded in §1.1), while a governed trial's running-adapter observation remains future evidence.

Upstream [ZCode's build guide](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/README.en.md) specifies Node.js 24.14.0 and pnpm 10.33.2 from `mise.toml`. The normal build route for this profile is to compile the adapter and dependent Agent packages, stage the Windows Agent, then use `pnpm bundle:desktop -- --os win --arch x64` (or `--arch arm64` for the actual target). The Desktop [`bundle.mjs`](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/packages/desktop/scripts/bundle.mjs#L704-L753) prepares runtime assets, builds Desktop, invokes electron-builder, and verifies the package unless skip flags are explicitly selected; **do not use `--skip-prepare` or `--skip-build`** for the first qualification. `pnpm build:zcode` is a separate CLI/TUI/Web distribution and is not the Desktop deliverable.

### 3.1 Desktop qualification sequence

1. Pin ZCode source, fork diff, dependency lock, target architecture, Desktop build metadata, model-adapter and tool-dispatch tests. Build with the supported Windows target and retain the last known good packaged build. Keep a separate test application identity/data profile so the official installed Desktop session and existing account credentials are not silently reused or overwritten.
2. Record SHA-256 for `cli/dist/zcode.cjs`, staged `bundled-agents/win32-<arch>/glm/zcode.cjs`, unpacked/installed `resources/glm/zcode.cjs`, and package artifact. Verify content equality at the relevant stage; a version string or `.node-bundle-meta.json` alone is not a payload hash. Validate the installed Electron main/Host and Agent version pairing and any package/update metadata that the inspected Windows build actually uses; do not assert an unverified external signature or server entitlement behavior.
3. Start the isolated **packaged Windows Desktop application**, open a local project task, and capture the Agent's effective command, process ancestry, resolved path and loaded artifact digest. A staged bundle, running CLI command, or `GLM_BINARY_PATH` setting cannot substitute for this observation. Restart the Desktop Agent after changes; ensure an already running app-server is not continuing on the old bytes.
4. Run Desktop-originated model invocations (normal and stream), actual user prompt ingestion, a delegated worker, one denied tool operation and the retry/abort trials. Preserve the existing Desktop browser OAuth return path and ZCode provider authentication/refresh in the fork; test login and model use in the **isolated** Desktop profile only if required for the controlled trial. This plan does not claim a modified Desktop build is accepted by the provider or shares the official client's account entitlements until verified in that build.
5. Record `ZCODE_AGENT_SERVER_COMMAND` overrides, dev-tree detection and bytecode mode as explicit preflight inputs. For SSH/WSL/remote sessions, the inspected resolver can use a deployed native Agent rather than local `resources/glm/zcode.cjs`; mark those sessions outside this first profile until separately built and qualified. The same applies to CLI, TUI and Web paths.

The initial configuration is opt in and scope declared. Startup attestation binds the installed Desktop Agent, adapter protocol, authenticated Runtime Host availability and source generation. A rollback selects the previously verified whole Desktop build while preserving user data, and records any trial under an old or unknown Agent as unqualified. Do not treat a raw installed-file swap as the deployment plan; use the reproducible staged build and package, then observe the running process.

No source overlay is adopted merely because it compiles. The Windows Desktop profile and the final provider/tool boundary inventory determine whether the installed Agent actually carries the required control. A ZCode upstream upgrade must re-run the Desktop launch census and negative trials before restoring the corresponding claim.

## 4. Suggested change order and reviewable PR slices

1. **Inventory PR:** source and runtime call graph; alternate model clients; all main/subagent/tool entrypoints; assertions of observed versus inferred facts; hook documentation/probe discrepancy; explicit uncovered list. Prove how every logical invocation reaches `BeforeModelInvocation`, including non-design calls and a `jason-brother` subagent. Map fresh worker review, canary and independent sealed-trial ingress to distinct input policies; enumerate automatic context carriers and uncovered reads rather than assuming an injected bundle preserves reviewer independence. Consume the owner-ordered subagent live boundary experiments ([OBL-20260925T090500Z-F1A2B3](https://github.com/JasonHuang3D/qiven-context/blob/5d6653db54d1d41c913ea6a0729524a8b7b235de/obligations/OBL-20260925T090500Z-F1A2B3.md)) as primary census inputs: the transcript-provenance harvest instruments real subagent conversations, the router-denial boundary probe doubles as the hook `PreToolUse` behavior reconciliation trial, and the clean-input A/B measures reviewer input-policy drift on live sessions.
2. **Protocol PR:** Runtime adapter service and ZCode connector interfaces with authentication, Host-enforced per-client verb rights, deadlines, typed errors, a recorder and deterministic receipt/renderer validation. Prove that an Agent-facing identity cannot request Host `Shutdown` or `Mutation`. No execution-path switch yet.
3. **Model PR:** logical insertion and physical attempt gates in both runners, full-binding reuse and post-compaction/subagent input tests, stream/type safety, delivery-before-output, retry/cancel/auth tests, mock-wire capture and startup build ID. Keep same options and telemetry behavior when the feature is disabled.
4. **Action PR:** proven tool dispatcher interception, final-action/profile binding and pre-MVP-5 deny-only Host/Qiven parity; provider-hosted effect inventory, background/custody and outcome reconciliation. Affirmative JasonPC local mutation will require Host broker execution, not a direct ZCode executor. Positive execution of governed mutations belongs to the later MVP-5 control path and is not a CA-2 entry gate.
5. **Trial PR/record:** enable only the declared profile, run the real controlled CA-2 task and independent negative tests, publish coverage and provenance. Promote only after owner-governed acceptance.

This is a suggested subdivision for the **separate harness lane**. It does not add these PRs to the already declared CA-1 batch budget or authorize bypassing the CA-1 stop rule.
