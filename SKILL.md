---
name: mtok-market
description: >
  Sell or buy spare AI inference tokens on mtok.market. Use when the user says
  things like "go sell my spare tokens / GPU / Claude (or other) subscription",
  "share my unused AI capacity", "put my Ollama/local model on the market", or
	  "get me cheap AI tokens to run this". Turnkey: you execute a short plan,
  you don't narrate a discovery chain. Delivery is SELLER-HOSTED: the seller runs
	  their own relay and buyers pay per chunk on-chain in USDC on Base; the
	  platform holds nothing and never sees a key. There is no no-wallet or price-0 path.
---

# mtok.market, sell or buy spare AI tokens

mtok.market is a spot market for AI inference tokens, consumed by agents. You
(the agent) do the work; the human only answers a few questions, gives one
consent, and funds a wallet. Delivery is **seller-hosted**: a seller runs their
own relay pointed at an upstream they control, and buyers prepay per chunk
on-chain. This skill carries the common paths + the gotchas inline so you can act
immediately. **The always-current authoritative steps are structured JSON at
`GET https://mtok.market/api/guides/selling` and `…/api/guides/buying` (or the
`selling_guide` / `buying_guide` MCP tools if mtok's MCP is connected). Fetch
those when in doubt; this skill is the offline summary.**

## Free sharing, no market

If the human only wants to hand one person a private endpoint and key, do not
list an offer. Run `npx mtok-bridge` instead. It serves any model as an
OpenAI-compatible API with no payment, no mtok account, no market listing, and
nothing reported anywhere:

```sh
npx mtok-bridge --upstream https://api.openai.com/v1 --upstream-key sk-... --model gpt-4o-mini
```

`mtok-bridge` is the transport core the paid relay wraps. Start free there; move
one layer up to `mtok-relay` only when the human wants on-chain payment and
market discovery.

## Selling, run your own relay

**First, state your plan and get consent.** Installing software and opening a
public tunnel for your relay is the one thing you must get the human's OK for.
One line, e.g.:
> "You want to sell spare llama3.2 capacity. I'll install Ollama +
> cloudflared, run a small relay in front of it, open a public tunnel, bind your
> seller wallet once, and list a positive-price tier:direct offer on mtok.market.
> OK to install those and expose a tunnel?"

Ask the human only: which supply; the price per MTok (must be positive; buyers
pay per chunk in USDC plus the configured platform fee); a usage cap (the offer
quantity, which drains as chunks are drawn); a `settlementPubkey` wallet on Base
with a little ETH for the one-time contract bind; and that consent. You decide everything else. PAID is open in the current launch
config: the platform does not gate model licenses, and the seller is responsible
for the right to sell what they list. The model license registry is guidance.

**Every path ends the same way:** run `mtok-relay` (or your own conforming
passthrough) pointed at the chosen upstream, expose it over **public HTTPS**
(`cloudflared tunnel --url http://localhost:<relay-port>` => a free
`https://<random>.trycloudflare.com`), and `POST /api/offers` a `tier:"direct"`
offer with `relayEndpoint` + `settlementPubkey` (and NO credentialId). The relay
verifies each on-chain payment before delivering, caps output to the paid budget,
and echoes the real model. It is REPORT-FREE: the platform indexes each draw from
MtokDripLedger events on Base, so the relay reports nothing (the canonical tape is
`GET /api/chain/draws`).

**Pick the upstream:**

- **(A) Local model** (Ollama / vLLM / LM Studio):
  1. `ollama serve` then `ollama pull <model>` (install: `brew install ollama` / the Linux script).
  2. If your relay tunnels to the upstream, **the host-header flag is REQUIRED** or Ollama 403s (DNS-rebinding check):
     `cloudflared tunnel --url http://localhost:11434 --http-host-header localhost:11434`
     (cloudflared is free, no account; URL is ephemeral). The upstream base URL you point your relay at is **without /v1**.
  3. Run your relay + list a tier:direct offer with positive `inputPricePerMTok` / `outputPricePerMTok`, `settlementPubkey`, and `payoutAddress`. Bind the seller wallet once when `dripContractAddress` is configured. You are responsible for the right to sell what you list; `GET /api/models/licenses` is guidance, not a gate.

- **(B) A subscription via a CLI bridge** (Claude via `tools/scripts/sell-opus.mjs`; a Copilot CLI; any prompt-completing CLI):
  1. Run it capped **and token-protected**, a public surface to a subscription must not be open to all:
     `SELL_TOKEN=<strong-random> node tools/scripts/sell-opus.mjs --model claude-opus-4-8 --port 11435 --cap-input 1000000 --cap-output 1000000`
  2. Your relay forwards the token as `Authorization: Bearer` to the bridge. Run the relay, expose it, and list a positive-price tier:direct offer. The seller is choosing to sell and is responsible for the relevant subscription/provider terms.

- **(C) A provider API key** (Gemini/OpenAI/...):
  Your relay forwards to the provider using the key (the key stays on your machine, inside your relay, the platform never sees it). Positive-price listing is allowed by the platform, but may breach provider terms; the seller owns that decision and risk.

**Public HTTPS + reliability:** your `relayEndpoint` must be a public HTTPS URL, remote buyer agents call it directly. Deliver reliably; disputes drop your reputation and shrink your max chunk size. Withdraw your offers (`DELETE /api/offers/{id}`) on shutdown.

**Fallback:** if you can't install software or open a tunnel (locked-down host,
no shell, odd platform), STOP and hand the human the exact commands, or point
them at `https://mtok.market/sell-local.md`. Don't improvise around a missing
capability.

## Buying, draw paid chunks from a seller's relay (fast)

Ask: which model, roughly how many tokens, and max positive price. You need a
**funded EVM wallet on Base** (USDC + a little ETH for gas), only a human can
fund it. **There is no no-wallet path and no price-0/free lane.** Delivery is
seller-hosted: you pay the seller per chunk on-chain and draw from their relay;
there is no grant to redeem and no platform proxy.

1. Register: `POST /api/agents/register {"name":"...","pubkey":"<Ed25519 SPKI PEM>"}` => the response **body** has `{agentId, apiKey}` (shown once). Send `apiKey` as the `x-api-key` header on writes. The SDK manages the keypair.
2. Fund the wallet (the one human step): an agent can't fund itself, `mtok.ensureFundedFor` returns the copy-paste ask; relay it to the human and wait for the top-up.
3. Bid (a signed order): `POST /api/bids {"model":"<m>",...,"maxInputPricePerMTok":...,"maxOutputPricePerMTok":...}` => the response carries `routes[]` (the crossing seller-hosted offers, lowest price first, each `{offerId, sellerId, relayEndpoint, settlementPubkey, inputPricePerMTok, outputPricePerMTok, availableInputTokens, availableOutputTokens}`). OR read `GET /api/book?model=<m>` for a `tier:"direct"` offer directly.
4. `GET /api/config` => `{feeAddress, feeBps, chainId, usdcAddress, dripContractAddress}`. Check the seller: `GET /api/agents/<sellerId>/reputation` for `recommendedMaxChunkUsd`.
5. Draw chunks from the route's `relayEndpoint`: bind your agent wallet once, call MtokDripLedger `payDraw` for one bounded chunk (seller amount + configured fee), then `POST <relayEndpoint>/chunk` with `{bookingId, n, drawPaidTxHash, buyerId, request}`. The relay verifies DrawPaid including requestHash before upstream delivery, caps output to the paid budget, and returns the completion plus `_bookingId`/`remainingUsd` (report-free: the platform indexes the draw from the contract's events). Verify the response (non-empty content, `completion.model === model`, `usage` present). A paid draw is inProcess until you affirm or dispute it on-chain; the delivered/settled tape is `GET /api/chain/draws`.
6. Bad/missing response => `disputeDraw` on-chain and STOP (max loss = one draw). Good => `affirmDraw` on-chain (that IS the close; real non-self draws above dust build the seller's reputation). No refunds, reputation is your protection.

Or with the SDK: `import { Mtok } from 'mtok-sdk'`, then `const mtok = await Mtok.create(); await mtok.register(...); const {routes} = await mtok.bid({...}); await mtok.drawFromSeller({offer: routes[0], totalNeedUsd, sellerId: routes[0].sellerId, request})`, `drawFromSeller` runs the whole chunk loop (pay => POST /chunk => verify => affirmDraw/disputeDraw on-chain). (The dependency-free `https://mtok.market/client.mjs` covers the market reads; the on-chain draw lives in the Node SDK.)

Gotchas:
- **One auth token**: the market API (register/bid/book/config) uses your agent key as `x-api-key`. The seller's relay (`POST <relayEndpoint>/chunk`) is NOT a platform endpoint, you authenticate to it by PAYING on-chain (the `drawPaidTxHash`), not with the agent key.
- **Always funded**: every live draw carries a positive USDC payment plus gas. There is no no-wallet path.
- **Quantity drains**: a seller-hosted offer's quantity drains as chunks are drawn and closes at 0. If a route runs out mid-draw, move to the next route or place another bid.
- **TOFU wallet**: your `buyerId` is bound to your wallet on the first chunk, use ONE wallet per identity.
- **Output tokens look high**: reasoning models (e.g. gemini-flash-latest) count thinking tokens as output, so a short reply can report 100+ output tokens, not a bug.
