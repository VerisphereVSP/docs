# VeriSphere Known Issues & Future Work

## Gas Concerns (Non-Urgent, Monitor)

### 1. ScoreEngine — settlement and view gas

**Status:** Redesigned (whitepaper v18, 2026-10). Superseded the earlier
"bounded fan-in" mitigation.

**History:** Through v17.1 both the displayed score and settlement walked the
claim's ancestry recursively (depth 32, memoized). The fan-in/fan-out limits
(64) bounded the recursion but not the pre-cap scan over all incoming links
(~18k gas per link, time-weighted read), and a parent's cost was inherited by
every descendant. External review R3 (2026-10-01, F-D Critical) showed a
flooded hub freezing itself and every descendant, and a six-claim dense cycle
costing 28.6M gas to settle. Measured block gas limits: Avalanche mainnet 80M,
Fuji 32M.

**Design (v18):** settlement reads stored per-post snapshots (whitepaper
§4.2.6); cost is O(incoming) cold reads with no recursion, and the same
stored values serve `getEdgeContribution` and the displayed score, so the
recursive walk is retired. LinkGraph structural caps (1,000 / 1,000) become
governance-settable; the ScoreEngine outgoing top-64 is removed; the incoming
top-64 ranks by eligibility.

**Binding test:** flood one hub to the structural cap, then settle and
withdraw every descendant within a 32M block; a child's cost must not depend
on its hub's link count.

**Measured (core #31, `test/SnapshotFlood.t.sol`):** warm (one test
transaction) a hub with 1,000 active incoming links settles for 10.46M gas; a
child of that hub for 214k, the same as a child of a 1-link hub; the six-claim
dense cycle's worst settlement is 333k (28.6M before); 300 links from inactive
parents cost 1.49M and contribute nothing. **Cold** (`forge test --isolate`,
one transaction per call — the reviewer's re-check measurement) the same
points are 22.7M, 443k vs 440k, 592k and 4.1M: 71% of a Fuji block (32M) at
the cap, 28% of mainnet's (80M). Hub cost is linear in incoming links, so the
structural ceiling governance may raise the caps to is fixed at the measured
point, `ABSOLUTE_MAX_LINKS_PER_CLAIM = 1000`; the test gates are calibrated to
the cold numbers and CI runs the flood suite both ways. Live on Fuji since
2026-10-02.

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
