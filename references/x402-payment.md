# Paying for a call

These endpoints are metered with x402: the price travels in the HTTP response, so no
account, key or subscription exists to set up. An unpaid call is not an error — it is
the price list.

Price list: `GET https://api.signalbound.art/.well-known/x402`
Schemas: `GET https://api.signalbound.art/openapi.json`

## The exchange

1. **Call the tool.** `POST` the JSON body you want.
2. **Read the 402.** The reply carries the terms in the body and, base64-encoded, in
   the `PAYMENT-REQUIRED` header:

   ```json
   {
     "x402Version": 2,
     "resource": { "url": "…/drop-radar", "description": "…", "mimeType": "application/json" },
     "accepts": [ { "scheme": "…", "network": "…", "asset": "…", "amount": "…", "payTo": "…" } ]
   }
   ```

3. **Choose a price.** Two options are always published:

   - the **open rate**, first in the list, which anyone can pay — no NFT, no account;
   - the **discounted rate**, which only settles for a wallet that holds a Signalbound
     NFT meeting that tool's tier. Signing it without one gets the payment refused and a
     fresh 402 offering only the open rate.

   Amounts are atomic units of the named `asset` on the named `network`. Read the
   asset's own precision from the challenge rather than assuming six decimals.

4. **Sign it** with a wallet that supports x402, then repeat the same call with the
   resulting value in the `X-PAYMENT` header.
5. **Get the result plus a receipt.** The response carries `X-Payment-Response` (the
   settlement, base64 JSON), `X-Payment-Tier` (`open` or `holder`) and
   `X-Payment-Facilitator` (which service settled it).

## What this skill does and does not do

- It **does** call the endpoint, read the terms, print them, and forward a payment you
  hand it.
- It **does not** hold keys, sign authorizations, choose a price, or retry a call with
  money.

That line is deliberate. An agent that could pay without being asked is an agent that
can spend without being asked.

## Failure modes worth telling apart

| What you see | What it means |
|--------------|---------------|
| 402 with both rates | Nothing was charged; you have not paid yet |
| 402 after paying, listing only the open rate | The discounted rate was signed by a wallet that does not hold a qualifying NFT |
| 402 saying the payment was not accepted | The settlement service refused it — the message names the reason. Verify on chain before signing a new one |
| Result with `X-Payment-Response` | Settled. The receipt carries the transaction |
| 403 | The caller does not satisfy the tool's access policy |
| 5xx | An upstream data source failed; retry rather than re-paying blindly |

Each authorization is single-use, so repeating one cannot double-charge — but signing a
*new* one for a call that already settled can. When a call is ambiguous, check the
transaction from the receipt before paying again.

## Standard

Protocol: [x402](https://x402.org). The challenge is published in the v2 wording and is
readable by any x402 client — including agent wallets that pay 402s on their own.
