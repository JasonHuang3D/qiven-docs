# Evidence: The MVP-4 Owner-Live H1 Trial History and the Stringly-Typed Defect Census

> Status: EVIDENCE APPENDIX to the [ADR-0054 draft](00-adr-0054-typed-identity-law.md) and [ADR-0055 draft](01-adr-0055-simulated-loop-acceptance.md). Factual record for deliberation; not itself a decision document.
> Sources: the canonical `qiven-context` repository (public): session checkpoints, evidence audits, memory records, and ADRs cited inline. Paths below are repository-relative links into that repository's `main`.

## 1. The four owner-live H1 trials

MVP-4's exit gate row 1 was defined as a real owner-run trial: the owner enables the kit's hook configuration, boots the packaged RuntimeHost, opens a fresh agent session, and pastes six probes (an in-scope write, an in-scope edit, an in-scope Bash redirect, an out-of-scope Bash redirect that MUST be allowed, a non-mediated tool, and a no-host fail-closed probe).

| Trial | Date (UTC) | Failure | Layer at fault |
| --- | --- | --- | --- |
| 1 | 2026-09-23 | The first kit required a hook payload field the real harness never sends; every tool call in the owner's session was denied (the "deny-118" incident); the owner emergency-disabled the hooks | kit/config contract assumption |
| 2 | 2026-09-24 | One-frame serve loop vs a multi-frame client (silent drop); separately, the installation client-record contained only the host image, so every hook client was denied admission | host connection model + install record |
| 3 | 2026-09-24 | Kit preflight passed in the session-side environment but the kit shipped without config/profiles and the host defaulted its profile from the working directory; the owner-live double-click environment failed differently (the preflight-incident audit) | kit environment derivation |
| 4 | 2026-09-25 | Every probe denied with an unrelated-looking code; see §2 | protocol contract split |

Canonical records: `evidence/audits/2026-09-23-exec-process-leak-incident.md` (surrounding program), `evidence/audits/2026-09-24-mvp4-h1-kit-preflight-incident.md` (trial 3), `evidence/audits/2026-09-26-mvp4-h1-trial4-deadline-incident.md` (trial 4; the filename date is local-time). Trials 1-2 are recorded in the session checkpoints and ADR-0049.

## 2. Trial 4: root cause (reproduced and verified)

**Symptom.** All probes — including the out-of-scope probe that must be ALLOWED — returned the same denial: "no registered runtime session for this harness session (SessionStart must fire first)".

**Mechanism.**

1. The hook client assigns the session-registration event a refresh-grade deadline of 9750 ms (the harness's 10000 ms budget for that hook minus a margin); every other event carries 4750 ms.
2. The client writes the event's deadline value into the connection's HANDSHAKE frame as well as the event frame.
3. The host validates deadlines per frame kind with a policy defined in one place: only an event frame classified as session-registration may exceed 5000 ms. The handshake frame carries no event classification, so it is capped at 5000.
4. The registration connection's handshake (9750) is rejected. The client reports handshake failures for the registration event as a NON-BLOCKING advisory note ("session NOT registered") — invisible to the owner as a failure.
5. The session therefore never registers. Every subsequent tool probe's connection (handshake 4750 — legal) reaches the host and hits the unregistered-session check, which fires BEFORE any scope classification — producing a uniform, misleading denial code for all six probes, including the one that should have been allowed.

**Why the packaged preflight stayed green.** The preflight exercises only a tool-probe round trip; it never drives the session-registration path. Its "real pipe round trip" pass is compatible with any host verdict, including the unregistered-session denial.

**Verification.** The full sequence was reproduced byte-for-byte in an isolated scratch root (same binaries, same profile, verbatim payloads with paths rewritten to the scratch root): registration handshake rejected with the deadline message, all probes denied with the unregistered-session code, journal isomorphic to the owner's trial (zero registered sessions, zero verdict transactions). Reproduction artifacts are retained in the workspace (disposable) and summarized in the canonical audit.

**Class reading.** The deadline is an untyped integer whose validity policy lives on the server while its value is chosen on the client, split across two frame types of one connection. The same shape — an identity value defined in more than one place — generated the path incidents below.

## 3. The stringly-typed path census (recorded instances)

The owner's summary judgment ("hundreds of compile-time and runtime errors" across the project's sessions) motivates the law; the auditable record includes at minimum:

- **Hook-path normalization as an afterthought.** The runtime's hook surface contains a dedicated `normalize_hook_path` because governed-scope checks receive Windows-absolute, forward-slash, and MSYS-style forms; each comparison site predates the normalizer or reimplements a subset of it.
- **Installation-record image comparison.** The client-image admission check compares path strings character-by-character (case-folded, same length required) with no canonicalization — a latent separator-form defect of exactly the class this law retires (it did not fire in trial 4 only because the operating system reports a canonical image path).
- **CWD-derived default (trial 3).** The host resolved its deployment profile from the current working directory when the argument was absent; the owner's double-click environment is not the session's shell environment.
- **Template and configuration path-form drift.** Kit and workspace hook configurations have carried forward-slash command lines while native code records backslash forms; every consumer re-decides which forms are equivalent.
- **Adjacent same-class incidents (policy split across files).** A workspace router's segment classifier required a specific env-prefix order until a review test proved the taught re-call form escaped classification entirely (fixed 2026-09-26); a SQLite keyword-identifier defect compiled in DDL and failed in DML (same theme: one value, two definitions).

## 4. Cost record

- Four owner-live trials, each consuming owner hands and a diagnosis cycle; the fourth consumed a full diagnostic session including from-scratch reproduction.
- The project owner's stated cumulative cost attribution: roughly one billion tokens of session work across the project trace substantially to "implement first, fix later" deferrals of exactly this class.
- Each trial's failure was invisible to the preceding validation layer (unit/conformance tests, packaged preflight) because each validated its own layer's assumptions, never the cross-boundary contract as a whole.

## 5. What the evidence supports

1. The defect class is cross-boundary identity values with split policy definitions; paths are the most frequent instance, protocol scalars the most expensive.
2. Per-session discipline does not hold: every incident was authored by a capable session optimizing its local diff.
3. A validation layer that does not exercise the full cross-boundary sequence (including registration, not just probes) certifies nothing about that sequence — the argument for ADR-0055's Layer 1 taxonomy and Layer 2 full loop.
4. Mechanical prevention (typed primitives plus a pattern ban) is the only countermeasure with a record of working elsewhere in this project: the heredoc-authoring class stopped recurring the day a hook made it unconstructible.
