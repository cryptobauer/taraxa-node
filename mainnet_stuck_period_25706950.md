# Mainnet Incident: Chain Stuck at Period 25706950

**Date of investigation:** April 2026
**Last technical validation update:** April 12, 2026
**Affected network:** Taraxa Mainnet
**Impact:** All nodes stuck — re-syncing does not resolve the issue

---

## 1. Symptoms

All mainnet nodes are stuck at PBFT period 25706950. On startup, node logs show cascading errors:

```
DAG block #03a283bb… with 73779606 level failed on VDF verification
  with pivot hash #2d38e144…
  reason Invalid VdfSortition: VRF verify failed.
  VRF input 840465c996a04967a886f2b5ec23ec67420e6504056b64597be4479cdb4fc8cd55ed7d883b17
```

Followed by:
```
DAG block #03a283bb… could not be added to DAG on startup since it has missing tip/pivot
```

The block `#03a283bb` is the **anchor** (pivot DAG block) for period 25706950. Without loading it, the period cannot finalize, and the chain halts.

---

## 2. Error Trace

All four error messages were traced to `dag_manager.cpp` in `recoverDag()` (current tree: around lines 461–530):

| Error | Source | Line |
|---|---|---|
| `VRF verify failed` | `VdfSortition::verifyVdf()` → `verifyVrf()` → `crypto_vrf_verify()` | `libraries/vdf/src/sortition.cpp:81–86`, `libraries/common/src/vrf_wrapper.cpp:35–37` |
| `failed on VDF verification` | `recoverDag()` verification-failed branch | `libraries/core_libs/consensus/src/dag/dag_manager.cpp:513–516` |
| `missing tip/pivot` | `pivotAndTipsAvailable()` fails (block not in DAG graph) | `libraries/core_libs/consensus/src/dag/dag_manager.cpp:528–530` |
| `could not be added to DAG` | `addDagBlock()` startup insertion fails | `libraries/core_libs/consensus/src/dag/dag_manager.cpp:523–525` |

The VRF failure at step 1 causes `break`, skipping the block. Subsequent blocks that depend on it as a tip/pivot then also fail.

---

## 3. Root Cause

### 3.1 The `proposal_period_levels_map` collision

The `proposal_period_levels_map` is a RocksDB column family that maps DAG levels to PBFT proposal periods. When a period is finalized in `final_chain.cpp` (line 225–228), an entry is written:

```
key = anchor_level + kMaxLevelsPerPeriod (100)
value = period
```

The C++ function `getProposalPeriodForDagLevel(level)` uses `Seek(level)` — it finds the **first key ≥ level** — to determine which proposal period governs a given DAG level. This period's block hash is used as part of the VRF input when verifying DAG blocks.

### 3.2 The anomaly

Diagnostic tool output shows the map entries around the stuck block's level:

```
map[73779601] = period 25706871  (gap: 2 levels)
map[73779602] = period 25706872  (gap: 1 levels)
map[73779603] = period 25706873  (gap: 1 levels)
map[73779604] = period 25706875  (gap: 1 levels)
map[73779605] = period 25706876  (gap: 1 levels)
map[73779606] = period 25706949  (gap: 1 levels)  ← EXACT target level — JUMP OF 73 PERIODS
map[73779608] = period 25706877  (gap: 2 levels)  ← first entry ABOVE target — DROPS BACK 72 PERIODS
map[73779609] = period 25706878  (gap: 1 levels)
map[73779611] = period 25706879  (gap: 2 levels)
```

The entry at key `73779606` is **non-monotonic** — it jumps from period ~25706876 to 25706949, then drops back to 25706877. This happened because:

- Period 25706949's anchor was at DAG level **73779506**
- The map entry written was `73779506 + 100 = 73779606 → 25706949`
- This key falls inside the existing monotonic sequence, overriding the lookup for level 73779606

### 3.3 The timeline

```
T1: Block #03a283bb is created at DAG level 73779606.
    Seek(73779606) finds map[73779608] = period 25706877.
    VRF proof is computed with period 25706877's block hash (0xf3275381…).
    Block is validated and stored in the non-finalized DAG.

T2: Period 25706949 is finalized.
    Its anchor is at level 73779506.
    Map entry written: 73779506 + 100 = 73779606 → 25706949.
    This INSERTS a new entry at exactly key 73779606.

T3: Node restarts, recoverDag() runs.
    Seek(73779606) now finds map[73779606] = period 25706949 (the new entry).
    Period 25706949's block hash is 0x4967a886… (different from 0xf3275381…).
    VRF input changes → crypto_vrf_verify fails → block rejected.
```

### 3.4 Evidence

The diagnostic tool confirmed this by reconstructing VRF inputs:

```
Current lookup:  Seek(73779606) = period 25706949
  Period block hash: 0x4967a886f2b5ec23ec67420e6504056b64597be4479cdb4fc8cd55ed7d883b17
  VRF input: 840465c996a04967a886f2b5ec23ec67420e6504056b64597be4479cdb4fc8cd55ed7d883b17
  ✓ EXACTLY matches the VRF input in the error log

Fallback lookup: Seek(73779607) = period 25706877
  Period block hash: 0xf32753813299dfcda83bacd25b3a5b75316004d7e168ed358e5df6fbd9177d04
  VRF input: 840465c996a0f32753813299dfcda83bacd25b3a5b75316004d7e168ed358e5df6fbd9177d04
  → This is the ORIGINAL VRF input the block was signed with
```

The `PROPOSE_PERIOD MISMATCH: Seek(level)=25706949 vs Seek(level+1)=25706877` confirms the collision.

---

## 4. Why all nodes are stuck

- **Running nodes that restart:** `recoverDag()` re-verifies non-finalized blocks with the post-collision propose_period → VRF fails → anchor block not loaded → period 25706950 cannot finalize.

- **Fresh-syncing nodes:** After syncing finalized periods (including 25706949 and its map entry), they receive `#03a283bb` from peers → same VRF mismatch → block rejected.

- **Re-syncing is not a fix:** The map entry is created deterministically during finalization of period 25706949. Every node that reaches this point will have the same corrupted lookup.

---

## 5. Design flaw analysis

The `proposal_period_levels_map` assumes that `kMaxLevelsPerPeriod = 100` provides sufficient headroom — that non-finalized DAG blocks will never exist at level `anchor_level + 100`. This assumption breaks under degraded conditions:

- 73 periods (25706877 → 25706949) were finalized while DAG levels barely advanced (~73779506 to ~73779508)
- This means PBFT consensus was running many rounds while DAG block production was stalled
- The combination allowed `anchor_level + 100` for period 25706949 to land exactly at a level with active non-finalized blocks

### Why increasing `kMaxLevelsPerPeriod` is NOT a fix

1. It's a **consensus constant** — changing it is a hard fork requiring all nodes to upgrade simultaneously
2. It doesn't fix the design flaw — just widens the safety margin; the same collision can occur at any value under sufficiently degraded conditions
3. It **breaks existing DB data** — all existing map entries were written with offset 100; new entries with a different offset would interleave incorrectly

---

## 6. Evaluated solutions

### Option A: Skip VRF re-verification during recovery (REJECTED)

Rationale: Already-stored blocks were validated when first received; recovery only needs to rebuild in-memory structures.

Pros:
- Smallest change (1 line)
- Unblocks recovery immediately

Cons:
- Does NOT fix the live `verifyBlock()` path — if the same race occurs during normal operation, newly received blocks would be rejected
- Reduces security of the recovery path

### Option B: Fallback `Seek(level + 1)` on VRF failure (IMPLEMENTED)

Rationale: On VRF failure, retry verification with the *next* map entry (`Seek(level + 1)`), which was the propose_period in effect before the collision entry was inserted.

Pros:
- Fixes both `recoverDag()` AND `verifyBlock()` (live path)
- Only 2 adjacent lookup positions are tried (`Seek(level)` then `Seek(level+1)`) — bounded behavior
- VRF proof must cryptographically verify against one of the two periods

Cons:
- Slightly more complex than Option A
- `Seek(level+1)` is still a heuristic; it is not a formal guarantee of the originally used propose period in all possible map layouts

### Option C: Store propose_period with non-finalized DAG blocks (DEFERRED)

Rationale: Record the exact propose_period used at validation time alongside each non-finalized block. Recovery uses the stored value instead of re-looking it up.

Pros:
- Eliminates the entire class of map collision bugs
- Clean architectural fix

Cons:
- Requires DB schema change (new field in non-finalized DAG block storage)
- More invasive change for a time-critical fix
- Recommended as a follow-up improvement

### Block exclusion / blacklist (REJECTED)

Rationale: Exclude the specific bad block hash via config or hardfork.

Why it doesn't work:
- The block itself is **not bad** — it was valid when created
- Block `#03a283bb` is the **anchor for period 25706950** — excluding it means the period can never finalize
- There is no existing block exclusion mechanism in the Taraxa codebase
- Would require discarding the PBFT block for period 25706950 and re-voting, which is impractical

---

## 7. Implemented fix (Option B)

### Files changed

**`libraries/core_libs/storage/include/storage/storage.hpp`**
- Added `getNextProposalPeriodForDagLevel(uint64_t level)` declaration

**`libraries/core_libs/storage/src/storage.cpp`**
- Implemented `getNextProposalPeriodForDagLevel()` — delegates to `getProposalPeriodForDagLevel(level + 1)` to skip any exact-match entry at `level`

**`libraries/core_libs/consensus/src/dag/dag_manager.cpp`** — Two call sites:

1. **`recoverDag()`**: VRF/VDF verification extracted into a `verifyWithProposePeriod` lambda. On failure, retries with `getNextProposalPeriodForDagLevel()`. If the fallback succeeds, the block loads normally.

2. **`verifyBlock()`**: Same fallback pattern via `verifyVdfWithPeriod` lambda.
   Follow-up correctness fix: transaction retrieval (`getTransactions(...)`) now runs **after** fallback period resolution, so transaction filtering and all subsequent checks use the same effective proposal period.

### Deployment strategy

- **Minimum viable fix:** Patch 5 validators that collectively hold ≥2/3+1 of total stake. They finalize period 25706950, moving the stuck block into finalized period data. Unpatched nodes can then sync past this point normally.

- **Full rollout:** All validators should eventually apply the fix to prevent recurrence. The bug is structural and can happen again under similar degraded network conditions.

---

## 8. Malicious attack assessment

**Verdict: Unlikely deliberate, but theoretically possible.**

To trigger this intentionally, an attacker would need to:
1. Suppress DAG level advancement while maintaining PBFT liveness
2. Force many periods to finalize with low-level anchors so `anchor_level + 100` catches up to the DAG frontier
3. This requires significant stake and network influence

The observed pattern (73 periods finalized with minimal DAG advancement) is consistent with natural network degradation/congestion. The attacker also cannot control which specific level becomes the anchor, and the effect only manifests on restart — making it a poor deliberate DoS vector.

**However**, the underlying design flaw means this **can and will happen again** under organic network stress. The fix should be deployed regardless of intent.

---

## 9. Diagnostic tooling

A Rust diagnostic binary (`rustaxa-storage`) was used during investigation. In this workspace snapshot, the binary artifacts are present under `rust/target/{debug,release}/`, while the original source path referenced during investigation (`rust/crates/rustaxa-storage/src/main.rs`) is not present in the current checkout.

The tool capabilities used in the investigation were:

- Reads all 36 RocksDB column families
- Identifies the last finalized period, non-finalized DAG blocks, and the target anchor
- Simulates `recoverDag()` flow (proposal period lookup, pivot/tips availability)
- Reconstructs VRF inputs to compare with error log values
- Dumps `proposal_period_levels_map` entries to identify non-monotonic anomalies
- Performs `Seek(level)` vs `Seek(level+1)` comparison to detect collisions

Usage (when the crate source is available in the checkout):
```bash
cd rust && cargo build --release -p rustaxa-storage
./target/release/rustaxa-storage /path/to/db
```

---

## 10. Key constants and code references

| Constant | Value | Location |
|---|---|---|
| `kMaxLevelsPerPeriod` | 100 | `libraries/common/include/common/constants.hpp:18` |
| `kDagExpiryLevelLimit` | 1000 | config |

| Function | File | Line (current tree) |
|---|---|---|
| `recoverDag()` | `libraries/core_libs/consensus/src/dag/dag_manager.cpp` | 461–535 |
| `verifyBlock()` | `libraries/core_libs/consensus/src/dag/dag_manager.cpp` | 589–735 |
| `getProposalPeriodForDagLevel()` | `libraries/core_libs/storage/src/storage.cpp` | 1256–1269 |
| `getNextProposalPeriodForDagLevel()` | `libraries/core_libs/storage/src/storage.cpp` | 1271–1276 |
| `addProposalPeriodDagLevelsMapToBatch()` | `libraries/core_libs/storage/src/storage.cpp` | 1282–1283 |
| Map write during finalization | `libraries/core_libs/consensus/src/final_chain/final_chain.cpp` | 223–227 |
| `makeVrfInput()` | `libraries/common/src/vrf_wrapper.cpp` | 48–53 |
| `VdfSortition::verifyVdf()` | `libraries/vdf/src/sortition.cpp` | 81–105 |
| `DagBlock::verifyVdf()` | `libraries/types/dag_block/src/dag_block.cpp` | 126–137 |

---

## 11. Specific on-chain data

| Item | Value |
|---|---|
| Stuck period | 25706950 |
| Target anchor block | `#03a283bb…` |
| Target anchor level | 73779606 |
| Current propose_period (post-collision) | 25706949 |
| Original propose_period (pre-collision) | 25706877 |
| Period 25706949 anchor level | 73779506 |
| Map collision key | 73779506 + 100 = **73779606** |
| Period 25706949 block hash | `0x4967a886f2b5ec23ec67420e6504056b64597be4479cdb4fc8cd55ed7d883b17` |
| Period 25706877 block hash | `0xf32753813299dfcda83bacd25b3a5b75316004d7e168ed358e5df6fbd9177d04` |
| VRF input from error log | `840465c996a04967a886...` |
| Sortition params last changed | Period 371/372 (not a factor) |
| Non-finalized blocks at target level | 4 blocks |
| Total non-finalized levels | 6 levels (73779464–73779609) |

> Note: These values are from the analyzed mainnet DB snapshot used during investigation.

---

## 12. Forensic analysis — Block producers and malicious intent

### 12.1 Recovered identities

Using ECDSA signature recovery (`ecrecover`) on DAG blocks and PBFT block RLP data, the following actors were identified:

**a) Target anchor block (the stuck block `#03a283bb`):**

| Field | Value |
|---|---|
| Producer (sender) | `0x264a30e880d3b53bb2cdeb234e34201ebf9a9965` |
| Level | 73779606 |
| Pivot | `0x2d38e144534dc908ae8189eac16c23b0e16d5299d5761c40b7dfb3c0046e241b` |
| Timestamp | 1772723364 (2026-03-05 15:09:24 UTC) |
| Transactions | 15 |

**b) PBFT block for collision-causing period 25706949:**

| Field | Value |
|---|---|
| Proposer | `0xa6cec53b4d7920709f05ca1f7ac0d66338d29cb9` |
| Period | 25706949 |
| Anchor (pivot DAG block) | `0x861dce07265e03eeb28672cda185df2b537a8164c16f517e1afb367fa39a80d3` |
| Timestamp | 1772723363 (2026-03-05 15:09:23 UTC) |

**c) PBFT block for original (pre-collision) period 25706877:**

| Field | Value |
|---|---|
| Proposer | `0x138c1b9baea7d93a981ba694f5318f735c997298` |
| Period | 25706877 |
| Anchor | `0x3014ed8003735307145f64ad5b8a590d8cbe173ede278184ce6271bbf889af8f` |
| Timestamp | 1772723114 (2026-03-05 15:05:14 UTC) |

**d) All 4 DAG block producers at level 73779606:**

| Block hash | Producer |
|---|---|
| `0x03a283bb…` | `0x264a30e880d3b53bb2cdeb234e34201ebf9a9965` ← target anchor |
| `0x1d40e3ab…` | `0xe50b5452b2e8435404dbe06e6a05410c47b7583d` |
| `0x4c512143…` | `0xc9b2ed9876bd463f67a412bb7ce3ea269e3fdafa` |
| `0x9df5ec85…` | `0x1d8240c50cc3eefa80bca068624236f35ade7e1b` |

### 12.2 Malicious intent assessment

**Verdict: No evidence of deliberate attack.**

Evidence against malicious intent:

1. **Multiple independent actors.** The stuck block was produced by one of 4 different validators at level 73779606 — not a single actor dominating the level.

2. **Different proposers.** The PBFT proposer for period 25706949 (`0xa6cec53…`) is a completely different address from the stuck block producer (`0x264a30e…`). No single entity controlled both sides of the collision.

3. **Timing race, not coordination.** The PBFT finalization timestamp (15:09:23) and the stuck block timestamp (15:09:24) are 1 second apart — consistent with a natural timing race, not a coordinated attack.

4. **73 periods in ~4 minutes.** The 73 periods (25706877→25706949) that finalized with minimal DAG advancement occurred over ~4 minutes (15:05:14→15:09:23). This pattern is consistent with network congestion or partition causing slow DAG growth while PBFT consensus continued.

5. **What an attacker would need.** To trigger this deliberately, an attacker would need to: suppress DAG level advancement network-wide while keeping PBFT liveness, then ensure a specific anchor lands at a level whose `anchor_level + 100` collides with active non-finalized blocks. The attacker cannot control which level becomes the anchor for a given period.

6. **Poor attack vector.** The effect only manifests on node restart — running nodes are unaffected until rebooted. An attacker cannot trigger restarts remotely.

### 12.3 Can it happen again?

**Yes.** The underlying design flaw is structural. Anytime the DAG stalls while PBFT keeps finalizing (network stress, large block propagation delays, temporary partitions), the same `anchor_level + kMaxLevelsPerPeriod` collision can occur. The fix (Option B) prevents the collision from causing a chain halt, but the non-monotonic map entry will still be written. The long-term fix (Option C: storing propose_period alongside non-finalized blocks) would eliminate the class of bugs entirely.

---

## 13. Post-fix validation findings (added April 12, 2026)

### 13.1 Additional correctness finding

During review of the initial Option B patch, one additional issue was found in `verifyBlock()`:

- `propose_period` could switch to fallback after VRF/VDF retry, but transaction retrieval had already run using the original period.
- This could make period-dependent transaction filtering inconsistent with the final `propose_period`.

This has been fixed by moving transaction retrieval to run after fallback period resolution.

### 13.2 Regression test added

New test:

- `DagBlockMgrTest.verify_block_uses_fallback_period_for_transaction_lookup`
  (`tests/dag_block_test.cpp`)

What it verifies:

- A DAG block that requires fallback (`Seek(level)` collides, `Seek(level+1)` is correct) is validated using fallback period for both VRF/VDF and period-sensitive transaction lookup.

### 13.3 Tests executed after patch

Executed in `/buildfix`:

- `dag_block_test`:
  - `DagBlockMgrTest.proposal_period`
  - `DagBlockMgrTest.verify_block_uses_fallback_period_for_transaction_lookup`
  - `DagBlockMgrTest.incorrect_tx_estimation`
- `full_node_test`:
  - `FullNodeTest.reconstruct_dag`

Result: all passed (sequential execution).
