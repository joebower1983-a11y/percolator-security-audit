# Bug Bounty Report — Percolator Launch Frontend Audit

**Auditor:** OpenClaw AI Agent  
**Date:** 2026-02-15  
**Scope:** `app/` directory — all pages, components, hooks, API routes, and lib files  

---

## Critical Bugs

### BUG-001: BigInt `.toLocaleString()` crashes in older browsers/SSR
- **Files:**
  - `app/components/trade/EngineHealthCard.tsx` lines 60-62
  - `app/components/trade/LiquidationAnalytics.tsx` lines 53-54
  - `app/components/trade/CrankHealthCard.tsx` lines 81-82
- **Severity:** Critical
- **Description:** `engine.currentSlot.toLocaleString()`, `engine.lifetimeLiquidations.toLocaleString()`, and `engine.lifetimeForceCloses.toLocaleString()` are called directly on BigInt values. `BigInt.prototype.toLocaleString()` is **not supported in Safari < 15.4 or any environment without full BigInt Intl support**. This will throw `TypeError: Cannot convert a BigInt value to a string with locale` and crash the entire trade page component tree.
- **CrankHealthCard** at lines 99 and 107 correctly wraps with `Number()` first, but EngineHealthCard and LiquidationAnalytics do not.
- **Proposed fix:** Replace all `bigintValue.toLocaleString()` with `Number(bigintValue).toLocaleString()` or use the existing `formatTokenAmount()` utility. Example:
  ```ts
  // Before:
  { label: "Current Slot", value: engine.currentSlot.toLocaleString() }
  // After:
  { label: "Current Slot", value: Number(engine.currentSlot).toLocaleString() }
  ```

### BUG-002: Hardcoded funding rate display — misleading financial data
- **File:** `app/components/trade/PositionPanel.tsx` line 259
- **Severity:** Critical
- **Description:** The "Est. Funding (24h)" row shows a hardcoded `+$5.12` string with a comment `/* Mock value - will be replaced with real funding calculation */`. This is rendered for **all positions in production**, not just mock mode. Users see fake PnL data, which could influence trading decisions. This is essentially displaying fabricated financial data to users making leveraged trades.
- **Proposed fix:** Either compute real funding using `engine.fundingRateBpsPerSlotLast` and position size, or hide this row entirely until the calculation is implemented:
  ```tsx
  // Option 1: Hide until implemented
  // Remove the "Est. Funding (24h)" row entirely
  
  // Option 2: Compute from engine state
  const fundingRate = engine?.fundingRateBpsPerSlotLast ?? 0n;
  const slotsPerDay = BigInt(Math.round(24 * 60 * 60 / 0.4));
  const dailyFunding = (absPosition * fundingRate * slotsPerDay) / 10000n;
  ```

### BUG-003: Helius API key exposed in client bundle
- **File:** `app/lib/config.ts` lines 1-5, 18, 25
- **Severity:** Critical (for mainnet)
- **Description:** The `NEXT_PUBLIC_HELIUS_API_KEY` is embedded in the client-side JavaScript bundle via both devnet and mainnet RPC URL construction. The code has a security comment acknowledging this is acceptable for devnet but must be replaced before mainnet. Since the config already includes a `mainnet` block with this key, this is a ticking time bomb. Anyone can extract the key from the bundle and abuse the RPC quota.
- **Also in:** `app/app/devnet-mint/devnet-mint-content.tsx` line 30 — key is directly interpolated into a string constant.
- **Proposed fix:** Create an `/api/rpc` proxy route that keeps the key server-side. For mainnet config:
  ```ts
  mainnet: {
    rpcUrl: "/api/rpc", // Server-side proxy
    // ...
  }
  ```

---

## High Bugs

### BUG-004: Simulation page hardcodes program IDs instead of using config
- **File:** `app/app/simulation/page.tsx` lines ~40-43
- **Severity:** High
- **Description:** The simulation page hardcodes `PROGRAM_ID` and `MATCHER_PROGRAM_ID` as constants instead of reading from `getConfig()`. If program IDs change or the user is on a different network, the simulation will target the wrong programs, creating invalid transactions.
- **Proposed fix:** Replace hardcoded constants with `getConfig().programId` and `getConfig().matcherProgramId`.

### BUG-005: Simulation page hardcodes rent values
- **File:** `app/app/simulation/page.tsx` lines ~50-52
- **Severity:** High
- **Description:** `MINT_RENT = 1_461_600`, `SLAB_RENT = 438_034_560`, `MATCHER_CTX_RENT = 3_118_080` are hardcoded. These values change with Solana's rent calculation. Using stale values will cause `CreateAccount` instructions to fail if rent increases, or waste SOL if it decreases.
- **Proposed fix:** Use `connection.getMinimumBalanceForRentExemption(dataSize)` to calculate rent dynamically.

### BUG-006: `usePortfolio` fetches ALL slabs for ALL markets — O(N) RPC calls
- **File:** `app/hooks/usePortfolio.ts` lines 56-72
- **Severity:** High (Performance)
- **Description:** For every market discovered on-chain, the hook calls `fetchSlab(connection, market.slabAddress)` in a serial loop. Each slab can be ~1MB. With 50+ markets, this downloads 50+MB of data and makes 50+ RPC calls sequentially. This will cause the portfolio page to:
  1. Take 30+ seconds to load
  2. Hit RPC rate limits
  3. Potentially time out
- **Proposed fix:** Use `connection.getMultipleAccountsInfo()` to batch fetch (max 100 per call). Or better, query Supabase for the user's positions instead of scanning all slabs:
  ```ts
  // Batch fetch instead of serial
  const infos = await connection.getMultipleAccountsInfo(
    markets.map(m => m.slabAddress)
  );
  ```

### BUG-007: Market discovery grid columns mismatch — hidden data on mobile
- **File:** `app/app/markets/page.tsx` — market row grid definition
- **Severity:** High (UI)
- **Description:** The header row defines 7 columns for desktop (`token | price | OI | volume | insurance | max lev | health`) but the `sm:` breakpoint hides "insurance" and "max lev" with `hidden sm:block`. However, the data rows use the same 7-column grid at all breakpoints, so on mobile the grid has 5 visible columns but 7 cells — causing "insurance" and "max lev" data to still render (overflowing or misaligned) while their headers are hidden.
- **Proposed fix:** Add matching `hidden sm:block` classes to the insurance and max leverage data cells in the row component.

### BUG-008: `InsuranceDashboard` silently falls back to mock data on API error in production
- **File:** `app/components/market/InsuranceDashboard.tsx` line 114
- **Severity:** High
- **Description:** When the `/api/insurance/${slabAddress}` fetch fails, the catch block sets `setInsuranceData(MOCK_INSURANCE)` regardless of environment. This means production users see fabricated insurance fund data ($125,432 balance, 8.5x coverage) when the API is down, which is dangerously misleading for a financial application.
- **Proposed fix:** Only fall back to mock data when `isMockMode()` is true:
  ```ts
  } catch (err) {
    setError(err instanceof Error ? err.message : "Unknown error");
    if (isMockMode()) setInsuranceData(MOCK_INSURANCE);
    // else leave insuranceData null — the "No data available" state handles it
  }
  ```

---

## Medium Bugs

### BUG-009: `useTrade` creates AbortController but never aborts it
- **File:** `app/hooks/useTrade.ts` lines 21-23
- **Severity:** Medium
- **Description:** An `AbortController` and `cancelled` flag are created but `abortController.abort()` is never called. The commented-out matcher context validation was the only consumer. The `cancelled` flag is checked at line 30 but never set to `true`. This is dead code that signals incomplete cleanup logic.
- **Proposed fix:** Either remove the dead AbortController code, or wire it up properly with a cleanup return from the trade callback.

### BUG-010: `FundingRateChart` falls back to mock data in production
- **File:** `app/components/trade/FundingRateChart.tsx` line 58
- **Severity:** Medium
- **Description:** Comment at line 14 says "Generate mock 24h data" and line 58 indicates fallback to mock. Same pattern as BUG-008 — real users may see fabricated funding rate charts.
- **Proposed fix:** Only use mock data when `isMockMode()` is true.

### BUG-011: `SlabProvider` dependency on `publicKey?.toBase58()` causes unnecessary refetch
- **File:** `app/components/providers/SlabProvider.tsx` — useEffect dependency array
- **Severity:** Medium (Performance)
- **Description:** The main data-fetching useEffect depends on `publicKey?.toBase58()`. Every time the wallet adapter re-renders (which happens frequently), a new string is created, and string comparison is referentially new even if the value is the same. This triggers a full re-subscribe + re-fetch cycle. Since the slab data is market-wide (not user-specific), the wallet public key shouldn't be in the dependency array at all.
- **Proposed fix:** Remove `publicKey?.toBase58()` from the dependency array. The wallet key is not needed to fetch slab data. User-specific filtering happens downstream in `useUserAccount`.

### BUG-012: `create` page uses `require()` at runtime for PublicKey validation
- **File:** `app/app/create/page.tsx` line 19
- **Severity:** Medium
- **Description:** `const { PublicKey } = require("@solana/web3.js");` is used inside a client component. While this works, it's:
  1. A CommonJS `require()` in an ESM context — may break with stricter bundler settings
  2. Creates a second import of @solana/web3.js (the module is already imported elsewhere)
  3. Defeats tree-shaking
- **Proposed fix:** Use the existing import at the top of the file:
  ```tsx
  import { PublicKey } from "@solana/web3.js";
  // Then in the validation:
  try { new PublicKey(mintParam); initialMint = mintParam; } catch {}
  ```

### BUG-013: Rate limiter in middleware uses in-memory Map — no protection in serverless
- **File:** `app/middleware.ts` lines 5-7
- **Severity:** Medium (Security)
- **Description:** The rate limiter uses a plain `Map()` that lives in process memory. In serverless deployments (Vercel), each invocation may get a different instance, making the rate limit ineffective. The Map also grows unbounded (cleanup only happens with 0.1% probability per request at line 15).
- **Proposed fix:** Use a Redis-backed rate limiter (e.g., `@upstash/ratelimit`) or at minimum, add a size cap to the Map to prevent memory leaks:
  ```ts
  if (rateLimitMap.size > 10_000) rateLimitMap.clear();
  ```

### BUG-014: `TradeHistory` size formatting loses precision for large/decimal values
- **File:** `app/components/trade/TradeHistory.tsx` — `toBigInt` function
- **Severity:** Medium
- **Description:** The `toBigInt` function at line 25 uses `BigInt(val.split(".")[0])` for string values, discarding all decimal places. Then it calls `Math.abs()` on a potentially unsafe integer (`Number.isSafeInteger` check is only for number type). For trade sizes stored as e.g. `"1234567.890123"`, the display will show `1.234567` instead of `1.234567890123`.
- **Proposed fix:** Use the existing `parseHumanAmount()` utility or handle decimals properly:
  ```ts
  function toBigInt(val: number | string | bigint): bigint {
    if (typeof val === "bigint") return val;
    if (typeof val === "string") return parseHumanAmount(val, 6);
    return BigInt(Math.round(val));
  }
  ```

### BUG-015: Home page Supabase query has no error state for users
- **File:** `app/app/page.tsx` — `loadStats` function around line 100
- **Severity:** Medium
- **Description:** If Supabase is down or env vars aren't set, `loadStats` catches the error and logs to console, but leaves `stats` at `{ markets: 0, volume: 0, insurance: 0 }`. The `hasStats` check at line 116 means the stats section silently disappears. Users see no indication that data failed to load. The featured markets section also silently disappears.
- **Proposed fix:** Add an error state that shows a "Failed to load stats" indicator, or at minimum show the stats section with "—" values.

---

## Low Bugs

### BUG-016: `PositionPanel` margin health calculation is imprecise
- **File:** `app/components/trade/PositionPanel.tsx` — `marginHealthStr` calculation
- **Severity:** Low
- **Description:** Margin health is calculated as `(capital * 100n) / absPosition` — this is capital-to-position ratio, not actual margin health. True margin health should account for unrealized PnL: `(capital + unrealizedPnl) / (absPosition * maintenanceMarginBps / 10000)`. The current calculation could show "healthy" margin when a user is actually near liquidation.
- **Proposed fix:**
  ```ts
  const effectiveMargin = account.capital + pnlTokens;
  const requiredMargin = (absPosition * maintenanceBps) / 10000n;
  const healthPct = requiredMargin > 0n ? Number((effectiveMargin * 100n) / requiredMargin) : 0;
  marginHealthStr = `${healthPct.toFixed(1)}%`;
  ```

### BUG-017: `InsuranceLPPanel` uses `type="number"` input — allows `e`, `+`, `-` characters
- **File:** `app/components/trade/InsuranceLPPanel.tsx` line 198
- **Severity:** Low
- **Description:** The HTML `<input type="number">` allows scientific notation (`1e5`), negative values (`-100`), and `+` prefix. The `parseHumanAmount` function may not handle these gracefully. Other inputs in the codebase (TradeForm, DepositWithdrawCard) correctly use `type="text"` with regex filtering.
- **Proposed fix:** Change to `type="text"` with the same `onChange` filter pattern:
  ```tsx
  <input
    type="text"
    onChange={(e) => setAmount(e.target.value.replace(/[^0-9.]/g, ""))}
  ```

### BUG-018: `my-markets` page Summary Stats show TVL with misleading `$` prefix
- **File:** `app/app/my-markets/page.tsx` — Summary Stats Bar
- **Severity:** Low
- **Description:** TVL and Insurance values are formatted as `"$" + fmt(totalVault)` where `fmt()` divides by `10^6` and formats as a number. But the values are in **token units**, not USD. Displaying a `$` prefix when the value is in SOL, WIF, etc. tokens is misleading. The `fmt` function at line 8 does `Number(v) / 10 ** decimals` which converts from raw to human-readable token amounts, not USD.
- **Proposed fix:** Remove the `$` prefix or multiply by the token's USD price first.

### BUG-019: `devnet-mint-content.tsx` missing exhaustive deps in `useEffect`
- **File:** `app/app/devnet-mint/devnet-mint-content.tsx` line 87
- **Severity:** Low
- **Description:** The `useEffect` that sets `recipient` from `publicKey` has `[publicKey]` in deps but `recipient` is also read inside. The eslint-disable comment acknowledges this. Additionally, `refreshBalance` at line 96 depends on `connection` but the eslint-disable suppresses the warning. These could cause stale closures.
- **Proposed fix:** Add the missing deps or restructure to avoid the stale closure.

### BUG-020: CSP allows `unsafe-eval` and `unsafe-inline` for scripts
- **File:** `app/middleware.ts` — `addSecurityHeaders` function
- **Severity:** Low (Security)
- **Description:** The Content-Security-Policy includes `'unsafe-eval'` and `'unsafe-inline'` for script-src. While Next.js may require `unsafe-eval` in dev mode, this should be tightened for production. `unsafe-inline` defeats most XSS protections that CSP provides.
- **Proposed fix:** Use nonce-based CSP for inline scripts and remove `unsafe-eval` in production builds. Next.js 14 supports `nonce` via `next.config.js` headers.

### BUG-021: `useTrade` abortController `cancelled` flag is never set
- **File:** `app/hooks/useTrade.ts` line 22
- **Severity:** Low
- **Description:** The `cancelled` variable is declared but never set to `true`. If the component unmounts while a trade is in flight, the `setError` and `setLoading` calls in the finally block will attempt to update unmounted component state (React warning). The check at line 30 `if (cancelled) return;` is unreachable.
- **Proposed fix:** Return a cleanup function from the trade callback, or use a ref to track mount state.

---

## Summary

| Severity | Count |
|----------|-------|
| Critical | 3     |
| High     | 5     |
| Medium   | 7     |
| Low      | 6     |
| **Total** | **21** |

### Top Priority Fixes:
1. **BUG-002** — Remove hardcoded `+$5.12` funding display (users see fake financial data)
2. **BUG-001** — Fix BigInt `.toLocaleString()` crashes (breaks trade page in Safari)
3. **BUG-008** — Stop showing mock insurance data in production on API failure
4. **BUG-003** — Proxy the Helius API key before mainnet (known, but still needs action)
5. **BUG-006** — Portfolio page O(N) slab fetches will not scale
