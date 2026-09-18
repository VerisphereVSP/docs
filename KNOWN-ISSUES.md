# VeriSphere Known Issues & Future Work

## Gas Concerns (Non-Urgent, Monitor)

### 1. ScoreEngine.effectiveVSRay — Recursive Gas Cost

**Status:** Mitigated. Bounded fan-in implemented.

**Issue:** `effectiveVSRay` recursively calls `getIncoming` and
`getOutgoing` across LinkGraph and StakeEngine, with depth up to 32.
Each level loads dynamic arrays from storage. A claim with many
incoming links from parents that each have many outgoing links could
hit RPC node timeouts on view calls.

**Mitigation (deployed):**
- `maxIncomingEdges` (default 64): limits incoming edges processed per
  `effectiveVSRay` call. Edges beyond this limit are silently skipped.
- `maxOutgoingLinks` (default 64): limits outgoing links summed when
  computing a parent's link stake distribution.
- Both are governance-configurable via `ScoreEngine.setEdgeLimits()`.

**Remaining risk:** With both limits at 64 and depth 32, worst-case gas
is still significant. If RPC timeouts occur, increase cache duration in
the backend indexer and compute effective VS off-chain from indexed data.

### 2. StakeEngine — Ghost Lots in SideQueue

**Status:** Mitigated. Governance compaction implemented.

**Issue:** When a lot's amount reaches zero (fully burned by adverse
VS), the lot remains in the `SideQueue.lots` array with `amount = 0`.
`_recomputeSideTotal` and `_applyEpoch` iterate over all lots
including zero-amount ghosts. Over time, this increases snapshot gas.

**Why it's bounded:** StakeEngine uses lot consolidation — one lot
per user per side per post. Ghost lots can only accumulate from unique
users who were fully burned out. A post would need thousands of
distinct users who all lost 100% of their stake to cause material
gas increase.

**Mitigation (deployed):**
- `compactLots(postId, side)`: governance-callable function that
  removes zero-amount lots using swap-and-pop. O(N) per call.
  Storage-compatible (doesn't change layout).
- Ghost lots are also rescaled during position rescale (A.8 in the
  claim-spec) so their positions stay bounded even before compaction.

**Recommendation:** Run compaction when any post exceeds ~100 ghost lots.

## Security: MM_PRIVATE_KEY — RESOLVED (2026-09-11, Phase 5)

The company market maker was retired: its API routes return 410, its worker was removed, its
wallet swept, and its KMS keys destroyed. No protocol wallet key exists as an environment
variable: the relay and keeper sign via GCP Cloud KMS (HSM) under a signer identity that cannot
destroy keys, containers receive an env with every `*PRIVATE_KEY*` stripped, and the deployer
key lives only in sops and is retired after the mainnet Safe handoff. See
`MAINNET-CEREMONY.md` Step 4.7 and `PROD-SECRETS-AND-APP-ENV.md`.

## Addressed in Earlier Phases

- Stale/duplicate address files (Phase 1)
- mockUSDC.json log dump (Phase 1)
- deploy.sh volume wipe on upgrade (Phase 1)
- relay.py undefined variable (Phase 1)
- Documentation spec drift (Phase 2)
- ScoreEngineFuzz.t.sol OppositeSideStaked failures (Phase 3 — fixed
  by using dedicated challenger address in tests)
- Position rescale edge case: stakers clamped to zero rate after
  others withdraw (Phase 3 — fixed by post-snapshot _rescalePositions)
- Documentation drift: sMax decay rate, cycle handling, tranche
  terminology, KNOWN-ISSUES staleness (Phase 3)
- Full doc-vs-code reconciliation + ScoreEngine v2.1: replaced
  tranche/proportional-budget language with midpoint/per-lot rate;
  replaced "sMax decays at 0.5%" with "snap-to-leader; fallback decay
  only when empty"; added missing StakeEngine entrypoints (setStake,
  setSMaxDecayRate, etc.) to the spec; fixed MAX_CLAIM_LENGTH to 2000 (later tightened to 1000 in bundle05_d; 1000 is current)
  bytes; removed references to the (non-existent) GovernanceHub /
  YieldEngine / Treasury / Oracle contracts; replaced architecture.md
  §5.3 stale evidence-link formula with the code-correct parent-mass
  propagation; promoted ScoreEngine to v2.1 — outgoing-link bound now
  sorts by stake desc with linkPostId-asc tiebreak (matching incoming);
  added a top-N membership gate so links outside a parent's top-N
  contribute zero, restoring conservation of influence under bounded
  fan-out; getEdgeContribution view now also applies the incoming
  top-N gate (Phase 4 — this update)

## Governance is not relayable (2026-09-18, review F #1)

`onlyGovernance` and `acceptGovernance` gate on raw `msg.sender`, never on the ERC-2771
`_msgSender()`. A forwarder owner therefore cannot act as governance by upgrading the forwarder
to a suffix-appending implementation; the forwarder key and the governance timelock remain
separate trust domains. Regression: `test_F1_governanceNotRelayable`.

## Extension wallet bridge (threat-model note, reviews G/F)

The MAIN-world bridge accepts `postMessage` from any page script on wikipedia.org, limited to an
explicit method allowlist. Page scripts already hold `window.ethereum` from any injected wallet,
so the bridge grants no capability they lack; every signature still passes the wallet's own
confirmation UI. A nonce for overlay-initiated requests is queued as post-launch hardening; the
extension is not published to a store before mainnet (consent S-7).
