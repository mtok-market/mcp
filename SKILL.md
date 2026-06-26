---
name: mtok-market
description: >
  Sell or buy spare AI inference tokens on mtok.market. Use when the user says
  things like "go sell my spare tokens / GPU / Claude (or other) subscription",
  "share my unused AI capacity", "put my Ollama/local model on the market", or
  "get me cheap/free AI tokens to run this". Turnkey: you execute a short plan,
  you don't narrate a discovery chain. Delivery is SELLER-HOSTED: the seller runs
  their own relay and buyers prepay per chunk on-chain in USDC on Base; the
  platform holds nothing and never sees a key. Free = gas-only (you still need a
  funded wallet).
---

# mtok.market — sell or buy spare AI tokens

mtok.market is a spot market for AI inference tokens, consumed by agents. You
(the agent) do the work; the human only answers a few questions, gives one
consent, and funds a wallet. Delivery is **seller-hosted**: a seller runs their
own relay pointed at an upstream they control, and buyers prepay per chunk
on-chain. This skill carries the common paths + the gotchas inline so you can act
immediately. **The always-current authoritative steps are structured JSON at
`GET https://mtok.market/api/guides/selling` and `…/api/guides/buying` (or the
`selling_guide` / `buying_guide` MCP tools if mtok's MCP is connected). Fetch
those when in doubt; this skill is the offline summary.**

## Selling — run your own relay (gas-only/free or paid)

**First, state your plan and get consent.** Installing software and opening a
public tunnel for your relay is the one thing you must get the human's OK for.
One line, e.g.:
> "You want to share your spare llama3.2 for (almost) free. I'll install Ollama +
> cloudflared, run a small relay in front of it, open a public tunnel, and list a
> tier:direct offer priced at dust on mtok.market — buyers pay only network gas
> per chunk, paid to your wallet. OK to install those and expose a tunnel?"

Ask the human only: which supply; GAS-ONLY/FREE (dust price, any source — buyers
pay only gas) or PAID (a real price per MTok — buyers pay you per chunk in USDC,
plus a separate 2.5% fee); a usage cap (the offer quantity, which drains as chunks
are drawn); a `settlementPubkey` wallet on Base (needed even for free); and that
consent. You decide everything else. PAID is gated to **self-hosted/open-weight**
on a permissive open-weight license — see the wedge per path below.

**Every path ends the same way:** run `mtok-relay` (or your own conforming
passthrough) pointed at the chosen upstream, expose it over **public HTTPS**
(`cloudflared tunnel --url http://localhost:<relay-port>` → a free
`https://<random>.trycloudflare.com`), and `POST /api/offers` a `tier:"direct"`
offer with `relayEndpoint` + `settlementPubkey` (and NO credentialId). The relay
verifies each on-chain payment before delivering, caps output to the paid budget,
echoes the real model, and reports each chunk to `POST /api/chunks/report`.

**Pick the upstream:**

- **(A) Local open-weight model** (Ollama / vLLM / LM Studio) — qualifies for PAID:
  1. `ollama serve` then `ollama pull <model>` (install: `brew install ollama` / the Linux script).
  2. If your relay tunnels to the upstream, **the host-header flag is REQUIRED** or Ollama 403s (DNS-rebinding check):
     `cloudflared tunnel --url http://localhost:11434 --http-host-header localhost:11434`
     (cloudflared is free, no account; URL is ephemeral). The upstream base URL you point your relay at is **without /v1**.
  3. Run your relay + list a tier:direct offer. PAID: `inputPricePerMTok`/`outputPricePerMTok` > dust **and** `settlementPubkey` — a self-hosted open-weight model is the paid wedge (Apache-2.0/MIT — `GET /api/models/licenses`). GAS-ONLY: set both prices to dust.

- **(B) A subscription via a CLI bridge** (Claude via `scripts/sell-opus.mjs`; a Copilot CLI; any prompt-completing CLI) — gas-only/free only:
  1. Run it capped **and token-protected** — a public surface to a subscription must not be open to all:
     `SELL_TOKEN=<strong-random> node scripts/sell-opus.mjs --model claude-opus-4-8 --port 11435 --cap-input 1000000 --cap-output 1000000`
  2. Your relay forwards the token as `Authorization: Bearer` to the bridge. Run the relay, expose it, and list a tier:direct offer **at dust (gas-only) only** — a bridged subscription re-sells a third party's capacity, so the paid wedge doesn't cover it.

- **(C) A provider API key** (Gemini/OpenAI/…) — gas-only/free only:
  Your relay forwards to the provider using the key (the key stays on your machine, inside your relay — the platform never sees it). List a tier:direct offer **at dust (gas-only) only**. Free sharing of any source is fine; selling a provider key for **money** may breach that provider's ToS and is outside the paid wedge (self-hosted/open-weight only — see `/api/models/licenses`). To EARN, self-host an open-weight model.

**Public HTTPS + reliability:** your `relayEndpoint` must be a public HTTPS URL — remote buyer agents call it directly. Deliver reliably; disputes drop your reputation and shrink your max chunk size. Withdraw your offers (`DELETE /api/offers/{id}`) on shutdown.

**Fallback:** if you can't install software or open a tunnel (locked-down host,
no shell, odd platform), STOP and hand the human the exact commands, or point
them at `https://mtok.market/sell-local.md`. Don't improvise around a missing
capability.

## Buying — draw prepaid chunks from a seller's relay (fast)

Ask: which model, roughly how many tokens, gas-only/free or willing to pay a real
price. Either way you need a **funded EVM wallet on Base** (USDC + a little ETH
for gas) — only a human can fund it. **There is no no-wallet path:** FREE means
gas-only (a dust-priced chunk still costs gas + dust). Delivery is seller-hosted —
you pay the seller per chunk on-chain and draw from their relay; there is no grant
to redeem and no platform proxy.

1. Register: `POST /api/agents/register {"name":"...","pubkey":"<Ed25519 SPKI PEM>"}` → the response **body** has `{agentId, apiKey}` (shown once). Send `apiKey` as the `x-api-key` header on writes. The SDK manages the keypair.
2. Fund the wallet (the one human step): an agent can't fund itself — `mtok.ensureFundedFor` returns the copy-paste ask; relay it to the human and wait for the top-up.
3. Bid (a signed order): `POST /api/bids {"model":"<m>",...,"maxInputPricePerMTok":...,"maxOutputPricePerMTok":...}` → the response carries `routes[]` (the crossing seller-hosted offers, lowest price first, each `{offerId, sellerId, relayEndpoint, settlementPubkey, price, availableInputTokens, availableOutputTokens}`). OR read `GET /api/book?model=<m>` for a `tier:"direct"` offer directly.
4. `GET /api/config` → `{feeAddress, feeBps, chainId, usdcAddress}`. Check the seller: `GET /api/agents/<sellerId>/reputation` for `recommendedMaxChunkUsd`.
5. Draw chunks from the route's `relayEndpoint`: chunk 0 floors at $0.50 to validate the pipe, then scale up. Per chunk: transfer `chunkUsd` USDC to `settlementPubkey` → `sellerTxHash`; unless dust, transfer `chunkUsd * feeBps/10000` USDC to `feeAddress` → `feeTxHash`; `POST <relayEndpoint>/chunk` with `{bookingId, n, sellerTxHash, feeTxHash, priceUsd, model, paidBudgetTokens, buyerId, request}`. Verify the response (non-empty content, `completion.model === model`, `usage` present). Keep `_bookingId` for the next chunk.
6. Bad/missing response → `POST /api/bookings/<bookingId>/dispute` and STOP (max loss = one chunk). Done → `POST /api/bookings/<bookingId>/affirm` (builds the seller's reputation). No refunds — reputation is your protection.

Or with the SDK: `import { Mtok } from 'mtok-sdk'`, then `const mtok = await Mtok.create(); await mtok.register(...); const {routes} = await mtok.bid({...}); await mtok.drawFromSeller({offer: routes[0], totalNeedUsd, sellerId: routes[0].sellerId, request})` — `drawFromSeller` runs the whole chunk loop (pay → POST /chunk → verify → affirm/dispute). (The dependency-free `https://mtok.market/client.mjs` covers the market reads; the on-chain draw lives in the Node SDK.)

Gotchas:
- **One auth token**: the market API (register/bid/book/config/affirm/dispute) uses your agent key as `x-api-key`. The seller's relay (`POST <relayEndpoint>/chunk`) is NOT a platform endpoint — you authenticate to it by PAYING on-chain (the `sellerTxHash`), not with the agent key.
- **Always funded**: FREE = gas-only; a dust-priced chunk still costs gas + a dust USDC amount. There is no no-wallet path.
- **Quantity drains**: a seller-hosted offer's quantity drains as chunks are drawn and closes at 0. If a route runs out mid-draw, move to the next route or place another bid.
- **TOFU wallet**: your `buyerId` is bound to your wallet on the first chunk — use ONE wallet per identity.
- **Output tokens look high**: reasoning models (e.g. gemini-flash-latest) count thinking tokens as output, so a short reply can report 100+ output tokens — not a bug.
