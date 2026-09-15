# Kriya DEX — Security Review

**Date:** July 2026
**Chain:** Sui (Move)
**Packages:** CLMM (`0xf6c05e2d...`), Spot DEX (`0xa0eba10b173538c8fecca1dff298e488402cc9ff374f8a12ca7758eebe830b66`)
**Method:** On-chain bytecode disassembly + dry-run validation + numerical simulation
**Disclosure:** [Public Issue](https://github.com/efficacy-finance/kriya-dex-interface/issues/2) (no SECURITY.md, no private reporting channel)

CLMM full address unavailable — Kriya docs site is dead and the chain indexers I had access to at the time returned the prefix only. The Spot DEX address above is confirmed via [DefiLlama adapter PR #9212](https://github.com/DefiLlama/dimension-adapters/pull/9212).

No transactions were submitted. All verification used read-only RPC queries and `sui client ptb --dry-run`.

---

## Summary

6 findings across CLMM and Spot DEX packages. The headline finding (#1) is the same overflow mask bug that caused the Cetus $223M exploit — byte-for-byte identical — but it is not exploitable through Kriya's current interfaces because the protocol computes liquidity internally rather than accepting it as a parameter.

**Outcome:** No response from protocol team. Issue remains open.

---

## Findings

| ID | Severity | Title | Status |
|----|----------|-------|--------|
| F-01 | HIGH | `checked_shlw` overflow mask wrong by factor 2^64-1 (Cetus-identical) | Unresponded |
| F-02 | LOW | Newton's method +1 rounding → DoS on stable curve edge cases | Unresponded |
| F-03 | INFO | `update_pool` has no access control (force-sync from ProtocolConfigs) | Unresponded |
| F-04 | LOW | Flash loan fee reads `swap_fee_rate` instead of `flash_loan_fee_rate` | Unresponded |
| F-05 | LOW | Permissionless pool creation (no whitelist) | Unresponded |
| F-06 | HIGH (trust) | Single EOA, 4 UpgradeCaps Policy=0, no multisig/timelock | Unresponded |

---

### F-01 — `checked_shlw` overflow mask wrong by factor 2^64-1

**Where:** `math_u256::checked_shlw` in CLMM v1/v2/v3

`checked_shlw(n)` compares `n` against `(2^64 - 1) << 192` to detect shift overflow. The correct threshold is `1 << 192`. The mask is too large by a factor of `2^64 - 1`:

```
WRONG:   0xFFFFFFFFFFFFFFFF000000000000000000000000000000000000000000000000
CORRECT: 0x0000000000000001000000000000000000000000000000000000000000000000
```

For inputs where `2^192 <= n <= WRONG_MASK`, the shift silently truncates — Move VM does not abort on shift overflow.

**Dry-run proof:**

```
sui client ptb --move-call "0xf6c05e2d...::math_u256::checked_shlw" \
  6277101735386680763835789423207666416102355444464034512896u256 --dry-run
# success (no abort — overflow not detected)
```

This is byte-for-byte identical to the Cetus $223M root cause. Verichains publicly confirmed Kriya as affected.

**Why it doesn't drain today:** Cetus exposed `liquidity_delta: u128` as a direct parameter. Kriya's `add_liquidity` takes `Coin<X>/Coin<Y>` and computes L internally. Max reachable `L * delta_sqrt` is approximately `2^128`, well below the `2^192` threshold. The bug is latent — exploitable only if a future upgrade adds a direct-liquidity parameter.

---

### F-02 — Newton's method +1 rounding causes DoS

**Where:** `utils::get_y` in Spot DEX

The `+1` delta adjustment in Newton iteration can overshoot, producing `f(new) < f(old)`. The post-swap invariant check uses strict `Gt`, so edge-case swaps abort with `EIncorrectPoolConstantPostSwap`. 14 specific reserve/amount combinations identified via numerical simulation. DoS on legitimate swaps, not a fund-drain vector.

---

### F-03 — `update_pool` has no access control

**Where:** `spot_dex::update_pool` in Spot DEX

Anyone can call `update_pool` to force-sync a pool's fee/pause config from the global `ProtocolConfigs`. Unlike `set_fee_config`/`set_pause_config`, no whitelist check. Impact is limited — it copies from admin-controlled state, cannot set arbitrary values — but allows a third party to apply config changes to specific pools without authorization.

---

### F-04 — Flash loan fee reads wrong field

**Where:** `trade::flash_loan` in CLMM v3

The Pool struct has both `swap_fee_rate` and `flash_loan_fee_rate`, but `flash_loan` reads `pool::swap_fee_rate()`. The `flash_loan_fee_rate` field is never used. If the two rates are intended to differ, flash loans are mispriced.

---

### F-05 — Permissionless pool creation

**Where:** `spot_dex::create_pool`

No whitelist or admin check on pool creation. Anyone can create pools with any token pair. Common in permissionless DEXes — noting for completeness.

---

### F-06 — Single-EOA admin with full upgrade authority

**Where:** All packages

`0x2b08...06f4` holds 4 UpgradeCaps (all Policy=0), 2 AdminCaps, 2 VersionCaps, TreasuryCap. No multisig, no timelock. Compromise of this single key = full fund drain via package upgrade.

---

## Method

- On-chain bytecode disassembly via Sui RPC, 320K+ characters across 35+ modules
- Python numerical simulation for stable curve edge cases (14 failing combos identified)
- Mainnet dry-run validation (no on-chain trace)
- Cross-reference with Cetus exploit root cause analysis

---

## Status

**Unresponded.** [Issue #2](https://github.com/efficacy-finance/kriya-dex-interface/issues/2) filed on `efficacy-finance/kriya-dex-interface`. No reply from protocol team as of September 2026.
