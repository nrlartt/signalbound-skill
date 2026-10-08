# Tool reference

Every tool is `POST <api base><path>` with a JSON body and a JSON reply. The machine-readable
version of everything below is published at `/openapi.json`, and the price list at
`/.well-known/x402`.

`tokenId` (integer, 1-6666) is accepted everywhere. It names the Signalbound agent the call is
made as: send it when the caller holds that agent, because it is how the discounted rate is
matched to a clearance tier. Callers paying the open rate leave it out.

Every reply includes a `brief` — the same result written as a short natural-language read, so
an agent can answer without walking the raw arrays.

---

## drop-radar — `POST /drop-radar`

NFT drops on OpenSea that are verifiably active or still ahead. Ended, undated and stale
open-ended drops are excluded rather than returned with a warning.

| Param | Type | Notes |
|-------|------|-------|
| `type` | string | `all`, `featured`, `upcoming`, `recently_minted` |
| `limit` | integer | 1-50, default 20 |
| `chain` | string | e.g. `ethereum`, `base`. Omit for all chains |
| `scanChains` | boolean | Also query known chains individually (default true) |
| `enrich` | boolean | Fetch extra per-drop detail (slower, default false) |
| `checkWalletEligibility` | boolean | Probe mint eligibility for `walletAddress` |
| `walletAddress` | string | The wallet to probe when checking eligibility |
| `watchSlugs` | array | Specific collection slugs to include |

Returns: `type`, `generatedAt`, `count`, `drops[]` (stage price, supply, mint window, mint
link, chain), `brief`.

---

## allowlist-scan — `POST /allowlist-scan`

Batch mint-eligibility: takes the drop feed and returns only what the given wallet can
actually mint right now.

| Param | Type | Notes |
|-------|------|-------|
| `walletAddress` | string | **Required** for callers paying the open rate |
| `type` | string | Same feeds as `drop-radar` |
| `limit` | integer | 1-50 |
| `chain` | string | Optional chain filter |
| `eligibleOnly` | boolean | Drop the ineligible list from the reply |
| `scanChains` | boolean | Per-chain scanning |

Returns: `scanned`, `eligibleCount`, `eligible[]` (stage, price, timing), `ineligible[]`,
`brief`.

---

## portfolio — `POST /portfolio`

What a wallet holds, and what is currently bid on it.

| Param | Type | Notes |
|-------|------|-------|
| `walletAddress` | string | **Required** for callers paying the open rate |
| `chain` | string | Optional chain filter |
| `nftLimit` | integer | How many NFTs to inspect |

Returns: `wallet`, `nftCount`, `nfts[]`, `offersReceived[]`, `estimatedValueUsd`, `brief`.

Estimated value is derived from live floors and moves with them; it is a reading, not a quote.

---

## offer-radar — `POST /offer-radar`

Floor, best listings and standing collection offers, ranked by spread.

| Param | Type | Notes |
|-------|------|-------|
| `slugs` | array | **Required**, 1-10 OpenSea collection slugs |

Returns: `count`, `collections[]` (floor, best listing, best collection offer, spread), `brief`.

---

## b20-radar — `POST /b20-radar`

Base's native B20 token standard: what was deployed from the factory precompile, and how deep
one token's market is.

| Param | Type | Notes |
|-------|------|-------|
| `address` | string | Inspect one B20 token instead of scanning |
| `limit` | integer | How many deployments to return |
| `blocksBack` | integer | How far back to scan |

Returns: `fromBlock`/`toBlock`, `count`, `detectedCount`, `truncated`, `tokens[]` (price, 24h
volume, liquidity, thin-market read), `brief`.

---

## robinhood-radar — `POST /robinhood-radar`

The most active assets on Robinhood Chain, or one of them in detail.

| Param | Type | Notes |
|-------|------|-------|
| `address` | string | Inspect a single token |
| `limit` | integer | How many to return |
| `includeStables` | boolean | Keep stablecoins in the list |

Returns: `count`, `tokens[]` (price, 24h volume, liquidity, depth read), `brief`.

---

## intel — `POST /intel`

One Signalbound agent's own record.

| Param | Type | Notes |
|-------|------|-------|
| `tokenId` | integer | **Required**, 1-6666 |

Returns: `name`, `rarity_tier`, `rarity_score`, `edition`, `traits`, `brief`.

---

## Errors

| Status | Meaning |
|--------|---------|
| 400 | The body did not match the tool's schema — the reply names the field |
| 402 | Payment required; the body carries the price options |
| 403 | The caller does not satisfy the tool's access policy |
| 5xx | Upstream data source failed; retry rather than re-paying blindly |

A `402` never charges anything. Reading the terms is free, and so is `quote`.
