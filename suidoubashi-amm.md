# SuiDouBashi AMM — Security Review

**Date:** August 2026
**Chain:** Sui (Move)
**Repository:** [Jarekkkkk/SuiDouBashi_SC](https://github.com/Jarekkkkk/SuiDouBashi_SC)
**Scope:** `pool.move` (1098 lines), `amm_math.move` (242 lines), `pool_reg.move` (142 lines), `math_u128.move` (145 lines)
**Commit:** latest on `main` (last commit 2023-10-16)
**Method:** Source code review (manual line-by-line)
**Protocol status:** Inactive (website DNS unresponsive, zero activity on DefiLlama/DexScreener)

---

## Summary

SuiDouBashi is a Solidly/Velodrome-fork AMM on Sui — dual-curve pools (stable x³y+xy³ / variable xy=k), per-LP fee accounting, flash loans, single-sided zap, and TWAP oracle. Two copy-paste bugs in core accounting logic. One silently misdirects flash loan fee revenue to the wrong token index, permanently locking fees. The other compares the wrong variable in the liquidity deposit path.

**Outcome:** Report delivered to maintainer via Telegram. Protocol is inactive — no user funds currently at risk.

---

## Findings

| ID | Severity | Title | Status |
|----|----------|-------|--------|
| F-01 | HIGH | `repay_loan_y` updates wrong fee index (copy-paste) | Delivered |
| F-02 | MEDIUM | `add_liquidity_` compares `coin_x` against `opt_y` | Delivered |
| F-03 | LOW | `calculate_fee` charges +1 even on dust amounts | Delivered |
| F-04 | LOW | Oracle observation array grows without bound | Delivered |
| F-05 | INFO | Duplicate error codes (`ERR_INVALD_FEE` = `ERR_INVALD_PAIR` = 0) | Delivered |
| F-06 | INFO | 9 typos in public API identifiers | Delivered |

---

### F-01 — Flash loan Y repayment updates wrong fee index

**Where:** `pool.move` L660

In `repay_loan_y`, the Y-token fee is deposited into `self.fee.fee_y`, but the subsequent index update calls `update_fee_index_x_` instead of `update_fee_index_y_`:

```move
coin::put(&mut self.fee.fee_y, coin_fee);       // fee_y balance increases
update_fee_index_x_(self, fee);                  // but index_x gets bumped
```

The sibling `repay_loan_x` is correct. Both swap functions handle this correctly. Only the flash loan Y repayment has the mismatch — a direct copy-paste of `repay_loan_x` where `fee_x` was changed to `fee_y` but the index update call was not.

**Impact:** Every Y-token flash loan fee inflates `index_x` instead of `index_y`. LP holders receive inflated X-fee claims (potentially exceeding actual balance, causing aborts) while Y-fee claims remain permanently zero. Y-fee tokens sit in `fee.fee_y` but no LP can ever reach them — locked for the lifetime of the pool.

**Fix:** L660 — change `update_fee_index_x_` to `update_fee_index_y_`.

---

### F-02 — Liquidity deposit compares wrong variable

**Where:** `pool.move` L748

When the Y reserve is proportionally sufficient but X needs to be trimmed, the optimization check compares `coin::value(&coin_x)` against `opt_y` instead of `opt_x`:

```move
let take = if(coin::value(&coin_x) == opt_y){   // should be opt_x
    coin_x
}else{
    let take = coin::split(&mut coin_x, opt_x, ctx);
```

Copy-paste from the first branch where `coin_y` was changed to `coin_x` but `opt_y` was not changed to `opt_x`.

**Impact:** The optimization path almost never triggers in the wrong branch. In the rare case where `coin_x.value` equals `opt_y` by coincidence, more X enters the pool than intended. Normally falls through to the split path, which is functionally correct but wastes gas.

**Fix:** L748 — change `opt_y` to `opt_x`.

---

### F-03 — Fee calculation always rounds up by 1

**Where:** `pool.move` L302-304

```move
value * (fee_percentage as u64) / FEE_SCALING + 1
```

Unconditional `+1` means a swap of 1 unit pays 100% fee. Standard fix: ceiling division `(value * fee + FEE_SCALING - 1) / FEE_SCALING`.

---

### F-04 — Oracle observations grow without bound

**Where:** `pool.move` L922-929

A new `Observation` is appended every 30 minutes but never pruned. ~17,520 entries per year. Sui storage costs grow linearly and are borne by every swapper.

---

### F-05 — Duplicate error codes

**Where:** `pool_reg.move` L18-19

`ERR_INVALD_FEE` and `ERR_INVALD_PAIR` both equal 0. Impossible to distinguish which validation failed from the abort code alone.

---

### F-06 — Typos in public identifiers

9 typos including function name `smaple` (should be `sample`), `udpate_lock`, `earneed_times`, `E_INSIFFICIENT_INPUT`, `withdrawl` (×6).

---

## Method

Manual line-by-line review of all four source files. Fee accounting paths traced through `repay_loan_x/y`, `swap_for_x/y`, `update_lp_`, and `claim_fees`. Stable curve Newton solver convergence verified. K-invariant placement confirmed correct (Solidly pattern). Hot potato receipt pattern confirmed (no `drop` ability).

---

## Status

**Delivered.** Report sent to maintainer (Jarek Lin, Sui community moderator/triager at MystenLabs) via Telegram. Protocol is inactive — no deployed TVL, website unreachable.
