# BattleChain Confidence Pools — Competitive Audit

**Date:** July 2026
**Chain:** EVM (Solidity 0.8.26, Foundry)
**Platform:** [CodeHawks](https://codehawks.cyfrin.io/c/2026-07-battlechain-confidence-pools) (Cyfrin)
**Contest:** Jul 9–16, 2026 | 589 nSLOC | 7.25 ETH prize pool
**Method:** Manual review + Foundry PoC
**Result:** 1 validated finding (LOW)

---

## Summary

BattleChain Confidence Pools let sponsors bootstrap third-party confidence around Safe Harbor agreements. Stakers deposit capital, sponsors add an optional bonus, and each pool settles based on a moderator-flagged outcome or an expiry backstop. Stakers signal economic belief that in-scope contracts will survive the agreement term without being corrupted, and are rewarded from the bonus pool if they do.

Found a state-ordering bug where a permissionless sweep between moderator flags permanently reduces the whitehat attacker's bounty entitlement with no recovery path.

---

## Findings

| ID | Severity | Title | Status |
|----|----------|-------|--------|
| F-01 | LOW (judged; submitted as HIGH) | `sweepUnclaimedBonus()` missing `claimsStarted` latch | Valid, rewarded |

---

### F-01 — Permissionless sweep between moderator flags drains attacker's bonus

**Where:** `ConfidencePool.sol` L483–503 (`sweepUnclaimedBonus`), L357–362 (`flagOutcome` snapshot)

**What happens:**

`sweepUnclaimedBonus()` does not set `claimsStarted` (L503), so the re-flag window stays open after the sweep. When `riskWindowStart == 0`, the bonus is unreserved and fully sweepable (L483–487). On re-flag, the snapshot captures the now-zeroed `totalBonus` (L358), producing a reduced `bountyEntitlement`:

```solidity
// ConfidencePool.sol L357-362
snapshotTotalBonus = totalBonus;              // captures zeroed value after sweep
bountyEntitlement = willBeGoodFaithCorrupted
    ? snapshotTotalStaked + snapshotTotalBonus // bonus portion is zero
    : 0;
```

Because the snapshot occurs after the sweep, no later code path restores the swept bonus into `bountyEntitlement`.

**How `riskWindowStart == 0` is reachable:** The registry requires `UNDER_ATTACK` before `CORRUPTED` (AttackRegistry.sol L324). But the pool only seals `riskWindowStart` on a pool-touching transaction (L793). If `approveAttack()` and `markCorrupted()` execute with no pool interaction in between (e.g., same block), `riskWindowStart` stays zero.

**Attack path:**

1. Registry reaches `CORRUPTED` with no pool interaction during `UNDER_ATTACK` — `riskWindowStart` stays zero.
2. Moderator flags `SURVIVED`. No call to `stake()`, `contributeBonus()`, or `pokeRiskWindow()` occurred during the active-risk interval.
3. Anyone calls `sweepUnclaimedBonus()` — bonus drains to `recoveryAddress`, `totalBonus` zeroed.
4. `claimsStarted` remains false — moderator re-flags to good-faith `CORRUPTED`.
5. `bountyEntitlement = snapshotTotalStaked + 0`. Attacker permanently loses the bonus portion.

**Impact:** The protocol permanently pays the wrong recipient. The bonus (up to 100% of `totalBonus`) is irreversibly sent to `recoveryAddress` instead of the named whitehat attacker. In the PoC, the attacker receives 150 tokens instead of the expected 200 — a 25% loss. No re-flag, no subsequent call, and no admin action can restore it.

The sweep is permissionless and trivially automatable by a MEV bot watching for `SURVIVED` flags.

**Judge assessment:** Downgraded from HIGH to LOW. The 4-condition convergence (specific registry path + no pool interaction during UNDER_ATTACK + moderator SURVIVED flag + re-flag to CORRUPTED) was assessed as low likelihood. Finding valid, impact confirmed.

**PoC:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.26;

import {IAttackRegistry} from "@battlechain/interface/IAttackRegistry.sol";
import {PoolStates} from "src/libraries/PoolStates.sol";
import {BaseConfidencePoolTest} from "test/helpers/BaseConfidencePoolTest.sol";

contract PoC_SweepDrainsBounty is BaseConfidencePoolTest {
    function testSweepBetweenFlagsReducesAttackerBounty() external {
        _stake(alice, 100 * ONE);
        _stake(bob, 50 * ONE);
        _contributeBonus(carol, 50 * ONE);

        attackRegistry.setAgreementState(
            IAttackRegistry.ContractState.CORRUPTED
        );
        assertEq(pool.riskWindowStart(), 0);

        vm.prank(moderator);
        pool.flagOutcome(PoolStates.Outcome.SURVIVED, false, address(0));

        pool.sweepUnclaimedBonus();
        assertEq(pool.totalBonus(), 0);
        assertFalse(pool.claimsStarted());
        assertEq(token.balanceOf(recovery), 50 * ONE);

        vm.prank(moderator);
        pool.flagOutcome(PoolStates.Outcome.CORRUPTED, true, attacker);
        assertEq(pool.bountyEntitlement(), 150 * ONE); // should be 200

        vm.prank(attacker);
        pool.claimAttackerBounty();
        assertEq(token.balanceOf(attacker), 150 * ONE);
        assertEq(token.balanceOf(recovery), 50 * ONE);
    }
}
```

`forge test --match-test testSweepBetweenFlagsReducesAttackerBounty -vvvv`

**Recommended fix:** Gate `sweepUnclaimedBonus` on `claimsStarted`:

```solidity
function sweepUnclaimedBonus() external nonReentrant {
    if (outcome != PoolStates.Outcome.SURVIVED
        && outcome != PoolStates.Outcome.EXPIRED) {
        revert OutcomeNotEligibleForSweep();
    }
+   if (!claimsStarted) revert OutcomeNotEligibleForSweep();
```

Alternative: freeze `totalBonus` at the first `flagOutcome` and never re-snapshot on re-flag, decoupling the sweep's accounting from the re-flag's snapshot.

---

## Context

First competitive audit contest. Submitted as HIGH based on impact (irreversible loss to the whitehat). Judge downgraded to LOW based on the 4-condition convergence required. The reasoning was fair — the conditions are individually uncommon and their intersection is unlikely in normal operation. Finding valid, appeal not filed.

Contest: [BattleChain Confidence Pools on CodeHawks](https://codehawks.cyfrin.io/c/2026-07-battlechain-confidence-pools)
