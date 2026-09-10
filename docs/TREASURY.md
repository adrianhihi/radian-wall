# Phase 2 — The treasury and the buy wall

**Status: design. Nothing below is deployed.** Phase 1 (live) uses only the platform's Buyback & Lock mode. This document specifies what Phase 2 adds, what it reuses from the platform, and what is still open. It ships only after Arc mainnet, an audit of these contracts, and a real tokenized stock to hold.

## Goal

Turn $WALL's fee flow into (1) a growing, never-sold pile of NVDA and (2) a bid that is always under the price. The system is **oracle-free**: "book value" is computed from token balances, and the floor is defended by buying on the curve/pool at market — the trading price is the only price.

## Actors

| Actor | Role |
| --- | --- |
| `WallTreasury` (new) | Holds NVDA. Claims fee flow. Executes floor buybacks. Disposes of bought-back $WALL (burn / stream). |
| `WallStaking` (new, ERC-20 reward variant of `RadianStaking`) | Stake $WALL, earn NVDA. |
| $WALL + its curve / V4 pool (platform, exists) | Where buybacks execute. |
| NVDA quote token (stand-in on Arc testnet; real tokenized NVDA on BSC) | The treasury asset. |
| Keeper (anyone) | Calls `defend()` / `flush()`; earns a small bounty. |

## Flows

### 1. Fee inflow → treasury
On Radian, a launch's **creator fee share** (and optional creator tax) accrues in `PonsV2FeeEscrow` to `creatorFeeRecipient`. Phase 2 relaunches $WALL (or migrates) with `creatorFeeRecipient = WallTreasury`, which periodically calls `claimToken(NVDA)` on the escrow. The platform's buyback share keeps doing what it does today (buy & lock in the platform vault) — the two mechanisms stack.

### 2. Book value
```
circulating = totalSupply − burned − platformVaultLocked
bookValue   = treasuryNVDA / circulating          // NVDA per $WALL
```
Both inputs are on-chain balances. Published as a view; the site shows it next to the market price.

### 3. Floor defense — the wall
`defend()` is callable by anyone when
```
marketPrice < bookValue × (1 + margin)
```
where `marketPrice` is the curve's spot (`quoteReserve / tokenReserve`) pre-graduation, or the V4 pool's spot after. It buys $WALL with NVDA up to `min(epochBudget, amountToRestoreFloor)`, with a hard slippage bound, then records the epoch. Result: a standing bid funded by shares, sized so the treasury can never be drained in one block.

### 4. Disposition — the flywheel
Bought-back $WALL is split (same shape as `RadianTreasury.flush`):
- **burn** `burnBps` — supply falls, `bookValue` per remaining token rises;
- **stream** the rest to `WallStaking.notifyReward` — but *as NVDA*, i.e. the treasury sells nothing; it streams a slice of *newly claimed* NVDA fee flow to stakers instead of buying with it. (Exact split between "spend on floor" and "stream to stakers" is a parameter; see below.)

## Reuse from the platform (already proven on-chain)

- `RadianTreasury.flush(minOut)` — buy on curve + burn + `notifyReward`; adapt to an ERC-20 quote and add the floor condition and epoch budget.
- `RadianStaking` — Synthetix-style, 7-day reward duration; swap native-USDC rewards for an ERC-20 (`SafeERC20`) reward token.
- `PonsV2FeeEscrow.claimToken` — the inflow path; no changes.

## Parameters (initial proposal — to be argued over)

| Param | Proposal | Why |
| --- | --- | --- |
| `margin` | 5% | Defend slightly above book so the wall is visible, not only at the cliff |
| `epochBudget` | 10% of treasury NVDA per 24h | Can't be drained in one dump |
| `burnBps` | 7000 (70%) | Bias to rising book value; 30% streams yield |
| `minInterval` | 1h between `defend()` calls | Rate-limit keeper spam |
| `maxSlippageBps` | 300 | Bounded MEV exposure per buy |
| `keeperBountyBps` | 10 of the buy | Someone always has a reason to call it |

## Security notes

- **Owner = multisig** before any real NVDA. Ownable2Step like the platform.
- **No NVDA withdrawal path.** The treasury can only spend NVDA through `defend()` → buy $WALL. No rescue function for the treasury asset. (Rescue only for unrelated tokens sent by mistake.)
- **Oracle-free** by construction; book value can't be manipulated without actually moving balances.
- **Buyback MEV:** bounded slippage + epoch budget + rate limit. Accept that keepers/searchers earn something; design so it's small.
- **Post-graduation path:** buying via the locked V4 pool goes through the platform's `PonsV2MemeHook`; verify hook fee handling on the buy path in tests.
- **Reentrancy:** `nonReentrant` on `defend()`/`flush()`; checks-effects-interactions.

## Open questions

1. **Which chain holds the real thing?** On Arc the treasury would hold the *stand-in* until a real tokenized stock exists there. A real-NVDA treasury means deploying on BSC (Ondo) — and verifying that specific token has no transfer hook that breaks the V4 pool.
2. **How big is the creator-fee share?** It sets the inflow rate and therefore how fast the wall grows. Platform default vs. an added creator tax is a product decision.
3. **Floor vs. yield split.** All to the wall = strongest floor, no yield; all to stakers = no floor. The 70/30 above is a starting point.
4. **Migration.** Relaunch $WALL under Phase 2 params, or keep the Phase 1 token and point a new `creatorFeeRecipient` at the treasury (the platform allows the creator to change the recipient — confirm and test).
5. **Legal.** A token whose treasury holds tokenized equity has jurisdictional constraints. Decide before mainnet.

## Non-goals

No oracle. No leverage. No promise of a price — only a transparent bid funded by shares that are never sold.
