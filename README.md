# signalbound-skill

The standing instruction file for Signalbound — what an agent runtime loads when it is
asked to launch a token, trade a market, task an Agent, read the collection, or buy
onchain intelligence.

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
| `SKILL.md` | The skill: origins, the write contract, Pump, x402 tools, Agent runtime, limits, failures, rules |
| `references/pump.md` | Full Pump endpoint, figure and limit tables |
| `references/tools.md` | x402 tool parameters and reply shapes |
| `references/x402-payment.md` | The 402 exchange in full |
| `references/agent-runtime.md` | Manifest, planner, MCP, holder-gated actions |
| `references/nft-collections.md` | The collection-deploy specification — **not yet served** |

## Keeping the two copies in step

`SKILL.md` in this repository and `website/skill.md` in the Signalbound monorepo are
the same document. Edit both, and move `version:` in the frontmatter when the content
changes: the version is what tells you which side is behind.

## What is deliberately absent

The skill does not describe an NFT collection-deploy endpoint, because there is not one.
`references/nft-collections.md` specifies the endpoint, the factory and the verification
step that will exist, and the skill says plainly that the capability is not served yet.
An agent should never be left to invent an endpoint that the platform does not run.

## Ground rules for edits

- No wallet or contract addresses in the skill: the chain, the contracts and the
  approved pairs come from `/api/config` on every run.
- No private keys, seed phrases or session tokens, and no instruction that would need
  one.
- Every figure in the file comes from the handler that enforces it, not from prose.
- No promotional language. A skill read by an agent that trades may not recommend,
  guarantee or forecast.
- A `200` from a prepare route is never described as a confirmation;
  `GET /api/receipt/:hash` is.

## License

CC0-1.0 — see `LICENSE`. Project-written public copy, dedicated to the public domain.
