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

These are confirmed **model-adapter** paths, not yet a proof that every model invocation or tool action uses this adapter. No verified tool-dispatch source location is claimed here. Finding and qualifying it is an implementation entry gate.

## 2. Repository changes by responsibility

| Repository | Candidate delta | Guardrail |
| --- | --- | --- |
| `qiven-context` | Owner-accepted ADR/program amendment after deliberation; if admitted, update control policy/coverage inventory and obligation | Do not edit an accepted ADR's historical decision or mark this proposal accepted by a docs PR |
| `qiven-runtime` | Native, authenticated local adapter-facing prepare/verify/decide operations over CA-1 core; durable receipt and evidence events | Deterministic selection; no model/embedding/network dependency for protected selection; canonical truth unchanged |
| `qiven-devkit` | Connector contract conformance, generation/coverage verification, task/role schema mapping, independent negative trials and publication gate bindings | Never treat a bundle issuance as proof of delivery or a tool status as proof of control |
| ZCode source fork | Small adapter module, logical invocation hook, physical attempt guard at both runner call sites, actual tool dispatch guard, build ID and observation writer | Preserve provider binding, option validation, auth, cancellation, telemetry, background execution and release packaging |
| `qiven-docs` | This proposal and signed cross-LLM/owner deliberation | Merge and accepted migration only under owner acceptance |

### 2.1 Proposed local TypeScript contract

```ts
// qiven-harness-adapter.ts -- proposed names and types, not upstream API.
type InvocationKey = Readonly<{
  sessionId: string;
  taskId: string;
  phaseEpoch: string;
  role: "jason-worker" | "jason-brother" | "jason-extended-cognition";
  callId: string;
}>;

type PreparedInvocation = Readonly<{
  key: InvocationKey;
  receiptId: string;
  bundleSha256: string;
  activationGeneration: string;
  rendererSha256: string;
  insertedSegmentId: string;
  expiresAt?: string;
}>;

interface QivenMediationPort {
  prepare(input: {
    invocation: InvocationKey;
    task: ObservedAndClaimedTask;
    providerId: string;
    modelId: string;
    projectedRequest: PublicRequestProjection;
    abortSignal?: AbortSignal;
  }): Promise<{ prepared: PreparedInvocation; renderedSegment: string }>;
  authorizeAttempt(input: {
    prepared: PreparedInvocation;
    attempt: number;
    projectedMessages: unknown;
    projectedTools: unknown;
    modelIdentity: { providerId: string; modelId: string };
    abortSignal?: AbortSignal;
  }): Promise<AttemptPermit>;
  decideAction(input: ObservedAction): Promise<ActionDecision>;
  recordDispatch(input: AttemptDispatchObservation): Promise<void>;
}
```

`PublicRequestProjection` excludes `accountAccess`, headers, bearer tokens, auth callbacks and sensitive debug bodies. The adapter's IPC should be versioned, authenticated and bounded; the Host/Runtime already has an owner-only IPC surface under MVP-3. Exact C++ operation names and transport schema must be reconciled against CA-1 implementation before coding; no invented `qiven cognition` subcommand is represented as currently shipped.

### 2.2 Logical seam in `runner.ts`

The user's sketch is directionally correct for this seam. The following shape handles both executor return types and avoids passing sensitive `resolved` to Qiven:

```ts
executor: {
  generateText: async (request) => {
    const legacy = toLegacyRequest(request);
    const { mediated, prepared } = await qiven.prepareModelInvocation({
      legacy,
      boundModel: { providerId: resolved.providerId, modelId: resolved.modelId },
      invocation: captureHarnessInvocation(),
      abortSignal: request.abortSignal,
    });
    // prepareModelInvocation may insert one typed Qiven message segment.
    // It cannot change request.options, bound model, tool schema or auth closure.
    return this.generateTextWithResolved(
      mediated, resolved, resolveForRequest(mediated, request.options), prepared,
    );
  },
  streamText: (request) => {
    const legacy = toLegacyRequest(request);
    return (async function* () {
      const { mediated, prepared } = await qiven.prepareModelInvocation({
        legacy,
        boundModel: { providerId: resolved.providerId, modelId: resolved.modelId },
        invocation: captureHarnessInvocation(),
        abortSignal: request.abortSignal,
      });
      yield* runPreparedStream(
        mediated, resolved, resolveForRequest(mediated, request.options), prepared,
      );
    })();
  },
}
```

In real code, preserve `this` by using a bound/local `runPreparedStream`, and use the repo's concrete message types. A lazily iterated stream captures its invocation/role context at call time, **before** the caller consumes it; the generator revalidates phase and source freshness at execution time. Inject the segment through a typed helper with explicit priority/ordering and idempotence; a new `messages` array is derived without mutating the canonical conversation history. The returned object passes request validation rules; model options are bound from the original `request.options`. If `prepare` changes anything other than an allowed message addition, reject it with a typed protocol error.

### 2.3 Physical attempt seam in both runners

Pass `prepared` to `runGenerateText` and `runStreamText` (or an equivalent context object). After `createGenerateTextOptions` / `createStreamTextOptions`, and before `input.runtime.generateText(options)` / `.streamText(options)`, do the following for **every attempt**:

```ts
const permit = await qiven.authorizePhysicalAttempt({
  prepared,
  attempt,
  providerId: resolved.providerId,
  modelId: resolved.modelId,
  projectedMessages: options.messages,
  projectedTools: options.tools,
  abortSignal: attemptRequest.abortSignal,
});
if (!permit.matchesFinalProjection) throw new QivenMediationError("DeliveryMismatch");
// Publish authorization and placement verification here, not delivery.
// Only then invoke the existing input.runtime.generateText(options) or
// input.runtime.streamText(options). Do not await runtime.streamText itself.
```

`model_request_started` currently appears after options construction and before send. Make its semantics precise: authorization is not network acceptance; emit a separate authorized/placement event or move the started record after successful runtime call setup and preserve the existing accounting contract. Even successful runtime call setup is not independently proved provider acceptance: a separately qualified transport observation or real captured trial is needed before issuing `InvocationDeliveryEvent`. Route a Qiven pre-dispatch failure to a **nonretryable local failure class** that cannot be mistaken for provider throttling, stream idle timeouts, or a retryable auth error. Do not let existing generic catch/retry paths silently make an unauthorized next attempt. Keep request ID fresh for a new physical attempt; refresh auth once per existing ZCode rules. Signature-repair retry may legitimately alter earlier reasoning messages, so compare the registered injected segment in the final projection rather than requiring the whole request byte hash to equal the initial logical request. Record the whole final projection digest for correlation.

`projectRequestHistory` occurs in `*WithResolved` before the runners; `toAiSdkMessages` in `runner-options.ts` can change the provider-facing shape. The guard therefore verifies **presence, position and equality of the injected segment after both transformations**, not merely the preflight output. A model provider or AI SDK may further serialize messages; capture a mock HTTP body and at least one live model-visible task trial for the qualified profile. If final serialization changes the segment, move the check to the actual send point and requalify; do not issue a false delivery event.

### 2.4 Tool action seam, to be located by census

Instrument the actual dispatcher that has the final structured tool name, arguments, execution target and permission result. It must run before the side effect, and must cover tools exposed to main and subagent models as claimed in the profile. Model output is a proposal; classify trusted observed mechanism facts separately from claimed intent.

```ts
const observed = observeToolRequest(finalToolName, validatedArgs, target, invocationRef);
const [qivenDecision, hostDecision] = await decideBeforeDispatch(observed);
if (!qivenDecision.allow || !hostDecision.allow) return typedDenial(...);
return executeWithOutcomeObservation(observed, existingExecutor);
```

The sketch does not license parallel permission side effects: implement decision ordering according to the existing ZCode permission protocol, with no execute before both permissions. A hook denial or `PostToolUse` context cannot replace the complete action guard without profile qualification. Preserve `run_in_background`, completion notification, `TaskStop` tree termination, oversized-output persistence and ADR-0048 custody. Ensure public tool boundaries, MCP delegation, shell and filesystem writes have distinct census rows; if an auxiliary executor bypasses the central dispatcher, instrument it or exclude its operation class honestly.

## 3. Build, activation and reproducibility

Upstream [ZCode's build guide](https://github.com/zai-org/ZCode/blob/29628c9acdb81b703bbd4080c207a0e7ce5e276e/README.en.md) specifies Node.js 24.14.0 and pnpm 10.33.2 from `mise.toml`. It says an Agent source change uses `pnpm --filter @zcode/cli... build` followed by service restart; direct CLI verification is `node apps/zcode-cli/packages/cli/dist/zcode.cjs --help`. The `pnpm build:zcode` distribution build does not replace an installed `zcode` on `PATH`. Desktop development and desktop bundle have separate commands and processes.

1. Pin upstream source SHA, fork/patch SHA, lockfile and tool versions; record exact diff, test results and build manifest. Keep the fork locally isolated from production data while proving model and tool call-site coverage.
2. Build the affected packages and run the test suite. Verify the generated `zcode.cjs` digest and that an actual started CLI process is loading that exact artifact. Repeat for Web/TUI/Desktop entrypoints in the deployment profile; inspect the loaded executable/asset, not a guessed `resources/glm/zcode.cjs` path. Changing one bundle is insufficient if Desktop launches another service or if another `PATH` entry wins.
3. Package the distribution with `pnpm build:zcode`, extract it and run its `bin/zcode.mjs` from the extracted tree for a closed trial. For Desktop, build and test the platform bundle separately. Keep a rollback build, preserve existing session data, and pin the Qiven connector protocol version at startup.
4. The initial production configuration is **opt in and scope declared**, ideally on a dedicated development profile. A startup self-check attests the loaded artifact, adapter protocol, Runtime Host availability and source generations before marking governed work ready. Disable/revert the adapter by selecting the previously verified build and recording the affected trial as unqualified; do not mislabel an older process as patched.

No source fork is adopted merely because it compiles. The deployment profile and the final provider/tool boundary inventory determine whether the binary actually carries the required control. A ZCode upstream upgrade must re-run the census and negative trials before restoring the corresponding claim.

## 4. Suggested change order and reviewable PR slices

1. **Inventory PR:** source and runtime call graph; alternate model clients; all main/subagent/tool entrypoints; assertions of observed versus inferred facts; hook documentation/probe discrepancy; explicit uncovered list.
2. **Protocol PR:** Runtime adapter service and ZCode connector interfaces with authentication, deadlines, typed errors, a recorder and deterministic receipt/renderer validation. No execution-path switch yet.
3. **Model PR:** logical insertion and physical attempt gates in both runners, stream/type safety, retry/cancel/auth tests, mock-wire capture and startup build ID. Keep same options and telemetry behavior when the feature is disabled.
4. **Action PR:** proven tool dispatcher interception and Host/Qiven dual decision; background/custody parity and outcome reconciliation.
5. **Trial PR/record:** enable only the declared profile, run the real controlled CA-2 task and independent negative tests, publish coverage and provenance. Promote only after owner-governed acceptance.

This is a suggested subdivision for the **separate harness lane**. It does not add these PRs to the already declared CA-1 batch budget or authorize bypassing the CA-1 stop rule.
