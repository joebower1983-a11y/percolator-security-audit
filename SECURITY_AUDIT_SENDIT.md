# Percolator Launch — Security Audit Report

**Auditor:** OpenClaw Automated Audit  
**Date:** 2026-02-15  
**Scope:** `program/src/percolator.rs` (on-chain program), `percolator/src/percolator.rs` (risk engine), `percolator/src/i128.rs`  
**Commit:** HEAD at time of audit

---

## Executive Summary

The Percolator on-chain program is a well-engineered perpetual DEX with formal verification (Kani), comprehensive checked/saturating arithmetic, and layered authorization. However, several exploitable issues were identified ranging from critical fund drainage to medium-severity economic manipulation vectors.

---

## Critical Findings

### C1: DEX Oracle Flash Loan Price Manipulation (Fund Drainage)

**File:** `program/src/percolator.rs`, lines ~2101-2200 (PumpSwap oracle), ~1800-1900 (Raydium CLMM oracle)  
**Function:** `read_pumpswap_price_e6`, `read_raydium_clmm_price_e6`, `read_meteora_dlmm_price_e6`  
**Severity:** Critical

**Description:** DEX-based oracle readers (PumpSwap, Raydium CLMM, Meteora DLMM) read spot prices directly from AMM pool reserves with **zero staleness checks and zero TWAP protection**. The code acknowledges this in comments ("vulnerable to flash-loan manipulation") but the circuit breaker (`oracle_price_cap_e2bps`) is configured at 1% per slot (10,000 e2bps) for Hyperp mode and **0 (disabled)** by default for non-Hyperp.

**Proof of Concept:**
1. Attacker creates a market using PumpSwap pool as oracle (`index_feed_id` = pool address)
2. In a single transaction: flash-loan → manipulate pool reserves → open huge leveraged position at manipulated price → restore pool → close position at real price → profit
3. Even with circuit breaker enabled, the attacker can grind the price over multiple slots (1% per slot = ~100% in 100 slots ≈ 50 seconds)

**Proposed Fix:** 
- Require TWAP or multi-block aggregation for DEX oracles
- Set minimum `oracle_price_cap_e2bps` (e.g., max 10 e2bps = 0.001% per slot) for DEX oracle markets
- Or: reject DEX oracles entirely for new market creation without explicit admin acknowledgment

---

### C2: ResolveMarket PnL Calculation Uses Wrong Formula (Incorrect Settlement)

**File:** `program/src/percolator.rs`, lines ~3440-3480  
**Function:** `process_instruction` → `KeeperCrank` when `is_resolved`  
**Severity:** Critical

**Description:** When market is resolved and crank force-closes positions, the PnL calculation uses:
```rust
let pnl_delta = pos.saturating_mul(settle.saturating_sub(entry)) / 1_000_000i128;
```
This is a **linear PnL formula** (`pos * (settle - entry) / 1e6`), but the risk engine's `mark_pnl_for_position` uses a **coin-margined formula** (`pos * diff / oracle`). The mismatch means resolved-market settlement prices are wrong for coin-margined perps, potentially over-paying some users and under-paying others.

For a long position: engine uses `(oracle - entry) * abs_pos / oracle`, but resolve uses `pos * (settle - entry) / 1e6`. These produce very different values.

**Proof of Concept:**
1. Market has coin-margined positions (collateral = base asset)
2. Admin resolves market at price P
3. Crank settles positions using linear formula instead of coin-margined formula
4. Users with large positions receive incorrect payouts, potentially draining vault

**Proposed Fix:** Use `RiskEngine::mark_pnl_for_position()` or the existing `oracle_close_position_core()` instead of inline PnL calculation in the resolved-market crank path.

---

### C3: WithdrawInsurance Doesn't Reduce Vault Balance

**File:** `program/src/percolator.rs`, lines ~4410-4490  
**Function:** `process_instruction` → `WithdrawInsurance`  
**Severity:** Critical

**Description:** The `WithdrawInsurance` instruction zeroes `engine.insurance_fund.balance` and transfers tokens from the vault, but **never decrements `engine.vault`**. This violates the conservation invariant `V >= C_tot + I`. After withdrawal, the engine believes it has more vault balance than actually exists in the token account, which can cause:
- Subsequent withdrawals to succeed at the engine level but fail at the token transfer level (bricking the market)
- Or worse: if the vault has other funds (from deposits), those user funds can be drained via the insurance withdrawal

**Proof of Concept:**
1. Market has insurance_fund.balance = 100 units, vault = 200 units (100 insurance + 100 user capital)
2. Admin resolves market, force-closes all positions
3. Admin calls WithdrawInsurance → transfers 100 tokens out, but engine.vault still = 200
4. Users can still close accounts withdrawing 100 more tokens (engine thinks vault has 200)
5. But actual token balance is only 100 → last users get bricked

**Proposed Fix:** Add `engine.vault = engine.vault - insurance_units;` before transferring tokens in `WithdrawInsurance`.

---

## High Findings

### H1: Resolved Market Crank Doesn't Update LP Aggregates or OI

**File:** `program/src/percolator.rs`, lines ~3440-3480  
**Function:** `process_instruction` → `KeeperCrank` when `is_resolved`  
**Severity:** High

**Description:** The resolved-market crank path clears `position_size` to zero but doesn't update `total_open_interest`, `net_lp_pos`, `lp_sum_abs`, or `lp_max_abs`. This corrupts the engine's aggregate state, potentially affecting:
- Risk gating (uses LP aggregates)
- Funding rate calculations (uses net_lp_pos)
- Conservation checks

**Proposed Fix:** Use `oracle_close_position_core()` which properly maintains all aggregates, or manually update aggregates in the resolved-market crank path.

---

### H2: Resolved Market Crank Doesn't Settle Funding Before Closing

**File:** `program/src/percolator.rs`, lines ~3440-3480  
**Function:** `process_instruction` → `KeeperCrank` when `is_resolved`  
**Severity:** High

**Description:** The resolved-market crank path skips `touch_account()` (funding settlement) and `settle_mark_to_oracle()` before closing positions. Accounts with unsettled funding will have incorrect PnL, leading to wrong settlement amounts. The funding index delta could be substantial if the market has been running with high funding rates.

**Proposed Fix:** Call `touch_account()` and settle funding before computing settlement PnL in the resolved-market path.

---

### H3: I128 Neg Operator Can Overflow for i128::MIN

**File:** `percolator/src/i128.rs`, BPF version `Neg` impl  
**Severity:** High

**Description:** The `Neg` implementation for the BPF (non-Kani) I128:
```rust
impl core::ops::Neg for I128 {
    fn neg(self) -> Self {
        Self::new(-self.get())  // PANICS on i128::MIN in debug, wraps in release
    }
}
```
For `i128::MIN`, `-i128::MIN` overflows. In release mode this wraps to `i128::MIN` (no-op), in debug mode it panics. The Kani version uses `saturating_neg()` which is correct.

While `execute_trade` validates `size != i128::MIN`, this could be triggered through other paths where I128 values are negated (e.g., LP position updates via `net_lp_pos = net_lp_pos - pos`).

**Proof of Concept:** If an LP somehow has `position_size = I128::MIN` (through accumulation of many trades), any operation that negates this position would behave incorrectly.

**Proposed Fix:** Use `Self::new(self.get().saturating_neg())` in the BPF `Neg` impl, matching the Kani version.

---

### H4: Hyperp Mode Mark Price Manipulation via TradeCpi

**File:** `program/src/percolator.rs`, lines ~4070-4080  
**Function:** `process_instruction` → `TradeCpi` (Hyperp mark update)  
**Severity:** High

**Description:** In Hyperp mode, after a TradeCpi, the mark price is updated to the clamped execution price:
```rust
let clamped_mark = oracle::clamp_oracle_price(
    config.last_effective_price_e6, ret.exec_price_e6, config.oracle_price_cap_e2bps,
);
config.authority_price_e6 = clamped_mark;
```
A malicious matcher can repeatedly return `exec_price` at the maximum cap deviation, grinding the mark price in any direction over multiple transactions. With default cap of 1% per slot and multiple TradeCpi calls per slot, the mark price can be moved significantly.

The mark price is clamped against `last_effective_price_e6` (the **index**), not the previous mark. But the index itself moves toward the mark via `clamp_toward_with_dt`. This creates a feedback loop: manipulated mark → index follows → larger mark manipulation possible.

**Proposed Fix:** 
- Rate-limit mark price updates per slot (only first trade per slot should update mark)
- Or clamp against previous mark price rather than index
- Add a maximum deviation between mark and index

---

### H5: Admin Can Drain User Funds via ResolveMarket + WithdrawInsurance

**File:** `program/src/percolator.rs`, various admin instructions  
**Severity:** High (centralization risk)

**Description:** The admin has unchecked power to:
1. `SetOracleAuthority` → set themselves as oracle authority
2. `PushOraclePrice` → push manipulated price
3. `ResolveMarket` → force-resolve at manipulated price
4. `WithdrawInsurance` → drain insurance fund

Combined with C3 (vault not decremented), this can drain all user funds. While admin centralization is a known trust assumption, the lack of timelocks or multisig enforcement makes this trivially exploitable by a compromised admin key.

**Proposed Fix:** 
- Add timelock for admin operations (especially ResolveMarket)
- Require multi-step admin operations with delay
- Consider RenounceAdmin for trustless markets

---

### H6: Hardcoded Program IDs

**File:** `program/src/percolator.rs`  
**Severity:** High

**Description:** Program IDs for SPL Token, System Program, and DEX programs (PumpSwap, Raydium, Meteora) are hardcoded as byte arrays rather than using Solana's `declare_id!` or runtime checks against `spl_token::id()`. If Solana upgrades or forks any of these programs, the hardcoded IDs become stale and the program silently interacts with the wrong (or non-existent) program — potentially leading to fund loss or bricked markets.

**Proposed Fix:** Use canonical program ID constants from the respective crate (`spl_token::id()`, `system_program::id()`) or at minimum validate passed-in program accounts against known IDs at runtime.

---

### H7: Hardcoded Rent Exemption Values

**File:** `program/src/percolator.rs`  
**Severity:** High

**Description:** Rent exemption minimums are hardcoded as constants rather than queried from the `Rent` sysvar at runtime. If Solana adjusts rent parameters (which has happened historically), accounts may be created with insufficient lamports, making them eligible for rent collection and eventual deletion — destroying user positions and market state.

**Proposed Fix:** Use `Rent::get()?.minimum_balance(data_len)` at runtime instead of hardcoded values. Pass the Rent sysvar as an account where needed.

---

### H8: O(N) RPC Calls Per Market — DoS and Scalability Risk

**File:** Frontend / client SDK  
**Severity:** High

**Description:** The client fetches market data with O(N) individual RPC calls per market (one per account/position). For markets with many participants, this creates:
- **DoS vector:** An attacker can create many small accounts to bloat RPC load
- **Rate limiting:** Clients hit RPC rate limits, making the UI unusable
- **Scalability ceiling:** Markets with 1000+ participants become impractical to load

**Proposed Fix:** 
- Use `getMultipleAccounts` to batch reads
- Implement `getProgramAccounts` with filters for bulk fetching
- Add on-chain summary accounts (total OI, participant count) to reduce reads
- Consider a caching/indexing layer (e.g., Geyser plugin)

---

## Medium Findings

### M1: Missing `require_not_paused` Check in InitLP

**File:** `program/src/percolator.rs`, lines ~3280  
**Function:** `process_instruction` → `InitLP`  
**Severity:** Medium

**Description:** `InitUser` checks `require_not_paused(&data)?` but `InitLP` does not. A new LP can be created while the market is paused, which could be used to front-run an unpause.

**Proposed Fix:** Add `require_not_paused(&data)?` to the `InitLP` handler.

---

### M2: LiquidateAtOracle Missing Owner Check — Permissionless

**File:** `program/src/percolator.rs`, lines ~4020-4090  
**Function:** `process_instruction` → `LiquidateAtOracle`  
**Severity:** Medium (by design, but risky)

**Description:** `LiquidateAtOracle` requires no signer check — any account can call it. While permissionless liquidation is standard in DeFi, the instruction doesn't reward the liquidator. This means:
- No MEV incentive for timely liquidation (relies solely on keeper crank)
- Griefing: anyone can liquidate accounts that are barely below maintenance margin

This is likely by design but worth noting compared to protocols that provide liquidation rewards.

---

### M3: TopUpInsurance Missing Signer Validation on Source Token Account

**File:** `program/src/percolator.rs`, lines ~4150-4195  
**Function:** `process_instruction` → `TopUpInsurance`  
**Severity:** Medium

**Description:** `TopUpInsurance` validates `a_user` is a signer and verifies the token account owner matches `a_user.key`, but `verify_token_account` is skipped in test mode. In production, the SPL token `transfer` instruction itself validates authority, so this is protected by the SPL token program. However, the pattern is inconsistent — other instructions explicitly verify the token account.

---

### M4: Funding Rate Self-Heal Silently Resets Rate

**File:** `percolator/src/percolator.rs`, lines ~2155-2165  
**Function:** `accrue_funding`  
**Severity:** Medium

**Description:** When `funding_rate.abs() > 10_000`, the engine silently resets the rate to 0 and skips accrual:
```rust
if funding_rate.abs() > 10_000 {
    self.funding_rate_bps_per_slot_last = 0;
    self.last_funding_slot = now_slot;
    return Ok(());
}
```
This "self-heal" means if the funding rate is somehow corrupted, funding for the entire elapsed period is simply skipped. Accounts that should have paid/received funding are silently given a pass. An attacker who can corrupt the funding rate (e.g., via the Hyperp authority_timestamp reinterpretation) gets free funding.

**Proposed Fix:** Log a warning event when self-heal triggers. Consider applying a clamped rate rather than zero.

---

### M5: `clamp_toward_with_dt` Multiplication Overflow in Hyperp Mode

**File:** `program/src/percolator.rs`, lines ~2540  
**Function:** `oracle::clamp_toward_with_dt`  
**Severity:** Medium

**Description:** 
```rust
let max_delta_u128 = (index as u128)
    .saturating_mul(cap_e2bps as u128)
    .saturating_mul(dt_slots as u128)
    / 1_000_000u128;
```
While using saturating_mul, if `index * cap_e2bps * dt_slots` saturates to `u128::MAX`, the division by 1M gives `u128::MAX / 1M` which is still enormous. This means after saturation, the clamp effectively allows unlimited price movement. With `index = u64::MAX`, `cap = 10_000`, `dt = 1000`: `u64::MAX * 10000 * 1000` overflows u128, saturating to MAX, and allowing arbitrary price jumps.

**Proposed Fix:** Reorder to `(index * cap / 1_000_000) * dt` or use checked math with explicit overflow handling.

---

## Low Findings

### L1: `ACCOUNT_IDX_MASK` Requires Power-of-Two MAX_ACCOUNTS

**File:** `percolator/src/percolator.rs`, line ~50  
**Severity:** Low

**Description:** `const ACCOUNT_IDX_MASK: usize = MAX_ACCOUNTS - 1;` is used for wrapping indices but only works correctly when `MAX_ACCOUNTS` is a power of 2. All current feature configurations use powers of 2 (4, 64, 256, 1024, 4096), but there's no compile-time assertion enforcing this.

**Proposed Fix:** Add `const _: () = assert!(MAX_ACCOUNTS.is_power_of_two());`

---

### L2: `old_lp_pos` Variables Shadow in `execute_trade`

**File:** `percolator/src/percolator.rs`, lines ~2850 and ~2950  
**Function:** `execute_trade`  
**Severity:** Low

**Description:** `old_lp_pos` is computed early for risk-increase check, then used again later for OI and aggregate updates. If the split_at_mut borrow reordering changed, the two uses could diverge. Currently correct but fragile.

---

### L3: Warmup Slope Minimum of 1 Allows Slow-Drip Withdrawal

**File:** `percolator/src/percolator.rs`, lines ~2100-2120  
**Function:** `update_warmup_slope`  
**Severity:** Low

**Description:** `slope = max(1, avail_gross / warmup_period)` ensures slope ≥ 1 when avail_gross > 0. With warmup_period = 500 slots, an account with 1 unit of PnL gets slope = 1, meaning it can withdraw 1 unit per slot. This is correct behavior but means very small PnL amounts bypass the warmup effectively (1 unit per slot ≈ instant for dust amounts).

---

### L4: Dust Accumulator Overflow

**File:** `program/src/percolator.rs`, multiple instructions  
**Severity:** Low

**Description:** Dust is accumulated via `old_dust.saturating_add(dust)`. If many deposits with small dust amounts occur over time, the u64 dust counter could saturate at `u64::MAX`, losing track of actual dust. In practice this requires ~18 quintillion atomic units of dust which is unrealistic.

---

## Informational

### I1: Hyperp `authority_timestamp` Field Reuse

The `authority_timestamp` field in `MarketConfig` is reinterpret-cast as the funding rate (i64 bps/slot) in Hyperp mode. This dual-purpose field works but is confusing and error-prone. The `PushOraclePrice` handler correctly avoids overwriting it in Hyperp mode, but future changes could accidentally break this.

### I2: Formal Verification Coverage

The Kani verification covers wrapper-level authorization, nonce advancement, ABI validation, and pure math helpers. However, the core economic logic (mark PnL, haircut ratio, liquidation close amounts) is NOT formally verified — only tested. The most critical bugs found (C2, C3) are in the instruction processing layer, outside Kani's coverage.

### I3: `unsafe_close` Feature Gate

The `unsafe_close` feature skips all `CloseSlab` validation. Compile-time guards prevent enabling it with `mainnet` feature, which is good. However, if a devnet/testnet build is accidentally deployed to mainnet without the `mainnet` feature flag, the guard doesn't protect.

---

## Summary Table

| ID | Severity | Title | Exploitable? |
|----|----------|-------|-------------|
| C1 | Critical | DEX Oracle Flash Loan Manipulation | Yes - fund drainage on DEX-oracle markets |
| C2 | Critical | ResolveMarket Wrong PnL Formula | Yes - incorrect settlement amounts |
| C3 | Critical | WithdrawInsurance Doesn't Reduce Vault | Yes - vault accounting corruption |
| H1 | High | Resolved Crank Missing Aggregate Updates | Yes - corrupted engine state |
| H2 | High | Resolved Crank Missing Funding Settlement | Yes - incorrect settlement |
| H3 | High | I128 Neg Overflow on MIN | Edge case - requires extreme position |
| H4 | High | Hyperp Mark Price Grinding | Yes - price manipulation via matcher |
| H5 | High | Admin Can Drain Funds | Yes - requires compromised admin |
| H6 | High | Hardcoded Program IDs | Yes - breaks on program upgrades |
| H7 | High | Hardcoded Rent Exemption | Yes - accounts can be reaped |
| H8 | High | O(N) RPC Calls Per Market | Yes - DoS / scalability ceiling |
| M1 | Medium | InitLP Missing Pause Check | Yes - LP creation during pause |
| M2 | Medium | Permissionless Liquidation No Reward | Design concern |
| M3 | Medium | TopUpInsurance Token Validation | Protected by SPL program |
| M4 | Medium | Funding Self-Heal Skips Accrual | Yes - funding evasion |
| M5 | Medium | Hyperp Price Clamp Overflow | Yes - at extreme values |
| L1 | Low | Power-of-Two Assumption | No compile-time check |
| L2 | Low | Variable Shadowing | Fragile code |
| L3 | Low | Warmup Dust Bypass | By design |
| L4 | Low | Dust Counter Saturation | Unrealistic scenario |
