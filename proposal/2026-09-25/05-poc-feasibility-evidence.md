# Harness-Mediation PoC: Feasibility Evidence Note

> Status: PROPOSED evidence attachment to this PR, recorded 2026-09-25 UTC (local 2026-09-26 +0800). Owner-approved proof of concept executed by the GLM-5.3 round session (implementation delegated to fresh worker sessions per the ADR-0053 advisory division of labor). This note proves mechanical feasibility only: no governed delivery, no CA-2 evidence, no real provider account use (a local mock served every model call), nothing pushed. The ZCode working clone and scratch directories are disposable local material.

## 1. Environment feasibility (first named blocking point — retired)

- Portable Node v24.14.0 + pnpm 10.33.2 were installed side-by-side in a scratch directory in minutes; the machine's system Node (v22.11.0) was never touched.
- `pnpm install` of the full ZCode monorepo completed in **2m 28s**, including a native optional dependency build (node-gyp ssh2 crypto binding) and the desktop postinstall (node-pty prebuild). The electron binary came via the repository's configured mirror.
- The qiven-toolchain-win repository currently pins only clang-format 19.1.7 and CMake 4.4.3 (the C++ family); the Node/pnpm/electron-builder family is absent. Promotion of the PoC environment into the pinned toolchain is an owner decision for after this proposal's acceptance, not a prerequisite of it.

## 2. Patch, build, and observation smoke (Phase A — passed)

- Patch branch `poc/seam-events` on a clone pinned at `29628c9` (tag v3.14.3, equal to clone HEAD — no pin drift). Commits `5948bf6` (observation events) and `47da19f` (optional pre-send deny); four files, +132 lines total, patch file retained out-of-tree.
- Build via the narrow workspace-filter chain: **64 s** full and **30 s** incremental, producing `apps/zcode-cli/packages/cli/dist/zcode.cjs` (16.75 MB; SHA-256 `a66f89b9…` observation build, `2ea757d5…` deny build). Instrumentation markers verified present in the minified bundle.
- Smoke run (flag `QIVEN_POC=1`, one-shot headless prompt, zero-dependency mock provider on loopback, isolated scratch data home): exit 0 in ~3 s, mock answered, and the event log captured both phases:
  - `phase=logical` — `operation=agent_step`, `actorKind=main`, provider/model ids, msgCount 6, toolCount 17;
  - `phase=final-permit` — `attempt=1`, msgCount 4, `projectionDigest` present.
  The **msgCount change between the logical call and the final projection is live evidence for the two-phase design argued in earlier rounds**: a guard at the logical layer alone cannot attest the content actually sent; the post-options pre-send point observes a different, real projection.
- Negative control (flag unset): byte-identical completion, identical internal event count, and the event log file was never created — the feature-off path is inert.

## 3. Pre-send deny and upstream retry behavior (Phase C' — passed)

- With `QIVEN_POC_PERMIT=deny`, the deny fires after the final-permit event and before the SDK call:
  - **zero requests reached the wire** (mock hit count unchanged across the deny runs);
  - exit 1, error surfaced (`Turn execution failed`, underlying `QIVEN_POC permit denied (pre-send)`), never swallowed;
  - **exactly one attempt despite an eleven-attempt budget**: the failure classifier records `model_request_failed` with `errorPhase=prepare`, `retryable:false`. The runner's existing telemetry already carries per-attempt classification (exceptionType, errorPhase, retryable, attempt budget) — useful native vocabulary for the adapter's event design.
- Conclusion: the nonretryable-local-failure class that earlier rounds required for pre-dispatch denials is **natively satisfied** at this seam in the pinned revision; no custom retry suppression is needed.

## 4. Installed-Desktop loading (Phase B — partial, honest residual)

- Isolation mechanics were discovered and proven: without intervention a second instance of the installed app shares app name/userData, hits the Electron single-instance lock, and merely activates the running instance (forwarding our environment nowhere). Working isolation: `ZCODE_DESKTOP_APPLICATION_NAME` + `ZCODE_DESKTOP_USER_DATA_DIR`, plus overriding `USERPROFILE`/`HOME` to a scratch profile — because the business data root is read early from `<homedir>/.zcode/v2/setting.json` (`dataBaseDir`), which overrides `ZCODE_DATA_BASE_DIR` for the packaged app. With those set, the isolated instance configured its crash/telemetry/business paths entirely under the scratch root (verified in its logs).
- The agent-command override path is source-verified (`ZCODE_AGENT_SERVER_COMMAND` as a single executable + `ZCODE_AGENT_SERVER_ARGS_JSON` carrying the bundle path; the desktop appends the surface flag) and was accepted far enough to reach per-attempt argument parsing in the packaged host (an earlier failure traced to this session's tool channel eating backslash escapes in the JSON value — forward slashes fixed it; this is the same recorded escape-eating defect class that motivates the native-file-authoring law).
- Residual: the isolated packaged instance progressed through early main init, ARMS telemetry, and renderer/GPU initialization (Chromium caches created under the isolated userData), then **exited cleanly (exit 0, no crash entry) before any workspace session could spawn an agent**. Undiagnosed within the PoC budget. Recommended continuations: the repository's own isolated dev entry (`dev:desktop:test` with its scratch data base), or an owner-hands interactive run of the isolated packaged instance. Note that an agent process only spawns when a workspace session opens in the GUI, so a Desktop-originated model call requires GUI interaction in any case; the full-chain trial was never a headless PoC claim.

## 5. Operational findings recorded for the implementation plan

1. **Host-app environment inheritance**: agent-side shells inherit a prepared env from the running Desktop (observed: `ZCODE_DATA_BASE_DIR`, `ZCODE_PERSONAL_PROVIDER_CONFIG_FILE`, tool binary paths). Any scratch isolation must explicitly pin these; silently inherited values from the owner's live instance override them. Positively: this prepared-env channel is also a native propagation path a future adapter configuration could ride.
2. **Build-commit mismatch**: the running installed app self-reports `ZCODE_BUILD_COMMIT_ID=ab4d5e6b` while the tag `v3.14.3` points at `29628c9` — confirming the earlier caution that a version match is not a binary-equivalence proof; the trial packet must bind the observed build commit id, not only the version.
3. **Update posture**: the owner's app settings enable auto-download-and-install, but the shipped update feed is the inert localhost placeholder, so a patched build on this version has no remote auto-update wipe risk today; the feed remains a required preflight input.
4. **Provider config plumbing for trials**: the custom OpenAI-compatible provider works through a scratch `provider_config.json` (personal config env pinned to scratch; the bundled built-in release file must point at the repository's shipped builtin manifest). This is the exact mechanism a mock-driven CA-2 rehearsal can reuse without touching any real account.

## 6. What this PoC does not claim

No BeforeModelInvocation classification exists yet; the events are observation-only plus a manually switched deny; no Host, no Qiven runtime, no receipts, no protected-segment verification, no subagent-origin binding, and no CLI/TUI/Web or stream-retry coverage (the headless smoke exercised the stream path; the non-stream generate seam was compiled but not runtime-exercised). Every remaining claim of documents 00–04 still requires the gates they define.
