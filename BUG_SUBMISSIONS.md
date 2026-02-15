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
