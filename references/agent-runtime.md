# Signalbound Agent runtime

Each Signalbound NFT (1–6666) is an agent identity: an ERC-721 carrying an ERC-8004
discovery identity, an ERC-8217 token-bound controller, ERC-8257 holder-gated tools and
ERC-7496 clearance traits. The runtime layer below is what an orchestrator reads to
operate one.

All paths are on `https://api.signalbound.art`.

## Discovery

| Call | Answers |
| --- | --- |
| `GET /.well-known/agent-runtime.json` | whole-surface manifest: endpoints, tool list, tool rules |
| `GET /agents/{tokenId}/runtime.json` | one Agent's runtime card |
| `GET /agents/{tokenId}/agent.md` | tool rules |
| `GET /agents/{tokenId}/soul.md` | personality |
| `POST /agents/{tokenId}/plan` | maps a natural-language task to a tool |
| `POST /mcp` | MCP gateway for the same toolset |
| `GET /registry/tools` | tool registry |
| `GET /agents/info/{tokenId}` | identity, rarity, clearance, contract links |
| `GET /agents/{tokenId}/collection` · `GET /agents/{tokenId}/account` | holdings and balances |
| `GET /agents/{tokenId}/binding` · `GET /agents/{tokenId}/operator` | binding and Pump operator state |
| `GET /.well-known/skills/signalbound.md` | the runtime's own skill markdown |

## Operating flow

1. Load the runtime manifest, or go straight to the token's runtime card.
2. Read `agent.md` for tool rules and `soul.md` for personality.
3. For a natural-language task, `POST /agents/{tokenId}/plan` with `{"task": "…"}`.
4. Execute the tool the plan names — through a holder-authenticated flow when it is one
   of the ERC-8257 gated tools.
5. Return a concise brief. Expose raw JSON only when asked.

```bash
curl -X POST https://api.signalbound.art/agents/5742/plan \
  -H "Content-Type: application/json" \
  -d '{"task":"scan upcoming mints and tell me what my agent should watch"}'
```

The planner does not bypass gates and does not execute anything: it selects the route
and suggested parameters for an authenticated runtime.

## Holder-gated actions

Public reads are open. The actions that spend, mint or list need wallet-authenticated
proof that the caller controls the token:

| Call | Pair |
| --- | --- |
| `POST /agents/{tokenId}/actions/prepare-mint` | → `POST /agents/{tokenId}/actions/confirm-mint` |
| `POST /agents/{tokenId}/actions/prepare-listing` | → `POST /agents/{tokenId}/actions/submit-listing` |

Prepare answers `{ "transaction": { "to", "data", "value" } }`; sign it, broadcast it,
then follow the transaction hash to a receipt. Same three states as every other write:
`200` from prepare is a transaction you *may* sign, not one that happened.

## Safety

- Never request a private key or seed phrase, and never claim one is needed.
- Never bypass an ERC-8257 access gate or present holder control as valid before
  wallet-authenticated proof.
- Never promise profit, guaranteed mints or investment outcomes.
- Treat a `403` here as an access policy, not as a broken endpoint.
