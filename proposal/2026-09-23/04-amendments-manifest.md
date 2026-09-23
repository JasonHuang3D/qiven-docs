# TCA Proposals — Amendments Manifest

> Date (UTC0): 2026-09-23 · Branch: `GLM-5.3/tca-amendments`
> Amends: the four TCA proposals in this directory (00-03, Jason's originals of 2026-09-23)
> Authoritative hunk view: `git diff main...GLM-5.3/tca-amendments`

Review verdict: direction approved with ten amendments (A1-A3 blocking,
B1-B4 material, C1-C3 administrative), landed as 16 edits (00=1, 01=3,
02=4, 03=8). With the amendments folded in, the original set is ready to
enter the ADR acceptance process.

## Amendment-to-landing mapping

| ID | Amendment | Lands in |
|---|---|---|
| A1 | Restore the ADR-0024 semantic-ownership admission criterion; the "at least two appropriate consumers" threshold is the superseded ADR-0007 rule and is explicitly not restored | 03 §5.1 |
| A2 | Land the byte-serialization vocabulary in Foundation as a CA-1 prerequisite (growing bounded builder + fixed-width endian put/get + length-prefixed field helpers, with a one-time zero-cost assembly comparison note); consolidate runtime's hand-rolled byte packing, reconciling the divergent u32/u64 length prefixes per format family through an explicit representation decision — never silent unification | 03 §4 CA-1 Work, 03 §6.3, 03 §6.4, 01 §4.3 |
| A3 | Profile C descriptor condition-neutrality: judgment-bearing fields (`boundary_kinds`/`external_contracts`/`explicit_ids`/`signals`) only from mechanically derivable sources or left empty; selectors requiring curator judgment earn no utility credit | 02 §7.1 |
| B1 | Tiered transport for fresh-context review: routine R2/R3 via harness-native fresh subagent (sealed review package, disclosed same-family isolated-context class); milestone and R3 via owner-relayed H1 | 02 §8.2, 03 §4 CA-2 Work |
| B2 | New fixture F-08 (fail-closed mediator availability: lifecycle, pre-flight self-check before enabling the gate, diagnosable disjoint denial taxonomy, real multi-frame contract, comment-lies classified as defects); MVP-4 lane gains the connection-model decision, the denial-split, and kit pre-flight | 02 §4.2, 03 §5.1 |
| B3 | Transition discipline: the CA obligation records a stall trigger (session/calendar budget for CA-1 to reach Profile A/B pass; missing it forces a program review); declared program packs with expiry are permitted for CA-batch work during the interim | 03 §8.5, 00 §11.3 |
| B4 | Boundary-kind growth is a declared self-report seam: the independent reviewer must verify the design's boundary-kind set ⊆ the declared descriptor's set (or that reactivation occurred) | 01 §12.3, 02 §8.4 |
| C1 | The qiven-context repository gate rejects protected-class records lacking activation selector metadata | 01 §4.1, 03 §6.1 |
| C2 | The accepting transaction records the MVP-4 reclassification in state/current + active-work in the same commit (no competing status survives) | 03 §10 Acceptance transaction |
| C3 | Canonical landing homes for the four documents; their rules carry activation selector metadata from birth and never become undifferentiated mandatory prose | 03 §10 Canonical landing homes |
