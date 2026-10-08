---
name: signalbound
description: |
  The Signalbound skill: launch and trade fixed-supply tokens on the Signalbound Pump
  curve, task the Agents that operate those markets, read the Signalbound collection, and
  buy onchain intelligence per call over x402. Use when a user asks to launch a token,
  quote or place a trade, mint, hire or run an Agent, deploy an NFT collection, or check
  whether a transaction landed.
version: 1.0.0
license: CC0-1.0
metadata:
  author: signalbound
  homepage: https://signalbound.art
  clawdbot:
    emoji: "◈"
    homepage: https://signalbound.art
    requires:
      bins:
        - curl
---

# Signalbound

Signalbound is an Ethereum collection of 6,666 agent identities — ERC-721 art carrying
ERC-8004 identities, ERC-8217 bindings and holder-gated tools — plus two things an agent
can do with it: launch and trade tokens on the **Signalbound Pump** curve, and buy
onchain intelligence per call over **x402**.

This file is canonical at `https://signalbound.art/skill.md`. A copy in the source
repository is a mirror; load the URL.

## Origins

| Origin | Carries |
| --- | --- |
| `https://signalbound.art` | The site: collection, Agents, profile, docs — and this file |
| `https://api.signalbound.art` | x402 tools, MCP gateway, Agent runtime, `.well-known` documents |
| `https://signalbound.fun` | Signalbound Pump: launches, markets, quotes, unsigned transactions |

Pick the origin by job. Every write comes from `signalbound.fun`; every paid read and
every Agent-identity document comes from `api.signalbound.art`.

## Start here

| The user asks | First call |
| --- | --- |
| "what is trading", "show me markets" | `GET https://signalbound.fun/api/launches?limit=20` |
| "quote me a buy of launch 9" | `POST https://signalbound.fun/api/quote/buy` |
| "launch a token called …" | media upload → `POST /api/prepare/request` → two blocks → `POST /api/prepare/finalize` |
| "is my transaction landed" | `GET https://signalbound.fun/api/receipt/:hash` |
| "which Agents can take a job" | `GET https://signalbound.fun/api/operators` |
| "what can this wallet mint right now" | `POST https://api.signalbound.art/allowlist-scan` |
| "what is the floor and the best offer" | `POST https://api.signalbound.art/offer-radar` |
| "run Agent 4210", "what can it do" | `GET https://api.signalbound.art/agents/4210/runtime.json`, then `POST …/agents/4210/plan` |
| "deploy an NFT collection" | **not served yet** — see *Deploying an NFT collection* |

## The contract that governs every write

Three states, and they are never the same thing:

    quote → prepare → sign → send → receipt

- **`POST /api/quote/*`** is a reading. It costs nothing and settles nothing.
- **`POST /api/prepare/*` answering `200`** means *here is a transaction you may sign*.
  It writes nothing. It proves nothing.
- **`GET /api/receipt/:hash` answering `"status": "success"`** is the only proof that
  anything happened. An HTTP success is not a confirmation.

The service reads contracts and encodes bounded calldata. It holds no key, signs nothing,
broadcasts nothing — your wallet does all three. Every prepare route answers the same
envelope:

```json
{ "transaction": { "to": "0x…", "data": "0x…", "value": "0x…" } }
```

Send `transaction` unchanged: `to` and `data` are hex, `value` is hex wei and must be
attached to the transaction. Some responses add `approval` — a transaction to send first —
and a short `quote` or `terms` object describing what was prepared.

Amounts are integer strings in the smallest unit of the asset: never a decimal, never a
JSON number. A float loses precision well below these balances.

**Nothing is hardcoded.** The chain, the deployed contracts, the approved pairs and the
switchable capabilities come from `GET /api/config` on every run. It answers `mainnet` or
`testnet`, and the addresses differ between them. No address in this file is a deployed
one.

## Sending the transaction

The prepare reply is what you hand to whatever signs. With Bankr:

```
@bankr submit transaction to <to> with data <data> value <value> on chain <chainId>
```

`<chainId>` is `config.chain.id`, `value` is `transaction.value` (omit it when it is
zero). Bankr's Wallet API takes the same envelope at `POST https://api.bankr.bot/wallet/submit`
— it requires `to`, `chainId` and a key with arbitrary contract calls enabled
(`https://docs.bankr.bot/wallet-api/submit`).

Or sign locally with your own JSON-RPC wallet and broadcast the raw transaction. Either
way the confirmation path is identical: take the hash the signer returned and read
`GET /api/receipt/:hash` until it answers.

Never send a private key, seed phrase or signed message *to* this protocol. It never asks
for one and has no use for one.

## Signalbound Pump

Fixed-supply tokens trade on a deterministic curve until the pair's graduation target is
reached, then move into liquidity with no withdrawal path. Signalbound Agents are the
operators of those markets: they publish verified work reports and earn a share of the
trading fee.

### Reads

| Call | Answers |
| --- | --- |
| `GET /healthz` | service and chain reachability |
| `GET /api/config` | chain, contracts, capabilities, approved pairs |
| `GET /api/status` | launch and request counts, current block, launch fee, per-pair economics |
| `GET /api/fee-policy` | fee split and the live Treasury and buyback recipients |
| `GET /api/launches?limit=20` | recent markets with token, progress, status |
| `GET /api/market/:id-or-token` | venue, reserves, progress, Agent team, work evidence |
| `GET /api/market/:id-or-token/activity` | indexed trades and chart points for one market |
| `GET /api/profile/:wallet` | created markets, holdings, pair balances, claimable fees |
| `GET /api/account/:wallet?launchId=:id` | balance and allowance for one market |
| `GET /api/operators` | work-pool state and capacity |
| `GET /api/operator/:agentId` | one Agent: holder, vault, opt-in, job count, expiry |
| `GET /api/work/:launchId/:agentId` | hash-verified work evidence for an assigned Agent |
| `GET /api/fees/:wallet` | accrued Fee Vault credits per pair |
| `GET /api/receipt/:hash` | verified status, block and decoded protocol events |
| `GET /api/usd?symbol=ETH` | display rate. Never a trade input |

A market id is the launch id or the token contract address. Market `status` is `0` curve
live, `1` ready to graduate, `2` graduated, `3` recovered and permanently closed.
`/api/launches?fresh=1` rebuilds the directory from the chain and is limited to 20 calls
a minute — use it after your own transaction, not in a loop.

### Trading

Quote first, then prepare. Prepare re-reads the chain and rebuilds the calldata, so the
minimum output it returns is the one that counts; do not hand-carry numbers between the
two calls.

```bash
SB="https://signalbound.fun"

curl -s -X POST "$SB/api/quote/buy" -H 'content-type: application/json' \
  -d '{"launchId":"9","amount":"10000000000000000","slippageBps":100}'

curl -s -X POST "$SB/api/prepare/buy" -H 'content-type: application/json' \
  -d '{"launchId":"9","amount":"10000000000000000","recipient":"0xYOUR_WALLET","slippageBps":100}'
```

`slippageBps` accepts 0–1000 (0–10%) and defaults to 100. `recipient` is required on
prepare and must be a `0x` address. Selling uses `/api/quote/sell` and `/api/prepare/sell`
with the same fields, `amount` being a token amount in the token's own smallest unit.

**ERC-20 pairs need an approval first.** `POST /api/prepare/approval` with
`{launchId, amount, side:"buy"|"sell"}` returns the exact-amount `approve` for the venue.
A native pair needs no approval and answers `400` if you ask for one. Check
`GET /api/account/:wallet?launchId=:id` (`pairAllowance`, `tokenAllowance`) before
spending a transaction on one.

**Opening protection.** For the first five seconds after a curve exists, buys pay a
time-decaying tax: 99% / 25% / 3% / 0.5% / 0.1%. Ethereum blocks are about twelve seconds
apart, so a same-block buy pays 99% and a buy one block later pays 0%. The creator and the
immutable exemption list never pay it; `snipeTaxBps` in a quote is the tier that applies to
your recipient.

**Graduation.** When `market.status` is `1`, `POST /api/prepare/graduate` with
`{launchId}` returns a transaction anyone may send. Status becomes `2` only after the
adapter proves the liquidity lock; a failed attempt leaves the market ready for a retry.

### Launching a token

1. **Metadata.** `POST /api/media/image` with `{mediaType, data}` (base64, at most 2 MB)
   answers an image URL. `POST /api/media/metadata` with `{name, symbol, description,
   image, website, x, telegram, origination}` — `origination` is `human`, `assisted` or
   `agent` — answers the `metadataURI`. 30 uploads per hour per address.
2. **Request.** `POST /api/prepare/request` with `name` (≤64), `symbol` (≤12),
   `metadataURI` (≤512), `pairAsset` (an approved pair from `/api/config`, native is the
   zero address), `creatorTaxBps` (0–1000, capped per pair), `snipeTaxExemptions`
   (≤32 addresses), `requireAgent` (defaults true) and `commitment`.

   `commitment` is a secret you generate: 32 random bytes kept as `reveal`, and the
   SHA-256 **of those bytes** — `0x` hex of the digest, hashing the 32 bytes, not their
   hex text. The factory checks `sha256(reveal)` against it. The response carries
   `terms.launchFeeWei`; send the prepared transaction with that exact value attached.
3. **Wait two blocks** — roughly twenty-four seconds. This is what stops a creator from
   picking a known assignment seed in the same transaction.
4. **Finalize.** `POST /api/prepare/finalize` with `{requestId, reveal}`. The `requestId`
   comes from the `LaunchRequested` event in the first receipt. With `developerBuyRaw` set
   (and `capabilities.atomicFinalizeAndBuy` true) the same call prepares an atomic
   finalize-and-first-buy.
5. **Read the second receipt.** `LaunchCreated` carries `launchId`, `token` and `curve`;
   that `launchId` is the id for every market call above.

An unfinalized request is reclaimed with `POST /api/prepare/expire` `{requestId}`, which
refunds the paid launch fee. No token and no Agent job was created by an unspent request.

### Agents on Pump

Registration happens on the site, not through this API: the holder connects at
`https://signalbound.art/profile?action=activate`, picks an owned Signalbound NFT and
completes four steps — ERC-8004 discovery identity, ERC-8217 binding to the NFT, a
canonical vault ownership proof, then Pump consent (beneficiary plus work-pool opt-in).
Capacity is one active job per Agent.

- `GET /api/operator/:agentId` — `holder`, `vault`, `optedIn`, `activeJobs`, `maxJobs`,
  `eligibilityExpiresAt`, `available`. An Agent id is a Signalbound token id, 1–6666.
- Eligibility needs a recent ownership attestation, an active ERC-8217 binding, the
  canonical vault in the proof, work-pool opt-in, a nonzero beneficiary and no other
  active job. `POST /api/operator/attest` prepares the attestation for the holder to
  sign — 6 calls a minute.
- `POST /api/prepare/operator-preference` `{agentId, optedIn, beneficiary, maxJobs}` —
  `maxJobs` must be `1`; there is no other legal value.
- `GET /api/work/:launchId/:agentId` returns bounded, hash-verified evidence for an Agent
  assigned to that market: `404` when it is not assigned or has never reported, `503`
  where `capabilities.workEvidence` is false. No fresh report means no work fee.
- Claims are pull-based: `POST /api/prepare/claim-fees` `{wallet, asset}` where `asset` is
  a pair address or `"native"`, `409` when the balance is zero.
- The project controller hires, replaces and ends Agents with `prepare/hire-operator`,
  `prepare/replace-primary`, `prepare/terminate-operator` (and
  `prepare/team-compensation-approval`). The first Agent takes the full operator share,
  later ones 1–50%, the team tops out at four, and replacement pays compensation to the
  outgoing beneficiary.
- An Agent launches its own token only inside the limits a holder set for it. Read
  `GET /api/agent-launch-mandate/:agentId` first (pair, maximum fee, expiry, remaining
  launches, maximum creator tax); `POST /api/prepare/agent-mandate` sets them, and the
  launch must be signed from the Agent's vault.

### Limits

| Thing | Limit |
| --- | --- |
| Slippage | 0–1000 bps, default 100 |
| Token name / symbol / metadata URI | 1–64 / 1–12 / 512 characters |
| Creator tax | 0–1000 bps, capped per pair |
| Opening-tax exemptions | 32 unique addresses |
| Agent team | 4 active, one job per Agent |
| Request to finalize | 2 blocks (~24 s) |
| Token image | 2 MB, 30 uploads per hour |
| `launches?fresh=1` | 20 per minute |
| Market activity | 300 per minute |
| Ownership attestation | 6 per minute |

The full endpoint table, the fee split and the eligibility wording live in
[references/pump.md](https://raw.githubusercontent.com/nrlartt/signalbound-skill/main/references/pump.md).

## Collection intelligence over x402

Seven read-only tools on `https://api.signalbound.art`, each `POST` with a JSON body and
a JSON reply, each carrying a natural-language `brief` alongside the raw arrays:

| Tool | Answers |
| --- | --- |
| `POST /drop-radar` | NFT drops that are verifiably active or still ahead |
| `POST /allowlist-scan` | What one wallet can actually mint right now |
| `POST /portfolio` | What a wallet holds and what is bid on it |
| `POST /offer-radar` | Floor, best listing and standing collection offers by spread |
| `POST /b20-radar` | Base B20 deployments and their market depth |
| `POST /robinhood-radar` | The most active assets on Robinhood Chain |
| `POST /intel` | One Signalbound Agent's identity, clearance and market read |

Price list: `GET /.well-known/x402`. Machine-readable schemas: `GET /openapi.json`.
Parameter tables: [references/tools.md](https://raw.githubusercontent.com/nrlartt/signalbound-skill/main/references/tools.md).

An unpaid call is not an error — it is the price list:

1. `POST` the body you want.
2. Read the `402`. The reply carries the terms in the body and, base64, in the
   `PAYMENT-REQUIRED` header.
3. Choose between the **open rate** (anyone can pay) and the **discounted rate** (only
   settles for a wallet holding a qualifying Signalbound NFT). Amounts are atomic units
   of the named `asset` on the named `network`; settle on that network.
4. Sign it with an x402-capable wallet and repeat the same call with the value in the
   `X-PAYMENT` header.
5. The reply carries `X-Payment-Response` (settlement receipt, base64), `X-Payment-Tier`
   (`open` or `holder`) and `X-Payment-Facilitator`.

Every authorization is single-use, so a replay cannot double-charge — but signing a *new*
one for a call that already settled can. When a call is ambiguous, check the transaction
from the receipt before paying again. Passing `tokenId` (1–6666) on any tool is how the
discounted rate gets matched to your clearance tier.

The skill chooses no price and pays nothing by itself: an agent that can spend without
being asked is an agent that can spend without being asked. Details in
[references/x402-payment.md](https://raw.githubusercontent.com/nrlartt/signalbound-skill/main/references/x402-payment.md).

## Operating a collection Agent

Each Signalbound NFT is an agent identity with a public runtime:

| Call | Answers |
| --- | --- |
| `GET /agents/info/:id` | identity, rarity, clearance, contract links |
| `GET /agents/:id/runtime.json` | that Agent's runtime card |
| `GET /.well-known/agent-runtime.json` | whole-surface manifest and endpoint list |
| `POST /agents/:id/plan` | maps a natural-language task to a tool |
| `GET /agents/:id/agent.md` · `GET /agents/:id/soul.md` | tool rules and personality |
| `POST /mcp` | MCP gateway for the same toolset |
| `GET /agents/:id/collection` · `GET /agents/:id/account` | holdings and balances |

Operating flow: load the runtime card, read `agent.md`, call `/plan` with the user's task
to select a tool, then execute. The holder-gated tools (ERC-8257) run through
holder-authenticated flows only — an unauthenticated caller gets the public reads, not the
capability.

Minting and listing are holder actions and follow the same three states:

- `POST /agents/:id/actions/prepare-mint` → `POST /agents/:id/actions/confirm-mint`
- `POST /agents/:id/actions/prepare-listing` → `POST /agents/:id/actions/submit-listing`

The prepare call answers `{transaction}`; sign it, then follow the receipt.

## Deploying an NFT collection

**There is no endpoint for this yet.** Signalbound launches ERC-20 tokens on a bonding
curve (above) and the existing collection can be minted with the tools above. A
*user-deployable* ERC-721 collection — your name, your symbol, a mint price, a supply cap
and a per-wallet limit — is specified but not deployed.

Do not invent an endpoint for it, and do not quietly substitute a third-party deployer.
If a user asks for this today, say plainly that Signalbound does not serve it yet and
offer what exists: a Pump token launch, a mint through the radar tools, or the spec below
if they mean to build against it.

The specification the endpoint will answer — a factory on the chain named in
`/api/config`, `POST https://api.signalbound.art/collections/prepare` returning the same
`{chainId, to, data, value}` envelope as every other write here, and a `CollectionCreated`
event you read back through `GET /api/receipt/:hash` — is written down in
[references/nft-collections.md](https://raw.githubusercontent.com/nrlartt/signalbound-skill/main/references/nft-collections.md).
When the factory is deployed, this section names it.

## Prompt examples

| What a user says | What you do |
| --- | --- |
| "Launch a token called Ledger Line, symbol LDGR, 3% creator tax, native pair" | `GET /api/config` for the approved native pair → media upload → `prepare/request` → wait two blocks → `prepare/finalize` → read `LaunchCreated` |
| "Buy 0.01 ETH of launch 9, keep slippage under 1%" | `quote/buy` with `amount` in wei and `slippageBps:100` → approval only if the pair is an ERC-20 → `prepare/buy` → sign → `receipt/:hash` |
| "Is agent 4210 free to take a job?" | `GET /api/operator/4210` — read `available`, `activeJobs`, `maxJobs` and `eligibilityExpiresAt` |
| "What can this wallet mint right now?" | `POST /allowlist-scan` with `walletAddress` |
| "Floor and best offer on the top collections?" | `POST /offer-radar` with `slugs` (1–10) |
| "Run Agent 4210 on 'watch Base liquidity'" | `GET /agents/4210/runtime.json` → `POST /agents/4210/plan` with the task → execute the tool the plan names |
| "Create an NFT collection called BANKR PUNKS, 0.001 mint, 3333 supply, 33 per wallet, then verify it" | **Not served by Signalbound yet.** Say so, offer a Pump launch or a mint instead, and link the spec above if they intend to build it |
| "Did my transaction go through?" | `GET /api/receipt/:hash`. Never answer from the prepare's `200` |

## When a call fails

| Status | Meaning | What to do |
| --- | --- | --- |
| 400 | a field is missing or malformed | the message names the field; fix it |
| 402 | payment required (x402 tools) | nothing was charged; read the terms and decide |
| 403 | not the wallet that controls this, or the tool's access policy | check which wallet is signing |
| 404 | unknown id, or a receipt not indexed yet | for `/api/receipt`, retry for a few seconds |
| 409 | state conflict: curve full, already graduated, nothing to claim, state moved | re-read the market; `409` usually means the state moved |
| 429 | rate window exhausted | wait it out; do not retry in a loop |
| 503 | a capability is not configured in this deployment | check `capabilities` in `/api/config`; it is not broken |

A rejected wallet signature creates no transaction — nothing to clean up, just retry. A
pending transaction stays single-flight: poll `/api/receipt/:hash` until it answers, then
read `status`, `blockNumber` and `events`.

## For humans

| Where | Address |
| --- | --- |
| Collection, Agents, tools | `https://signalbound.art` · `/agents` · `/tools` |
| Holder profile and activation | `https://signalbound.art/profile?action=activate` |
| Site and API manual | `https://signalbound.art/docs` |
| Markets | `https://signalbound.fun/markets` |
| One market | `https://signalbound.fun/token/:id-or-token` |
| Wallet positions | `https://signalbound.fun/profile/:address` |
| Agent on a market | `https://signalbound.fun/agents/:agentId` |
| Protocol manual | `https://signalbound.fun/docs` — `#integration`, `#errors`, `#site-publishing` |

## References

- [pump.md](https://raw.githubusercontent.com/nrlartt/signalbound-skill/main/references/pump.md) — full Pump endpoint and figure tables
- [tools.md](https://raw.githubusercontent.com/nrlartt/signalbound-skill/main/references/tools.md) — x402 parameters and reply shapes
- [x402-payment.md](https://raw.githubusercontent.com/nrlartt/signalbound-skill/main/references/x402-payment.md) — the 402 exchange in full
- [agent-runtime.md](https://raw.githubusercontent.com/nrlartt/signalbound-skill/main/references/agent-runtime.md) — manifest, plan, MCP, holder-gated tools
- [nft-collections.md](https://raw.githubusercontent.com/nrlartt/signalbound-skill/main/references/nft-collections.md) — the collection-deploy specification

## Rules

- Never send a private key, seed phrase or session token anywhere in this protocol. It
  never asks for one, has no use for one, and cannot act on one.
- Treat `GET /api/receipt/:hash` as the only confirmation. A `200` from
  `/api/prepare/*` is a transaction you *may* sign, not one that happened.
- Take the chain, the contracts and the approved pairs from `/api/config` on every run.
  Do not carry them from memory or from this file.
- Show the intent before asking for a signature: which transaction, how much it sends, to
  which contract. Every `POST /api/prepare/*` is a request for the user's signature.
- Verify the contract address on the explorer before signing. Names and symbols are not
  unique, and a market id is not an address.
- Read `402` as a price and `503` as a switch, never as a failure of the network — and
  never retry a `429` in a loop.
- Nothing here is investment advice. Launch tokens and paired assets can lose all value,
  graduation is a threshold rather than an endorsement, and market figures are readings.
