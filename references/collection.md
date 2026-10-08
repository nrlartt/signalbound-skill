# SignalCollection: the contract and the two endpoints

Everything the skill compresses, in full: the constructor, every function, every error,
the art modes, and the exact request/response shapes of
`GET /collections/artifact` and `POST /collections/prepare`.

The contract is one flat Solidity file with **no imports**. Published source, ABI and
creation bytecode all come from the artifact endpoint, so nothing here needs a compiler
to verify or a dependency tree to reconstruct.

## Constructor

Thirteen arguments, in this order. `POST /collections/prepare` ABI-encodes them for you;
this table is what the encoded bytes mean.

| # | Argument | Type | Rule |
| --- | --- | --- | --- |
| 0 | `name` | `string` | 1–64 characters, required |
| 1 | `symbol` | `string` | 1–16 letters/digits; prepare derives it from the name when absent |
| 2 | `owner` | `address` | The collection owner. Zero address ⇒ contract takes the deploying account (`msg.sender`) — fine for a direct wallet deploy, wrong through a factory |
| 3 | `maxSupply` | `uint256` | 1–1,000,000. Immutable |
| 4 | `mintPrice` | `uint256` | Wei per public mint; `0` = free mint. Mutable by the owner |
| 5 | `maxMintPerWallet` | `uint256` | 1–`maxSupply`. Immutable. Cumulative across a wallet's lifetime |
| 6 | `maxCreatorMints` | `uint256` | ≤ `maxSupply`. Immutable. Owner's free reserve; `0` = none |
| 7 | `baseURI` | `string` | Hosted art; empty ⇒ on-chain art. Must end with `/` — prepare appends it |
| 8 | `contractURI` | `string` | Collection-level metadata (OpenSea reads it); may be empty |
| 9 | `royaltyReceiver` | `address` | EIP-2981 receiver. Required when `royaltyBps > 0` |
| 10 | `royaltyBps` | `uint96` | 0–10000 (10000 = 100%). `0` disables royalties |
| 11 | `artStyle` | `uint8` | `0` creature, `1` mosaic, `2` horizon — only used when `baseURI` is empty |
| 12 | `artSeed` | `uint256` | Seed for the on-chain render; prepare derives one from the name when absent |

Reverts at construction: `ZeroSupply`, `ZeroWalletCap`, `CreatorCapAboveSupply`,
`UnknownArtStyle`, `InvalidRoyalty` (bps > 10000, or bps > 0 with a zero receiver).

The constructor is **not payable** — it takes no value, so the creation transaction
carries only gas.

## Read functions

No transaction needed; an agent can phrase any of these as "Read `f()` on `<address>`".

| Returns | Function |
| --- | --- |
| `string` | `name()`, `symbol()`, `baseURI()`, `contractURI()` |
| `address` | `owner()`, `royaltyReceiver()` |
| `uint256` | `totalSupply()`, `maxSupply()`, `mintPrice()`, `maxMintPerWallet()`, `maxCreatorMints()`, `creatorMinted()`, `artSeed()`, `minted(address)` |
| `uint8` | `artStyle()` |
| `uint96` | `royaltyBps()` |
| `bool` | `mintPaused()` |
| `string` | `tokenURI(uint256)` — hosted `…/<id>.json`, or the on-chain `data:` SVG |
| `address, uint256` | `royaltyInfo(uint256 tokenId, uint256 salePrice)` |
| `bool` | `supportsInterface(bytes4)` — `0x01ffc9a7`, `0x80ac58cd`, `0x5b5e139f`, `0x2a55205a` |
| `uint256`, `address` | `balanceOf(address)`, `ownerOf(uint256)` |
| `address`, `bool` | `getApproved(uint256)`, `isApprovedForAll(address, address)` |

## State-changing functions

| Call | Who | Value | Effect |
| --- | --- | --- | --- |
| `mint(quantity)` | anyone | **exactly** `mintPrice × quantity` | Public mint, ids sequential from 1. Checks in order: not paused, `1 ≤ quantity ≤ maxMintPerWallet`, supply fits, wallet's cumulative mints fit, exact payment |
| `mintToCreator(quantity)` | owner | none | Free mint against `maxCreatorMints`, drawn from the same `maxSupply` |
| `setBaseURI(uri)` | owner | — | Switches `tokenURI` to hosted mode — this is the reveal |
| `setContractURI(uri)` | owner | — | Collection metadata |
| `setMintPrice(wei)` | owner | — | Reprice; caps never change |
| `setMintPaused(bool)` | owner | — | Halt/resume the public mint |
| `setRoyalty(receiver, bps)` | owner | — | EIP-2981 default royalty; `bps = 0` turns it off |
| `withdraw()` | owner | — | Entire balance to the owner |
| `transferOwnership(to)` | owner | — | `to` must not be the zero address |
| `transferFrom`, `safeTransferFrom` ×2, `approve`, `setApprovalForAll` | standard ERC-721 rules | — | Approvals cleared on transfer |

Supply counters are written **before** `_safeMint` calls into a receiver contract, so a
reentrant receiver sees the caps it just spent.

## Events

`Transfer`, `Approval`, `ApprovalForAll`, `OwnershipTransferred`, `Minted(minter, quantity)`,
`CreatorMinted(receiver, quantity)`, `BaseURIUpdated`, `ContractURIUpdated`,
`MintPriceUpdated`, `MintPausedSet`, `RoyaltyUpdated`, `Withdrawn(to, amount)`.

## Errors

| Error | When | Tell the user |
| --- | --- | --- |
| `WrongPayment(expected, sent)` | value ≠ `mintPrice × quantity` | Recompute from `mintPrice()`; there is no refund path |
| `PerWalletCapExceeded(cap)` | one order above the cap, or cumulative mints hit it | Read `minted(<address>)`; the cap is immutable |
| `SoldOut` | `totalSupply + quantity > maxSupply` | Read `totalSupply()` |
| `MintingIsPaused` | owner paused | Read `mintPaused()` |
| `CreatorCapExceeded(cap)` | reserve empty/zero | Read `creatorMinted()` |
| `NotOwner` | caller isn't the owner | Read `owner()` |
| `ZeroOwner` | `transferOwnership(0)` | Ownership cannot be renounced to zero |
| `ZeroSupply` / `ZeroWalletCap` / `CreatorCapAboveSupply` / `UnknownArtStyle` / `InvalidRoyalty` | constructor/royalty validation | Fix the field prepare named |
| `NothingToWithdraw` / `WithdrawFailed` | empty balance / rejected transfer | Check the balance or the owner's ability to receive |
| `ERC721InvalidOwner` | `balanceOf(0)` | — |
| `ERC721NonexistentToken` | id not minted | Ids start at 1 and exist only after mint |
| `ERC721IncorrectOwner` | `transferFrom` from the wrong owner | Read `ownerOf(id)` |
| `ERC721InsufficientApproval` | caller not owner/approved/operator | Read `getApproved(id)` |
| `ERC721InvalidApprover` | `approve` to self or by non-authorized | — |
| `ERC721InvalidOperator` | `setApprovalForAll(0, …)` | — |
| `ERC721InvalidReceiver` | transfer/mint to zero, or a contract that rejects ERC-721 | — |

## Art modes

**Hosted (constructor `baseURI` non-empty).**
`tokenURI(id) = baseURI + id + ".json"` — files must be `1.json`, `2.json`, … under the
base URL. Any `http`, `https` or `ipfs` base works. `setBaseURI` switches a live contract
into (or out of) hosted mode, which is how a reveal is done.

**On-chain (`baseURI` empty).** The contract renders a 16×16 mirrored pixel image as
`data:image/svg+xml;base64,…`. Traits come from `keccak256(artSeed, tokenId)`, so a token
always renders the same bytes. Colour is computed as integer HSL — identical on every
chain and compiler patch level.

| `artStyle` | Name | Composition |
| --- | --- | --- |
| 0 | creature | Complementary ground, wide oval body, pattern (plain/stripes/spots), 2×2 eyes with pupil, mouth, and one of four extras: antenna, horns, ears, none |
| 1 | mosaic | Four hues of one palette over a dark ground, per-cell hash scatter with gaps |
| 2 | horizon | Gradient sky, centre sun, scattered stars, ground with a lit edge and a centre hill |

Mirroring: a cell's colour depends only on its distance from the vertical axis, so both
halves are always identical.

## GET /collections/artifact

Free, `GET` or `HEAD`, `Cache-Control: public, max-age=300`, CORS `*`.

| Field | Content |
| --- | --- |
| `contract` | `"SignalCollection"` |
| `sourceName` | `contracts/SignalCollection.sol` |
| `source` | The complete source file — byte-identical to the repository's |
| `compiler.version` / `compiler.longVersion` | `0.8.26` / `0.8.26+commit.8a97fa7a` |
| `compiler.settings` | `{ optimizer: { enabled: true, runs: 200 }, viaIR: true, evmVersion: "paris" }` |
| `abi` | Full ABI, including the constructor |
| `constructorInputs` | Constructor parameter list (what `prepare` encodes against) |
| `creationBytecode` | `0x…` — 17,903 bytes of deploy code |
| `deployedBytecodeSize` | `15645` — runtime under the 24,576-byte code-size limit |

Status: `200` · `405` (other methods) · `503` `{ error: "Collection artifact not built" }`
when the artifact has not been built.

## POST /collections/prepare

Free, `POST` only, body `application/json`, `Cache-Control: no-store`, CORS `*`.

### Request

| Field | Type | Required | Default / rule |
| --- | --- | --- | --- |
| `name` | string | yes | 1–64 chars |
| `symbol` | string | no | first 4 letters/digits of the name, uppercased (`"Hood Turtles"` → `HOOD`) |
| `owner` | `0x` address | no* | zero ⇒ contract takes `msg.sender`; *required in practice when deploying through a factory, and required if you want royalties without naming a receiver |
| `chainId` | integer ≥ 1 | no | echoed back so the deploy is checked against it |
| `maxSupply` | integer or string | yes | 1–1,000,000 |
| `mintPriceEth` | decimal string | no | e.g. `"0.001"`; ignored if `mintPriceWei` is present |
| `mintPriceWei` | integer or string | no | `0` |
| `maxMintPerWallet` | integer or string | no | `maxSupply` |
| `maxCreatorMints` | integer or string | no | `0` |
| `baseURI` | URL | no | must start `http://`, `https://` or `ipfs://`; trailing `/` appended |
| `contractURI` | URL | no | same schemes, no trailing rule |
| `royaltyReceiver` | `0x` address | no | defaults to `owner`; with neither, a nonzero `royaltyBps` is a `400` |
| `royaltyBps` | integer or string | no | `500` (5%); 0–10000 |
| `artStyle` | `0`–`2` or `creature` / `mosaic` / `horizon` | no | `0` — only used in on-chain mode |
| `artSeed` | integer or string | no | derived from `keccak256(name)` so the same request always deploys the same art |

### Response (`200`)

```jsonc
{
  "ok": true,
  "contract": "SignalCollection",
  "chainId": 4663,                       // or null when none was given
  "creationBytecode": "0x…",
  "constructorArgs": "0x…",              // the 13 arguments, ABI-encoded
  "deployData": "0x…",                   // creationBytecode + constructorArgs
  "params": {                            // echo of everything encoded — read it back
    "name": "Hood Turtles",
    "symbol": "HOOD",
    "owner": "0x…",                      // or null + ownerNote when not given
    "ownerNote": "…",
    "maxSupply": "2222",
    "mintPrice": { "wei": "1000000000000000", "eth": "0.001" },
    "maxMintPerWallet": "20",
    "maxCreatorMints": "20",
    "contractURI": "",
    "royalty": { "receiver": "0x…", "bps": "500" },
    "art": { "mode": "onchain", "baseURI": null, "style": 0, "styleName": "creature", "seed": "…" }
  },
  "compiler": { /* as the artifact */ },
  "verify": {
    "compiler": "0.8.26+commit.8a97fa7a",
    "optimizerRuns": 200,
    "evmVersion": "paris",
    "viaIR": true,
    "constructorArgs": "0x…",
    "sourceAt": "https://api.signalbound.art/collections/artifact — field `source`"
  },
  "artifactUrl": "https://api.signalbound.art/collections/artifact"
}
```

### Status codes

| Code | Body | Meaning |
| --- | --- | --- |
| `200` | as above | ready to deploy |
| `400` | `{ error, field }` | the named field failed validation; fix exactly that field |
| `400` | `{ error: "Body must be valid JSON" }` | unparseable body |
| `405` | `{ error }` + `Allow: POST` | not a `POST` |
| `503` | `{ error: "Collection artifact not built", build: … }` | our side has not published the bytecode — never substitute another contract |

## Limits

| Thing | Limit |
| --- | --- |
| Name / symbol | 64 chars / 1–16 letters+digits |
| `maxSupply` | 1–1,000,000 |
| `maxMintPerWallet` | 1–`maxSupply`, cumulative |
| `maxCreatorMints` | ≤ `maxSupply` |
| Royalty | 0–10000 bps, receiver required above 0 |
| `mintPrice` | < 1,000,000 ETH (sanity ceiling) |
| Art styles | 0, 1, 2 |
| Creation code | 17,903 bytes (≈36 KB hex) |
| Runtime code | 15,645 bytes / 24,576 limit |
