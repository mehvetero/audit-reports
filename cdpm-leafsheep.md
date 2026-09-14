# CDPM / LeafSheep — Security Review

**Date:** August 2026
**Chain:** Sui (Move)
**Package:** `0x573584cc4698e82fd85f2b54e64ad4cd901c42b768f7628ec167bf2d24aa2aa7`
**Method:** On-chain bytecode disassembly (`sui move disassemble`) + read-only object queries
**Upgrade policy:** `only_dep_upgrades` (no code-level patching possible)

No transactions were submitted and no funds were moved during this review.

---

## Summary

Four `public` functions meant for the `sui-prover` formal verification toolchain (`#[spec_only]`) compiled into production bytecode as regular public functions. The `#[spec_only]` attribute is a custom annotation — not a Move language primitive — so the compiler ignores it. These functions take `&mut PositionManager` (a shared object) with zero authorization checks, allowing any address to pull lending assets from any PositionManager.

**Severity:** Critical
**Outcome:** Protocol team performed an emergency drain of all lending positions (~$300K at risk). Credited publicly in their post-mortem.

---

## Findings

### C-01 — Unauthenticated lending drain via `#[spec_only]` public functions

**Affected functions:**

| Function | Effect |
|---|---|
| `spec_call_pull_from_scallop_lending<T>` | Returns `Balance<MarketCoin<T>>` from any PM |
| `spec_call_pull_from_kai_lending<T, YT>` | Returns `Balance<YT>` from any PM |
| `spec_call_add_to_scallop_lending<T>` | Inserts arbitrary `ScallopVault` into any PM |
| `spec_call_add_to_kai_lending<T, YT>` | Inserts arbitrary `KaiVault` into any PM |

All four are `public`, take `&mut PositionManager`, and forward directly to private lending helpers without calling `assert_caller_authorized` or any other access check.

**Bytecode evidence (pull wrapper):**

```
public spec_call_pull_from_scallop_lending<Ty0>(Arg0: &mut PositionManager, Arg1: u64): Balance<MarketCoin<Ty0>> * u64 {
B0:
    0: MoveLoc[0](Arg0: &mut PositionManager)
    1: MoveLoc[1](Arg1: u64)
    2: Call pull_from_scallop_lending<Ty0>(&mut PositionManager, u64): Balance<MarketCoin<Ty0>> * u64
    3: Ret
}
```

Three instructions and a return. No branch, no abort path, no authorization.

**Attack scenario:**

A single PTB drains all Scallop lending from a target PositionManager:

```
Cmd 0: cdpm::spec_call_pull_from_scallop_lending<SUI>(victim_pm, u64::MAX)
       → (Balance<MarketCoin<SUI>>, u64)

Cmd 1: coin::from_balance<MarketCoin<SUI>>(Result(0).0)
       → Coin<MarketCoin<SUI>>

Cmd 2: scallop::redeem<SUI>(version, market, Result(1), clock)
       → Coin<SUI>

Cmd 3: TransferObjects([Result(2)], attacker)
```

The `spec_call_add_to_*` functions also enable a griefing attack: injecting a small lending entry into a victim PM prevents `user_close_pm` from succeeding (`ELendingNotEmpty`), locking the PM until the injected entry is redeemed.

---

### I-01 — Source-bytecode mismatch

The GitHub source contains code not present in the deployed bytecode:

| In source | In bytecode |
|---|---|
| `agent_create_position` / `agent_destroy_position` | No |
| `pool_id: ID` field in PositionManager | No |
| `Option<Position>` wrapping | No (stored directly) |
| `AgentPositionCreated` / `AgentPositionDestroyed` events | No |
| Error codes 1010, 1011, 1012 | No |

The deployed contract stores `Position` directly, not `Option<Position>`. The `option` module has zero references in bytecode. Anyone auditing the source without comparing to deployed bytecode would review code that isn't running on-chain.

---

## Root Cause

`#[spec_only]` is recognized only by `sui-prover`. The project's own SPEC.md documents this: "Regular `sui move build` ignores the attribute." The 17 read-only `spec_*` accessors (e.g. `spec_fee_house_rate`) are harmless because they take immutable references. The 4 `spec_call_*` wrappers are dangerous because they take `&mut PositionManager` and move lending assets.

This is a vulnerability class, not a one-off bug. Any Sui Move package using custom attributes to mark functions as non-production while leaving them `public` carries the same risk. The `#[test_only]` attribute, by contrast, is a Move language primitive — the compiler strips tagged functions from non-test builds.

---

## Exposure at Time of Disclosure

7 out of 9 active PositionManagers held lending positions. Total at risk: approximately $7,200 across Scallop SUI and USDC lending. Balance bags, fee bags, and Cetus DLMM positions were not affected.

The protocol team subsequently moved approximately $300K in total assets to safety through an emergency drain.

---

## Timeline

- **Aug 18, 2026:** Vulnerability identified through bytecode disassembly
- **Aug 18, 2026:** Private advisory submitted to protocol team
- **Aug 18, 2026:** Protocol team confirmed, began emergency drain
- **Aug 19, 2026:** Emergency drain completed, all lending positions cleared
- **Aug 19, 2026:** Public credit in protocol post-mortem — [protocol team acknowledgment](https://x.com/pikapikasui/status/2089890056890900836)

---

## Status

**Fixed** — emergency drain removed all lending assets from affected PositionManagers. The underlying package cannot be patched (`only_dep_upgrades`), so the functions still exist in bytecode but there are no longer lending assets to drain. A new package deployment is required for a permanent fix.
