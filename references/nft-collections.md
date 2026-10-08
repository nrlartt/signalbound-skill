# Deploying an NFT collection — specification

**Status: specified, not deployed.** No route, contract or address in this file exists
yet. The skill says so plainly, and this document is what the endpoint will answer when
it does. Do not call any URL below before this section of the skill names it — you will
get a `404`.

## Why the shape is what it is

The ask is ordinary: *create a collection called X, mint price 0.001 ETH, 3333 supply, 33
per wallet, then verify the contract.* Four parameters, one deployment, one verification.

The obstacle is mechanical. An agent wallet signs a transaction through a submit API, and
those APIs address a `to` contract — a contract *creation* has no `to`, so a bare deploy
cannot be submitted through the same path that submits a trade. Two ways out:

1. a signer that accepts contract creation (works for a local wallet, not for a hosted
   submit API), or
2. **a factory**: a deployed contract that receives a normal call, deploys the collection
   and returns its address.

Signalbound takes the second. One ordinary transaction with a `to`, from any signer that
can already trade.

## The endpoint

`POST https://api.signalbound.art/collections/prepare`

Request:

| Field | Type | Rule |
| --- | --- | --- |
| `name` | string | 1–64 characters |
| `symbol` | string | 1–12 characters |
| `maxSupply` | integer string | > 0, at most 10,000 |
| `mintPrice` | string | wei, integer string, never a JSON number |
| `maxMintPerWallet` | integer string | ≥ 1, at most `maxSupply` |
| `baseURI` | string | ≤512 characters; `tokenURI` is `baseURI + tokenId` |
| `contractURI` | string | marketplace metadata, ≤512 characters |
| `royaltyBps` | integer | 0–1000 |
| `wallet` | string | the deployer's `0x` address, optional |
| `chainId` | integer | optional; defaults to the chain in `/api/config` |

Response — the same envelope every other write here answers, so one signer path serves
all of them:

```json
{
  "transaction": { "to": "<factory>", "data": "0x…", "value": "0x0", "gasLimit": "0x…" },
  "terms": { "collection": "<predicted or pending address>" },
  "chainId": 1
}
```

- `to` is the factory on the chain named in `GET /api/config`.
- `value` is `0`: the deployment is paid as gas, never as call value.
- `terms.collection` is only supplied when the address is knowable before the
  transaction; otherwise it is omitted and the address is read from the event.

Failures follow the rest of the platform: `400` names the malformed field, `409` means
the factory already knows that `(creator, symbol)`, `429` is the deploy rate window,
`503` means the factory address is not configured in this deployment.

## The factory

`SignalCollectionFactory.createCollection(...)` takes the constructor arguments above,
deploys `SignalCollection` and emits:

```solidity
event CollectionCreated(
  address indexed collection,
  address indexed creator,
  string name,
  string symbol,
  uint256 maxSupply,
  uint256 mintPrice,
  uint256 maxMintPerWallet
);
```

`SignalCollection` is a plain ERC-721 with:

- `mint(quantity)` payable: reverts above `maxSupply` and above the caller's
  `maxMintPerWallet`; price per token is `mintPrice`, sent to the creator's vault;
- `tokenURI(tokenId)` = `baseURI + tokenId`, `contractURI()` for marketplaces;
- EIP-2981 royalties at `royaltyBps`;
- owner-only `setBaseURI`, `setMintPrice`, `setContractURI`, `withdraw` — supply,
  symbol, name and the per-wallet cap are immutable after deploy;
- no pause, no blacklist, no admin mint above `maxSupply`.

Immutability is the point: a buyer reading the contract sees the same cap and price the
deploy request stated.

## Confirmation and verification

1. Take the hash the signer returned and read it back to a receipt — the same rule as
   every other write: prepare is not proof.
2. Read `CollectionCreated` from that receipt for the deployed address.
3. **Verify the source.** An unverified collection is a collection nobody can audit:
   submit the constructor arguments and the flattened source to the chain's explorer
   verification endpoint, and only report the deployment as complete once the explorer
   shows the source as verified.

## What exists today

- ERC-20 token launches on a bonding curve: *Signalbound Pump* (see `pump.md`).
- Minting the existing Signalbound collection: `allowlist-scan`, `drop-radar`,
  `portfolio` (see `tools.md`).
- Reading an Agent's identity and tools (see `agent-runtime.md`).

Nothing above deploys a user-defined ERC-721 collection.

## Build order

1. `SignalCollection` + `SignalCollectionFactory` with hardhat tests: supply cap,
   per-wallet cap, price accounting, immutability of the fixed fields, the
   `CollectionCreated` payload, and the `(creator, symbol)` uniqueness rule.
2. Deploy the factory to the chain named in production config and verify its source.
3. `POST /collections/prepare` in the API, returning the envelope above, `503` until
   the factory address is configured — the same pattern Pump uses for a switchable
   capability.
4. Update this file and the skill's *Deploying an NFT collection* section: the status
   line comes out, the factory address and the route go in, and `version` moves.
