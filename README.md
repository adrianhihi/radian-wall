# The Wall ($WALL) — Radian's flagship product

**A treasury that hoards NVDA. And only buys.**

The Wall is the first product built *on* [Radian](https://github.com/adrianhihi/radian) (the launchpad for Circle's Arc chain), the way BTCNVDA is the flagship product built on BSP Treasury. This repo is the **product**; the launchpad, protocol token and web app live in the **platform** repo.

| Repo | Role | Contains |
| --- | --- | --- |
| [`radian`](https://github.com/adrianhihi/radian) (platform) | The launchpad | Pons V2 contracts, $RADIAN fee flywheel, web app, indexer, deploy scripts |
| `radian-wall` (this, product) | One flagship token | Brand site, mechanism spec, deployment manifest — and, in Phase 2, the treasury contracts |

## What it is

$WALL is a token **priced in shares of Nvidia, not dollars**. It is launched on Radian's bonding curve paired against a tokenized-stock quote asset, in **Buyback & Lock** mode: the buyback share of every 1% trade fee buys $WALL back from the market and locks it in the platform's 5-year vault. That mechanic is live today with **zero new contracts** — see [`manifest.json`](manifest.json) for every address and the launch transaction.

Phase 2 (in design, see [`docs/TREASURY.md`](docs/TREASURY.md)) adds the actual wall: a treasury contract that accumulates NVDA from fee flow, defends a floor price by buying $WALL back with those shares, and burns or streams the bought-back tokens. It reuses the buyback-and-burn flywheel Radian already runs on-chain for its own token.

## Status — read this before anything else

- **Network:** Arc **testnet** (chain 5042002). No real money. Arc mainnet opens 2026-09-16.
- **Quote asset:** `NVDAx` is a clearly-labeled **testnet stand-in** for NVDA (a plain ERC-20 minted by us), because no real tokenized stock exists on Arc yet. Real tokenized stocks trade on BSC (Ondo) and Robinhood Chain; that is where a real-stock deployment goes.
- **Contracts:** Phase 1 uses only the platform's Pons V2 port (byte-for-byte match with Sourcify-verified Pons sources). Verification proves source = bytecode; **it is not an audit**. Phase 2 contracts do not exist yet and will be audited before holding funds.
- **Reference prices** on the site come from a delayed, keyless off-chain feed via the platform's `/api/stock-price` route. **No contract reads them.** The curve's own trading price is the price.

## Site

[`site/index.html`](site/index.html) is a single static page — no framework, no build. It reads live state from three public sources: the curve's `getReserves()` over JSON-RPC (spot price in NVDA), the Radian indexer (`/token/:addr`: curve progress, buyback-locked amount, trades), and the platform's reference-price route.

```sh
# preview locally
python3 -m http.server 8080 --directory site
# → http://localhost:8080
```

Deploy `site/` to any static host (Vercel, Cloudflare Pages, GitHub Pages).

## Chain strategy

Radian is **Arc-first, not Arc-only** — hub and spoke rather than symmetric multi-chain:

- **Arc = home.** USDC is the native gas coin, so dollar-priced launches are free. Brand, $RADIAN, the fee flywheel, and this token's birth all live here.
- **BSC = the real-stock spoke.** It is where tokenized stocks actually trade (Ondo, ~260 instruments) with canonical Uniswap V4 and no incumbent stock launchpad. When The Wall's treasury must hold *real* NVDA, it deploys there; fees flow home over Circle's CCTP.

The reference projects (PONS, PAIR, BSP) each picked one chain — the chain their pairing asset lives on. We have two stories on two chains, so we take exactly one spoke and no more.

## Roadmap

1. **Phase 1 — live (Arc testnet).** $WALL launched on Radian, priced in NVDAx, Buyback & Lock on. This site reads it live.
2. **Arc mainnet (Sept 16).** Radian deploys against canonical Uniswap V4; $WALL relaunches with real trading.
3. **Phase 2 — treasury + buy wall.** New contracts, audit, then deploy.
4. **Real stock.** Deploy where tokenized NVDA exists (BSC) so the treasury holds the real thing.

## Layout

```
site/            brand site (static)
docs/TREASURY.md Phase 2 mechanism spec + open questions
manifest.json    every address, tx hash, and parameter of the live deployment
```
