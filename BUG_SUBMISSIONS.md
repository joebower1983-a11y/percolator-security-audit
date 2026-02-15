# Percolator Bug Bounty Submissions

**Twitter:** @JBower1
**Bounty Wallet:** Cz6wzo71wcpGT3twb3mKM9q4FvFcZUBhm14nn8JNZhta

---

## Bug #1: WithdrawInsurance Doesn't Reduce Vault Balance (Fund Drainage)

**Severity:** Critical
**Page:** Other (on-chain program)
**Title:** WithdrawInsurance never decrements engine.vault — breaks conservation invariant

**Description:**
The `WithdrawInsurance` instruction zeroes `engine.insurance_fund.balance` and transfers tokens from the vault, but never decrements `engine.vault`. This violates the conservation invariant V >= C_tot + I.

After withdrawal, the engine believes it has more vault balance than actually exists in the token account. If the vault holds both insurance and user capital, the insurance withdrawal drains real tokens but the engine still counts them — meaning subsequent user withdrawals can either drain other users' funds or brick the market when the token account runs dry.

In `program/src/percolator.rs` around the WithdrawInsurance handler (~line 4410-4490), after `engine.insurance_fund.balance` is zeroed and tokens transferred, there's no corresponding `engine.vault = engine.vault - insurance_units;`

**Steps to Reproduce:**
1. Create a market with insurance fund topped up (e.g., 100 units)
2. Have users deposit capital (e.g., 100 units) — vault = 200 tokens total
3. Resolve market, force-close all positions via crank
4. Call WithdrawInsurance — 100 tokens transfer out, but engine.vault still = 200
5. Users close accounts — engine allows withdrawal of 200 total but only 100 tokens remain

**Expected Behavior:** engine.vault should be decremented by insurance_units before the token transfer, maintaining the conservation invariant.

**Actual Behavior:** engine.vault is unchanged after WithdrawInsurance, creating a mismatch between on-chain accounting and actual token balance.

---

## Bug #2: DEX Oracle Flash Loan Price Manipulation

**Severity:** Critical
**Page:** Markets / Trade
**Title:** DEX oracle readers (PumpSwap/Raydium/Meteora) have zero TWAP protection — flash loan exploitable

**Description:**
DEX-based oracle readers (`read_pumpswap_price_e6`, `read_raydium_clmm_price_e6`, `read_meteora_dlmm_price_e6`) read spot prices directly from AMM pool reserves with no staleness checks and no TWAP protection. The code comments even acknowledge this: "vulnerable to flash-loan manipulation."

The circuit breaker (`oracle_price_cap_e2bps`) is 0 (disabled) by default for non-Hyperp markets. Even when enabled at 1% per slot (10,000 e2bps), an attacker can grind the price ~100% in 100 slots (~50 seconds).

In `program/src/percolator.rs` around lines 2101-2200 (PumpSwap) and 1800-1900 (Raydium CLMM).

**Steps to Reproduce:**
1. Create a market using a PumpSwap pool as its oracle (index_feed_id = pool address)
2. In a single transaction: flash-loan large amount → manipulate pool reserves to spike price → open maximum leveraged long at inflated price → restore pool reserves → in next tx close position at real price → pocket difference
3. Alternatively, grind the price over multiple slots if circuit breaker is enabled (1% per slot compounds fast)

**Expected Behavior:** Oracle prices should use TWAP or multi-block aggregation to resist single-block manipulation. DEX oracle markets should enforce a minimum circuit breaker cap.

**Actual Behavior:** Spot pool reserves are read directly with no time-weighting, enabling flash loan manipulation for instant profit extraction from the vault.

---

## Bug #3: ResolveMarket PnL Uses Wrong Formula (Incorrect Settlement)

**Severity:** Critical
**Page:** Other (on-chain program)
**Title:** Resolved market crank uses linear PnL formula instead of coin-margined formula — incorrect payouts

**Description:**
When a market is resolved and the KeeperCrank force-closes positions, the PnL calculation uses:
```
pnl_delta = pos * (settle - entry) / 1_000_000
```
This is a linear PnL formula. However, the risk engine's `mark_pnl_for_position` uses a coin-margined formula: `(oracle - entry) * abs_pos / oracle`. These produce very different values, especially when oracle price diverges significantly from 1e6.

For example, with entry=1e6, settle=2e6, pos=1000:
- Linear formula: 1000 * (2e6 - 1e6) / 1e6 = 1000
- Coin-margined: (2e6 - 1e6) * 1000 / 2e6 = 500

The linear formula overpays longs by 2x in this case, potentially draining the vault.

In `program/src/percolator.rs` around lines 3440-3480 in the KeeperCrank resolved-market path.

**Steps to Reproduce:**
1. Create a coin-margined perp market
2. Users open long positions at various entry prices
3. Admin resolves market at a price significantly different from entry prices
4. Crank force-closes positions
5. Compare payouts to what `mark_pnl_for_position` would have calculated

**Expected Behavior:** Settlement should use the same coin-margined PnL formula as the risk engine (`oracle_close_position_core` or `mark_pnl_for_position`).

**Actual Behavior:** Linear formula is used, producing incorrect settlement amounts that can over/under-pay users and potentially drain the vault.

---

## Bug #4: Hardcoded Program IDs

**Severity:** High
**Page:** Other (on-chain program)
**Title:** Program IDs hardcoded as byte arrays instead of using canonical constants

**Description:**
SPL Token, System Program, and DEX program IDs (PumpSwap, Raydium, Meteora) are hardcoded as raw byte arrays in `program/src/percolator.rs` rather than using `spl_token::id()`, `system_program::id()`, or runtime validation against passed-in accounts.

If Solana upgrades or forks any of these programs (which has happened — e.g., Token-2022 migration), the hardcoded IDs become stale. The program would silently interact with the wrong or non-existent program, potentially leading to fund loss or bricked markets that can't process withdrawals.

**Steps to Reproduce:**
1. Search codebase for hardcoded program ID byte arrays
2. Compare against canonical program addresses
3. Note absence of runtime `key == expected_program_id` checks on passed program accounts

**Expected Behavior:** Use canonical constants (`spl_token::id()`) or validate passed-in program accounts against known IDs at runtime.

**Actual Behavior:** Raw byte arrays that will silently break on any program upgrade/fork.

---

## Bug #5: Hardcoded Rent Exemption Values

**Severity:** High
**Page:** Other (on-chain program)
**Title:** Rent exemption minimums are hardcoded instead of queried from Rent sysvar

**Description:**
Rent exemption minimums are hardcoded as constants rather than queried from the `Rent` sysvar at runtime via `Rent::get()?.minimum_balance(data_len)`. 

Solana has adjusted rent parameters historically. If this happens again, accounts created with hardcoded (now-insufficient) lamports become eligible for rent collection. This could destroy user position accounts and market state data — effectively rugging users whose positions get reaped.

**Steps to Reproduce:**
1. Search for hardcoded lamport constants used in account creation
2. Compare to what `Rent::get()?.minimum_balance()` would return
3. Note the Rent sysvar is not passed as an account in creation instructions

**Expected Behavior:** Query `Rent::get()?.minimum_balance(data_len)` at runtime for all account creation.

**Actual Behavior:** Hardcoded values that may become insufficient if Solana adjusts rent parameters.

---

## Bug #6: O(N) RPC Calls Per Market — DoS Vector

**Severity:** High
**Page:** Markets
**Title:** Client makes O(N) individual RPC calls per market participant — DoS and scalability risk

**Description:**
The frontend/client SDK fetches market data using one RPC call per account/position. For markets with many participants this creates a DoS vector: an attacker can create many small accounts to bloat RPC load, hitting rate limits and making the UI unusable for all users.

This also creates a hard scalability ceiling — markets with 1000+ participants become impractical to load, degrading UX as the protocol grows.

**Steps to Reproduce:**
1. Open a market page with 50+ participants
2. Monitor network tab — observe individual `getAccountInfo` calls for each position
3. Create 100+ small dust accounts on a market
4. Observe loading time and potential RPC rate limit errors

**Expected Behavior:** Use `getMultipleAccounts` for batching, `getProgramAccounts` with filters for bulk fetching, or on-chain summary accounts to reduce reads.

**Actual Behavior:** O(N) individual RPC calls that degrade linearly with participant count and are trivially exploitable for DoS.
