# Bucket Protocol — Security Review

**Date:** July 2026
**Chain:** Sui (Move)
**Repository:** [Bucket-Protocol/v1-core](https://github.com/Bucket-Protocol/v1-core)
**Scope:** ~4,600 lines across 11 source files (buck.move, bucket.move, bottle.move, tank.move, well.move, pipe.move, reservoir.move, strap.move, interest_table.move, flask/sbuck.move, oracle modules)
**Method:** Automated lint scan (move-test-gen MOV-002) → manual line-by-line review of all flagged sites + full protocol logic
**Prior audits:** OtterSec (2023), MoveBit (2024), Hashlock, Quantstamp (V2)
**TVL at time of review:** ~$12M

---

## Summary

Bucket Protocol is a Liquity-fork CDP on Sui — borrow BUCK stablecoin against SUI collateral, with stability pool (Tank), sorted troves (Bottles), redemption, liquidation, and flash loans. Reviewed the full v1-core after MOV-002 flagged decimal scaling overflow sites. No drainable vulnerabilities found. One MEDIUM (DoS on future collateral types) with a fix PR submitted.

---

## Findings

| ID | Severity | Title | Status |
|----|----------|-------|--------|
| F-01 | MEDIUM | Decimal scaling overflow aborts TCR/health/liquidation/redemption | [PR #12](https://github.com/Bucket-Protocol/v1-core/pull/12) submitted (closed, not merged) |
| F-02 | LOW | sBUCK first-deposit inflation + permissionless collect_rewards | Reported in PR |
| F-03 | INFO | mul_factor (rounds up) vs mul_div (rounds down) rounding inconsistency | Reported in PR |
| F-04 | INFO | strap fee_rate has no upper/lower bound (admin-controlled) | Reported in PR |

---

### F-01 — Decimal scaling overflow in compute_*_value_to_*

**Where:** `bucket.move` L704, L715; `bottle.move` L161

`compute_collateral_value_to_buck` and `compute_buck_value_to_collateral` scale a value by `pow(10, decimal_diff)` using raw u64 multiplication. The upstream `mul_factor` call on the line above already promotes to u128 internally — the scaling multiplication immediately after it undoes that protection.

```move
// bucket.move L704 — collateral → buck conversion
let buck_value = mul_factor(collateral_value, price)
    * pow(10, constants::buck_decimal() - collateral_decimal);
//  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//  raw u64 multiply — overflows when decimal_diff > 0 and value is large
```

Three affected sites, all with the same pattern:

| File | Line | Expression |
|------|------|-----------|
| `bucket.move` | 704 | `collateral_raw_value * pow(10, buck_decimal - collateral_decimal)` |
| `bucket.move` | 715 | `buck_raw_value * pow(10, collateral_decimal - buck_decimal)` |
| `bottle.move` | 161 | `buck_raw_value * pow(10, collateral_decimal - buck_decimal)` |

**Impact:** These functions are called by `is_in_recovery_mode`, `is_healthy_bottle`, `is_liquidatable`, `handle_redeem`, and `record_repay_capped`. An overflow aborts all of them. For a collateral type where `collateral_decimal > buck_decimal(9)`:

- Borrow health checks fail → new borrows blocked
- Liquidation reverts → undercollateralized positions can't be cleared
- Recovery mode detection reverts → protocol can't enter recovery mode
- Redemption reverts

**Current risk:** Currently deployed buckets use SUI (9 decimals = buck_decimal), so `pow(10, 0) = 1` — no overflow possible on existing collateral. The issue surfaces if a future `create_bucket` registers a collateral type with more than 9 decimals.

**OtterSec relationship:** OtterSec V1 audit finding OS-BKT-ADV-00 ("Improper Conversion") addressed a missing decimal adjustment in `record_repay_capped`. The fix introduced `compute_buck_value_to_collateral`. The overflow in the fixed code is a separate issue — OtterSec's patch created the surface for this bug.

**Fix submitted:** Cast both operands to u128 before multiplying, then downcast. Matches the pattern `mul_factor` already uses internally (framework/sources/math.move L7–11).

```move
// before
collateral_raw_value * pow(10, constants::buck_decimal() - collateral_decimal)

// after
(((collateral_raw_value as u128)
    * (pow(10, constants::buck_decimal() - collateral_decimal) as u128)) as u64)
```

PR: [Bucket-Protocol/v1-core#12](https://github.com/Bucket-Protocol/v1-core/pull/12) — closed without merge. No response from team.

---

### F-02 — sBUCK first-deposit inflation gap

**Where:** `flask/sbuck.move`

`collect_rewards` is permissionless. First deposit converts 1:1 (BUCK → sBUCK). An attacker could deposit 1 wei, then call `collect_rewards` to inflate the exchange rate, causing subsequent depositors to receive 0 sBUCK and lose their BUCK.

Mitigated: `sbuck_out == 0` aborts, preventing actual fund loss. But it does enable deposit DoS. Additionally, `deposit` is whitelist-gated, limiting direct exploitation.

---

### F-03 — Rounding inconsistency between math libraries

**Where:** `framework/math.move` vs `protocol/math.move`

`mul_factor` rounds up. `mul_div` rounds down. Both are used in value conversions throughout the protocol. No immediate exploit, but the inconsistency means the same conceptual operation can produce different results depending on which function is called.

---

### F-04 — strap fee_rate unbounded

**Where:** `strap.move`

`fee_rate` has no upper or lower bound. Admin (AdminCap holder) can set any value. Within admin trust boundary, but no sanity check to prevent accidental misconfiguration.

---

## What I checked and cleared

| Area | Result |
|------|--------|
| Flash loan (hot potato + `is_not_locked`) | Clean — receipt has no `drop`, `is_not_locked` checked on entry |
| Oracle staleness + source config | Clean on main branch (test-only branch has staleness check commented out — expected) |
| Borrow health + recovery mode | Clean — all checks present |
| Redemption (sorted linked list traversal) | Clean |
| Liquidation (normal + recovery mode) | Clean |
| Bottle insertion (`insertion_place` hint) | Clean — hint is a gas optimization, wrong hint still inserts at correct position |
| Flash-deposit prevention | Clean — tx digest comparison (L363–366) |
| Flash loan TCR protection | Clean — borrowed amount included in TCR calculation (L615, L523) |
| Interest model | Clean — u256 throughout, overflow checked |
| PSM (Pipe) | Clean — witness-based access control |
| Transfer bottle (duplicate check) | Clean |
| Admin functions (AdminCap) | Clean |

---

## Method

Phase 1 — Automated: ran move-test-gen `--lint` against full repo. Initial output: 71 findings (67 MOV-001, 4 MOV-002). Filtered MOV-001 permissionless functions (borrow/repay/liquidate/deposit are intentionally public). Remaining MOV-002 hits on decimal scaling sites led to F-01.

Phase 2 — Manual: line-by-line review of all 11 source files. Traced every value-moving path: borrow → repay → liquidate → redeem → flash loan → stability pool → PSM. Checked oracle integration, interest accumulation, fee distribution, and admin controls.

---

## Status

**Closed, not merged.** [PR #12](https://github.com/Bucket-Protocol/v1-core/pull/12) sat for ~6 weeks with no response from the Bucket team. Fork deleted during profile cleanup. The PR and its full diff remain visible on GitHub.

No funds currently at risk — SUI-only buckets have decimal_diff = 0. Risk activates only on future collateral types with >9 decimals.
