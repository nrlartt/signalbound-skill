---
name: signalbound
description: |
  The Signalbound NFT collection skill. Deploy a fixed-supply ERC-721 collection — name,
  symbol, mint price, max supply, mints per wallet, creator reserve, royalties, hosted
  or on-chain art — from compiled bytecode on any EVM chain, then verify the contract on
  the explorer, mint from it, and read its state back. Use when a user asks to create or
  launch an NFT collection, verify a deployed contract, mint from a collection, or read
  what a collection contract says.
version: 2.0.0
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

# Signalbound collections

You can deploy a real ERC-721 collection for someone else: their name, their supply, their
mint price, their wallet owning it. You never compile anything — Signalbound publishes the
compiled contract, and you describe the collection and sign the deploy it hands back.

This file is canonical at `https://signalbound.art/skill.md`. A copy in the source
repository is a mirror; load the URL.

## Origins

| Origin | Carries |
| --- | --- |
| `https://signalbound.art` | The site, the docs, and this file |
| `https://api.signalbound.art` | `GET /collections/artifact`, `POST /collections/prepare` — both free, no key, no payment |

Two endpoints, that is the whole integration. Anything else claiming to be a Signalbound
endpoint for collections does not exist.

## Start here

| The user asks | You do |
| --- | --- |
| "create an NFT collection called …, price …, supply …" | *Deploying a collection* below |
| "…and verify the contract you deploy" | *Verifying the contract* — never skip it |
| "mint 5 from my collection" / "what's the supply so far" | *Minting and reading* |
| "the art isn't ready yet, add it later" | Deploy now (on-chain art), then `setBaseURI` for the reveal |
| "change the mint price / pause minting" | owner setters in *Minting and reading* |
| "deploy on Robinhood / Base / Ethereum" | look the chain up in *Chains*, pass `chainId` to prepare |

## The contract: SignalCollection

One flat Solidity file, no imports, compiled for `paris` so its bytecode runs on any EVM
chain a wallet can be asked to deploy to. Everything is set in the constructor and echoed
back to you before you sign.

| Constructor argument | Rule |
| --- | --- |
| `name` | 1–64 characters, required |
| `symbol` | 1–16 letters/digits; derived from the name when absent ("Hood Turtles" → `HOOD`) |
| `owner` | the collection owner's `0x` address — **pass the user's wallet**, see below |
| `maxSupply` | 1–1,000,000, immutable for the life of the contract |
| `mintPrice` | wei per mint; `0` makes it a free mint |
| `maxMintPerWallet` | 1–`maxSupply`, immutable; counts cumulative mints per address |
| `maxCreatorMints` | owner's free reserve, ≤ `maxSupply`, immutable; `0` = none |
| `baseURI` | hosted art, see *Art*; empty string = on-chain art |
| `contractURI` | collection-level metadata (OpenSea reads it); optional |
| `royaltyReceiver` + `royaltyBps` | EIP-2981, ≤ 10000 bps; defaults to the owner, `0` = off |
| `artStyle` + `artSeed` | on-chain art mode: `0` creature, `1` mosaic, `2` horizon |
| `chainId` | not a constructor argument — prepare echoes it so you deploy on the right chain |

**Ownership is a constructor argument on purpose.** An agent deploying through a factory
would otherwise become the owner, and no factory is involved here: the deploy comes from
the wallet the user signs with. After the deploy, read `owner()` back and confirm it is
the user's address before telling anyone the collection is theirs.

Live rules (settable by the owner later): `mintPrice`, `mintPaused`, `baseURI`,
`contractURI`, royalty, and ownership itself. Fixed rules (never change): `maxSupply`,
`maxMintPerWallet`, `maxCreatorMints`. An agent reading the contract back is the check —
do not trust the deploy report alone.

The sale itself:

- `mint(quantity)` — send **exactly** `mintPrice × quantity` in native value; any other
  amount reverts with `WrongPayment`, there is no refund path.
- `mintToCreator(quantity)` — owner only, free, counts against `maxCreatorMints`.
- `withdraw()` — owner only, all proceeds to the owner.
- Standard ERC-721 (`transferFrom`, `safeTransferFrom`, approvals, `tokenURI`) plus
  EIP-2981 `royaltyInfo` and ERC-165 including `0x2a55205a`.

## Art: hosted or on-chain

The user's art description decides which mode you use — and if you cannot produce the
art, say so instead of inventing a URL.

**Hosted metadata (`baseURI` given).** `tokenURI(id)` becomes `<baseURI><id>.json`, so the
files must be named `1.json`, `2.json`, … under the base URL. Any host works: your own
storage, IPFS (`ipfs://…/` is accepted), a static site. Use this mode when you can
generate or upload images. The base URI may be empty at deploy and set later with
`setBaseURI` — that is the reveal.

**On-chain art (no `baseURI`).** The contract renders the token itself: a 16×16 mirrored
pixel image, base64 SVG in `data:` URI, deterministic from `artSeed` and the token id.
Three styles: `0` creature (body, eyes, one of four extras, a pattern), `1` mosaic,
`2` horizon (gradient sky, sun, stars, hills). No hosting, no images to upload, works
immediately — and it is what you fall back to when the description ("32-bit sideways
walking turtles with traits and cool backgrounds") is beyond what you can render.

Honesty rule: if you are deploying in on-chain mode, tell the user the art is the
contract's generative render and that `setBaseURI` swaps in their own art later. A
baseURI that 404s is worse than an honest fallback.

## Deploying a collection

**Step 0 — the owner address.** Ask the wallet the deploy will be signed from (on Bankr:
the connected wallet, `GET /wallet/me`; in a terminal: the address you control). Collections
deployed through a factory **must** pass it explicitly or the factory owns the collection.

**Step 1 — prepare.** Describe the collection once:

```bash
curl -s -X POST https://api.signalbound.art/collections/prepare \
  -H 'content-type: application/json' \
  -d '{
    "name": "Hood Turtles",
    "owner": "0xTHE_DEPLOYER_WALLET",
    "chainId": 4663,
    "maxSupply": 2222,
    "mintPriceEth": "0.001",
    "maxMintPerWallet": 20,
    "maxCreatorMints": 20,
    "artStyle": "creature"
  }'
```

The reply carries `creationBytecode` (the compiled contract), `constructorArgs`
(ABI-encoded, ready), `deployData` (bytecode + arguments for a raw creation transaction),
`params` (every value echoed — read it and check it against what the user asked for),
`chainId`, and `verify` (the exact compiler settings). `POST` only; a missing field
answers `400` naming the field.

**Step 2 — deploy.** A creation transaction has **no `to`** and **no value** (the
constructor is not payable; gas comes from the deployer's wallet). Three ways to send it:

- **Bankr (chat):** ask it to deploy from compiled creation bytecode, passing
  `creationBytecode` and `constructorArgs`, no value attached, on `chainId`. Bankr runs
  this through its deterministic CREATE2 deployer with the constructor arguments
  ABI-encoded — which is exactly the shape prepare returned. Its raw
  `POST /wallet/submit` takes no creation transactions (it requires `to`), so deploy
  through the agent's deploy capability, not that route.
- **Your own tooling:** viem/ethers `deployContract({ abi, bytecode, args })` using the
  `abi` from the artifact endpoint, or a raw transaction with `to` omitted and
  `data = deployData`.
- **A chain CLI:** `cast create --create --data <deployData>`, or `forge create` against
  the source in the artifact.

**Step 3 — read it back.** `owner()`, `name()`, `symbol()`, `totalSupply()`,
`maxSupply()`, `mintPrice()`, `maxMintPerWallet()`, `maxCreatorMints()`, `baseURI()`,
`artStyle()`. Confirm the owner is the user's wallet and the rules match the request.
This, plus a receipt with `status: success`, is the only proof the deploy happened — an
HTTP `200` is not.

**Step 4 — report.** The address, the explorer link, verification status, and the mint
instruction: price per mint, supply, per-wallet cap, creator reserve.

## Verifying the contract

Verification is part of the deploy, not an optional extra. The pack carries everything a
block explorer asks for:

| Field | Where it comes from |
| --- | --- |
| Source | `source` in `GET /collections/artifact` — the complete file, no imports to reconstruct |
| Compiler version | `verify.compiler`, e.g. `0.8.26+commit.8a97fa7a` |
| Optimization | `verify.optimizerRuns` — `200`, enabled |
| EVM version | `verify.evmVersion` — `paris`; do not accept a tool's "default" |
| Constructor arguments | `verify.constructorArgs` — already ABI-encoded, paste as hex |

With Bankr: "Verify contract <address> on <chain> with this source, compiler …, optimizer
runs 200, EVM version paris, constructor args …". On an explorer form: single-file
Solidity, paste the source, pick the version, optimization yes / runs 200, EVM version
paris, paste the constructor arguments. Re-verification with different settings fails
loudly — re-read `verify`, do not guess.

If verification is refused, the deploy is still live and still owned by the user: report
the address as deployed, unverified, with what you tried. Never claim verified when it
is not.

## Minting and reading

Mint for a user (or yourself) by calling the contract — the value must match exactly:

| Call | Value to attach |
| --- | --- |
| `mint(1)` | one mint price |
| `mint(5)` | five × mint price |
| `mintToCreator(n)` | none — owner only, ≤ remaining reserve |

Reads need no transaction and any agent can phrase them: "Read `totalSupply()` on
<address>", "Read `owner()` on <address>", "Read `minted(<address>)` on <address>",
"Read `royaltyInfo(1, 1000000000000000000)`". The state worth reading back after any
owner action: `mintPrice()`, `mintPaused()`, `baseURI()`, `contractURI()`, `royaltyBps()`.

Owner actions (signer must be the owner, else `NotOwner`): `setBaseURI`, `setContractURI`,
`setMintPrice`, `setMintPaused(bool)`, `setRoyalty(receiver, bps)`, `withdraw()`,
`transferOwnership(to)`. The caps never change.

## Chains

The contract deploys on any EVM chain whose hardfork includes `paris` (no PUSH0, no
MCOPY). The chain must still be one your wallet can sign for. Bankr's deploy surface, as
an example of a wallet that supports this flow end to end:

| Chain | chainId | | Chain | chainId |
| --- | --- | --- | --- | --- |
| Ethereum | 1 | | Arbitrum | 42161 |
| Polygon | 137 | | BNB Chain | 56 |
| Base | 8453 | | Robinhood Chain | 4663 |
| Unichain | 130 | | Arc | 5042 |
| World Chain | 480 | | | |

Pass the `chainId` to prepare so the reply is checked against where you will deploy.
Deploying on a chain the user did not name is a bug: ask, or follow what they said
("on Robinhood" → 4663).

## Limits

| Thing | Limit |
| --- | --- |
| Name / symbol | 64 characters / 1–16 letters and digits |
| `maxSupply` | 1–1,000,000 |
| `maxMintPerWallet` | 1–`maxSupply`, cumulative |
| `maxCreatorMints` | ≤ `maxSupply` |
| Royalty | 0–10000 bps, receiver required above 0 |
| `mintPrice` | below 1,000,000 ETH (sanity check) |
| Art styles | 0 creature, 1 mosaic, 2 horizon |
| Deploy bytecode | 17.9 KB creation, 15.6 KB runtime — under the 24 KB code-size limit |
| Prepare | `POST`, free, no key; `400` names the offending field |

## Prompt examples

> Hey @bankrbot can you create an NFT collection called Hood Turtles on Robinhood with the
> art being 32 bit sideways walking turtles with a bunch of different traits and cool
> backgrounds? Mint price of 0.001 ETH, max supply of 2222, max mints per wallet of 20, and
> max creator mints of 20. Use this skill for creating NFT collections
> https://signalbound.art/skill.md — Make sure to verify the contract you deploy as well

What that becomes: prepare with `{name:"Hood Turtles", chainId:4663, maxSupply:2222,
mintPriceEth:"0.001", maxMintPerWallet:20, maxCreatorMints:20, artStyle:"creature"}` (symbol
derived `HOOD`, owner = the signing wallet) → deploy with no value → read back the rules →
verify with the pack's fields → report the address, the explorer link, and that the art is
the contract's generative render, swappable for their own via `setBaseURI`.

| What a user says | What you do |
| --- | --- |
| "Create a collection called Pixie Market on Base, free mint, 1000 supply, no creator reserve" | prepare with `mintPriceEth:"0"`, `maxSupply:1000`, `maxCreatorMints:0`, `chainId:8453`, owner = wallet → deploy → verify → report |
| "…and verify it" | verification fields from `verify`; never skip, never guess them |
| "Mint 3 for me" | `mint(3)` with exactly 3 × mint price attached |
| "What has minted so far?" | read `totalSupply()` and `minted(<address>)` |
| "Art is ready now, hosted at https://cdn.example.com/turtles/" | owner calls `setBaseURI("https://cdn.example.com/turtles/")`; files must be `1.json`, `2.json`, … |
| "Make it 0.002 ETH now" | owner calls `setMintPrice`; the supply caps cannot change |
| "Pause minting while we fix the site" | owner calls `setMintPaused(true)` |
| "Who owns it?" | read `owner()` — if it is not the user's wallet, say so plainly |

## When a call fails

| Symptom | Meaning | What to do |
| --- | --- | --- |
| `400` with `field` | prepare rejected a value | the message names the field and the rule; fix that field only |
| `405` | wrong method | prepare is `POST`; the artifact is `GET` |
| `503` "artifact not built" | our side has not published the bytecode | say so; do not substitute another contract |
| `WrongPayment` | value ≠ price × quantity | recompute from `mintPrice()` read on chain |
| `PerWalletCapExceeded` | order above the cap, or the wallet's cumulative mints hit it | read `minted(<address>)`; the cap is fixed |
| `SoldOut` | `totalSupply` + order > `maxSupply` | read `totalSupply()`; nothing to do |
| `MintingIsPaused` | owner paused minting | read `mintPaused()` and tell the user |
| `CreatorCapExceeded` | reserve spent, or caller is not the owner | read `creatorMinted()` |
| `NotOwner` | signer is not the owner | check `owner()`; ownership is a constructor argument, not a guess |
| `ERC721NonexistentToken` | token id does not exist | ids start at 1 and only exist after mint |
| Explorer rejects verification | settings mismatch | re-read `verify` — version, runs 200, `paris`, encoded args |
| Deploy succeeded, `owner()` is not the user | the factory became owner | stop, say what happened, redeploy passing `owner` |
| Out of gas / insufficient funds | deployer pays the deploy gas | the user's wallet needs the chain's native token |

## For humans

| Where | Address |
| --- | --- |
| This skill | `https://signalbound.art/skill.md` |
| Site and docs | `https://signalbound.art` · `https://signalbound.art/docs` |
| Deploy pack | `https://api.signalbound.art/collections/artifact` |
| Prepare endpoint | `https://api.signalbound.art/collections/prepare` (POST) |
| Source and references | `https://github.com/nrlartt/signalbound-skill` |

## References

- [collection.md](https://raw.githubusercontent.com/nrlartt/signalbound-skill/main/references/collection.md) — constructor, functions, errors and both endpoints in full
- [deploy.md](https://raw.githubusercontent.com/nrlartt/signalbound-skill/main/references/deploy.md) — the deploy recipes: Bankr, viem, cast/forge, and how to confirm one
- [verification.md](https://raw.githubusercontent.com/nrlartt/signalbound-skill/main/references/verification.md) — every explorer field, plus the failure modes

## Rules

- Never send a private key, seed phrase or session token anywhere in this flow. Not to the
  endpoints, not to a deploy, not in chat. A deploy is signed by the wallet that owns it.
- The owner is the user's wallet, never yours, never a factory's. Read `owner()` after
  every deploy and say what it answers.
- Every value you sign comes from prepare's echo (`params`, `constructorArgs`). Do not
  re-derive, round or hand-carry numbers between calls.
- Show the intent before asking for a signature: name, symbol, supply, price, caps,
  royalty, chain, owner — the reply already lists them.
- A deploy is real when a receipt says success **and** the read-backs match. An HTTP `200`
  is a transaction you may send, not one that happened.
- Verify the contract you deploy. If verification fails, report the address as unverified
  with what you tried — never as verified.
- Use only the two endpoints in this file. Do not invent one, and do not quietly
  substitute a third-party deployer or collection contract when ours answers `503`.
- Art honesty: hosted art needs files that actually exist at `<baseURI><id>.json`; when
  you cannot produce the user's art, deploy on-chain and say exactly that.
- Nothing here is investment advice. A collection's mint, floor and royalties can go to
  zero, and every figure above is a reading, not a promise.
