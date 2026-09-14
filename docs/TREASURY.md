# Phase 2 — The treasury and the buy wall

**Status: design. Nothing below is deployed.** Phase 1 (live) uses only the platform's Buyback & Lock mode. This document specifies what Phase 2 adds, what it reuses from the platform, and what is still open. It ships only after Arc mainnet, an audit of these contracts, and a real tokenized stock to hold.

## Goal

Turn $WALL's fee flow into (1) a growing, never-sold pile of NVDA and (2) a bid that is always under the price. The system is **oracle-free**: "book value" is computed from token balances, and the floor is defended by buying on the curve/pool at market — the trading price is the only price.

## Revision 2026-09-13: hybrid wall (taker before graduation, maker after)

Studying BTC Stock Protocol's *DefenseNet* (the sibling stock-treasury design on Robinhood
Chain) changed one thing in this spec: **after graduation the wall is a maker-side ladder, not a
taker-side buyback.** A market buy fills into the treasury's own wick and can be sandwiched; a
ladder of single-sided quote-token ranges below the price is filled *by* sellers, at prices the
treasury chose. Before graduation there is no order book (the bonding curve only takes market
orders), so `defend()` as specified in §3 stays the pre-graduation mechanism.

| | Before graduation (curve) | After graduation (V4 pool) |
| --- | --- | --- |
| Mechanism | `defend()` market-buys on the curve when `spot < bookValue × (1 + margin)` | seven single-sided NVDA ranges below an anchor, at −5 / −10 / −15 / −20 / −30 / −40 / −50% |
| Who executes | keeper (anyone), bounded by `epochBudget` | keeper `poke()` re-anchors, migrates surviving rungs, funds new ones; `harvest()` pulls fully crossed rungs |
| Price reference | book value (oracle-free) | anchor = high-water ratchet of the pool tick: `max(sealedTick, twapTick)`, moves only toward higher value, at most ×1.25 per beat, never follows a dump |
| Deepest rung | n/a | never above book value: the −50% rung sits at `min(anchor − 50%, bookValue)`, so the ladder always ends at or below the fundamental floor |
| Sizing | `min(epochBudget, amountToRestoreFloor)` | coverage κ = deepest-layer NVDA × 2 × anchorPrice ÷ circulating; κ < 1 → 100% to the deep layer, else 40% shallow / 60% mid; deep layer clamped to [40%, 80%] of funds |
| On fill | bought $WALL split burn / stream (§4) | $WALL recovered from a crossed rung is 100% burned on `harvest()`; freed NVDA re-stands in the unfilled part of the rung or returns to the pending pot |
| Bounty | `keeperBountyBps` of the buy | fixed unit bounty paid **last** in `poke()`, from an escrow capped at 30× the bounty and fed by ≤ 2% of fee inflow; sized to Arc gas with margin (BSP's covered 28% of gas and its keepers stalled) |

Rules carried over from DefenseNet, adopted as written:

- **First-wall gate.** The first `poke()` reverts unless the anchor is ready and spot is still
  above the −5% rung, and it must fund a non-zero position or the whole call reverts. No bounty
  for an empty wall.
- **Conservation.** `totalFeesIn == unfilled + converted + pending + keeperEscrow + keeperPaid +
  streamed` is an on-chain view; the site shows it and the ledger page reconciles against it.
- **Never a taker after graduation.** The treasury never initiates a swap in the pool; it only
  adds and removes liquidity.
- **Generations.** A full crossing of the deepest rung ends the generation: re-anchor at the
  sealed spot, new positions under a new salt. Two consecutive all-empty underwater beats
  trigger the same re-base (stall escape).

What we deliberately do differently:

- **Book value floors the ladder.** BSP has no fundamental anchor. Ours is on-chain
  (`treasuryNVDA ÷ circulating`) and caps how deep the wall can sit.
- **No emission.** BSP funds its wall from the fee of a trade-to-mine token. Ours is funded by
  the creator-fee share of an ordinary Radian launch (§1), so the wall never depends on a
  halving schedule.
- **Keeper bounty sized to gas**, re-settable by the owner within bounds, not immutable.
- **Maintenance must not depend on discarded accounting.** BSP's audit found that a public
  `donate()` could block `poke()` by polluting a ledger that ignored `feesAccrued`. Our
  conservation check keeps pool-side fee accruals in their own bucket.

Open, to argue over: rung widths on Arc's V4 tick spacing; whether a low-float token needs the
mid layer at all; whether `harvest()` streams part of the recovered NVDA to stakers or burns
everything it recovers.

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

### 3. Floor defense before graduation (taker; see the 2026-09-13 revision for the post-graduation ladder)
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
