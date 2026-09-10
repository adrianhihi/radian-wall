# The Wall ($WALL) — Radian's flagship product

**A token priced in NVDA shares. Phase 1 live on Arc testnet; Phase 2 (the treasury) in design.**

The Wall is the first product built *on* [Radian](https://github.com/adrianhihi/radian) (the launchpad for Circle's Arc chain), the way BTCNVDA is the flagship product built on BSP Treasury. This repo is the **product**; the launchpad, protocol token and web app live in the **platform** repo.

| Repo | Role | Contains |
| --- | --- | --- |
| [`radian`](https://github.com/adrianhihi/radian) (platform) | The launchpad | Pons V2 contracts, $RADIAN fee flywheel, web app, indexer, deploy scripts |
| `radian-wall` (this, product) | One flagship token | Brand site, mechanism spec, deployment manifest — and, in Phase 2, the treasury contracts |

## What it is — exactly

$WALL is launched on Radian's bonding curve paired against a tokenized-stock quote asset, so it is **priced in shares, not dollars**. It was launched in **Buyback & Lock** mode. What that mode does today, no more and no less:

1. Every trade pays a 1% fee in the quote asset (NVDAx).
2. 35% of that fee is the *buyback share*. It accumulates on the curve; when the platform's fee-sweep operator runs a sweep, it buys $WALL back from the market. **It is not automatic per trade.**
3. Bought-back $WALL goes into the platform's BuybackVault and is **released linearly over 5 years — 70% to the creator, 30% to the protocol**. It is a vest, not a burn and not a permanent lock.

That mechanic exists in the platform today with **zero new contracts**. Everything else on the site — the treasury that hoards NVDA, the floor-price buy wall, burn/stake — is **Phase 2, designed and not built**: see [`docs/TREASURY.md`](docs/TREASURY.md). See [`manifest.json`](manifest.json) for every address and the launch transaction.

## Status — read this before anything else

- **Network:** Arc **testnet** (chain 5042002). No real money. Arc mainnet opens 2026-09-16.
- **Quote asset:** `NVDAx` is a clearly-labeled **testnet stand-in** for NVDA — a plain ERC-20 minted by us, whose deployer can mint without limit. It is not an asset. Real tokenized stocks trade on BSC (Ondo) and Robinhood Chain; that is where a real-stock deployment goes.
- **Contracts:** Phase 1 uses only the platform's Pons V2 port (source-identical to the Pons sources that are Sourcify-verified on Robinhood Chain; on Arc testnet only the factory is verified on Arcscan so far). Verification proves source = bytecode; **it is not an audit**. Phase 2 contracts do not exist yet and will be audited before holding funds.
- **Reference prices** on the site come from a delayed, keyless off-chain feed via the platform's `/api/stock-price` route. **No contract reads them.** The curve's own trading price is the price.

## Site

[`site/index.html`](site/index.html) is a single static page — no framework, no build. It reads live state from three public sources: the curve's `getReserves()` over JSON-RPC (spot price in NVDA), the Radian indexer (`/token/:addr`: curve progress, vault balance, trades), and the platform's reference-price route. Indexer rows are treated as untrusted input (validated, rendered with `textContent` only).

```sh
# preview locally
python3 -m http.server 8080 --directory site
# → http://localhost:8080
```

Deploy `site/` to any static host (Vercel, Cloudflare Pages, GitHub Pages). `vercel.json` sets a Content-Security-Policy that only allows connections to the RPC, the indexer and the platform.

## Chain strategy

Radian is **Arc-first, not Arc-only** — hub and spoke rather than symmetric multi-chain:

- **Arc = home.** USDC is the native gas coin, so dollar-priced launches are free. Brand, $RADIAN, the fee flywheel, and this token's birth all live here.
- **BSC = the real-stock spoke.** It is where tokenized stocks actually trade (Ondo, ~260 instruments) with canonical Uniswap V4 and no incumbent stock launchpad. When The Wall's treasury must hold *real* NVDA, it deploys there. **BSC has no native USDC** (Circle's CCTP there carries USYC only), so moving fees home is an open item — a third-party bridge or USYC — not a solved one. Verified facts are in the platform's `MULTICHAIN.md`.

The reference projects (PONS, PAIR, BSP) each picked one chain — the chain their pairing asset lives on. We have two stories on two chains, so we take exactly one spoke and no more.

## Roadmap

1. **Phase 1 — live (Arc testnet).** $WALL launched on Radian, priced in NVDAx, Buyback & Lock on. This site reads it live.
2. **Arc mainnet (Sept 16).** Radian deploys against canonical Uniswap V4; $WALL relaunches on mainnet. Its quote asset there depends on whether a tokenized stock exists on Arc by then.
3. **Phase 2 — treasury + buy wall.** New contracts, audit, then deploy.
4. **Real stock.** Deploy where tokenized NVDA exists (BSC) so the treasury holds the real thing.

## Layout

```
site/            brand site (static)
docs/TREASURY.md Phase 2 mechanism spec + open questions
manifest.json    every address, tx hash, and parameter of the live deployment
```
