# signalbound-skill

The standing instruction file for Signalbound NFT collections — what an agent runtime
loads when it is asked to deploy an ERC-721 collection from compiled bytecode, verify it
on the explorer, mint from it, or read what its contract says. Every token it deploys is
an agent NFT: an ERC-8004 registration document with an optional RESTAP endpoint, plus
an ERC-8048 (ERC-721T) agent profile readable on-chain, and mints and transfers post
themselves to Net Protocol.

**Canonical URL: <https://signalbound.art/skill.md>**

That URL is served by the Signalbound site itself and is the copy to point a skill
loader at. The file in this repository, `SKILL.md`, is its mirror; the deeper documents
under `references/` live only here and are linked from it over
`raw.githubusercontent.com`.

## Install

If your runtime loads a skill by URL, point it at <https://signalbound.art/skill.md>.

Otherwise clone this repository and save `SKILL.md` into your runtime's skills
directory, e.g. `~/.claude/skills/signalbound.md`.

## Layout

| File | What it holds |
| --- | --- |
| `SKILL.md` | The skill: the two endpoints, the contract's rules, the deploy/verify/mint/read flow, art modes, agent NFT registration (ERC-8004 + ERC-8048), Net Protocol posting, limits, failures, rules |
| `references/collection.md` | Constructor, functions, errors, art modes, the agent registration and profile, Net Protocol, and both endpoint request/response shapes in full |
| `references/deploy.md` | The creation transaction in four shapes — Bankr, viem/ethers, chain CLI — plus the confirmation checklist |
| `references/verification.md` | Every explorer field, the form walkthrough, failure modes |

## The capability

`POST https://api.signalbound.art/collections/prepare` encodes the constructor for a
described collection; `GET https://api.signalbound.art/collections/artifact` publishes
the compiled contract (creation bytecode, ABI, exact compiler settings, complete
source). Both are free and need no key. The deploy itself is signed by the wallet that
owns the collection — this repository never holds, transmits or requests a key.

## Keeping the two copies in step

`SKILL.md` in this repository and `website/skill.md` in the Signalbound monorepo are the
same document (byte-identical). Edit both, and move `version:` in the frontmatter when
the content changes: the version is what tells you which side is behind.

## What is deliberately absent

The Pump launchpad, the x402 paid tools, and the Agent runtime are not part of this
skill — its scope is NFT collections only. There are also no wallet or contract
addresses in it: the owner comes from the wallet that signs, the chain from the
user's request, the rules from reading the contract back.

## Ground rules for edits

- No wallet or contract addresses in the skill or its references: an agent reads state,
  it does not trust a pasted constant.
- No private keys, seed phrases or session tokens, and no instruction that would need
  one — a deploy is signed by the wallet that owns the collection.
- Every figure comes from the handler or the contract that enforces it, not from prose.
- No promotional language. A skill read by an agent that trades may not recommend,
  guarantee or forecast.
- An HTTP `200` is never described as a confirmation; a success receipt plus the
  read-backs (`owner()` and the rules) are.
- A deploy is unfinished until it is verified — and if verification fails, the skill
  says to report it as unverified, never as verified.

## License

CC0-1.0 — see `LICENSE`. Project-written public copy, dedicated to the public domain.
