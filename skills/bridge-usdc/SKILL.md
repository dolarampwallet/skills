---
name: bridge-usdc
description: "Move USDC between Solana, Base and Arbitrum with the dolaramp API: GET /bridge/quote, POST /bridge, the accepted then bridge.confirmed pattern, fees and the minimum, rebalancing your own hotwallets or paying a third party on another network. Use when: rebalancing USDC across networks, paying a supplier on a different chain than the one the customer paid on, or integrating cross-chain USDC transfers on dolaramp."
---

## Overview

The bridge moves **USDC** from your hotwallet on one network to an address on another. It runs on Circle's CCTP: USDC is burned on the origin chain and minted by Circle on the destination chain. The asset is the same on both sides, so this is a network move, not a swap: no pool and no slippage.

Routes: every pair of `solana`, `base` and `arbitrum`, in both directions. USDC only.

Base URL `https://api.dolaramp.com/v1`, header `X-Api-Key` (an `operate` key). Amounts are strings in micro-units (6 decimals). The examples assume a `config` object with `apiKey` and `walletSecret`, loaded from your secret store on the server.

## Quote

`GET /bridge/quote?from=base&to=solana&amount=100000000`

The answer itemizes the operation:

- `our_fee_usd` and `our_fee_breakdown` (`flat_usd` + `percent_usd`): the dolaramp fee, a flat amount plus a percentage that depends on the plan.
- `network_fee_usd`: Circle's fee for the fast mode, passed through unchanged.
- `recipient_receives_usd`: what lands on the destination.
- `fee.minimum_usd` and `below_minimum: true` when the amount is under the minimum. `POST /bridge` refuses amounts below it.
- `modes`: the estimated time. Only `fast` executes today.

Omit `amount` to get the fee schedule alone. Always show the user `recipient_receives_usd` before executing. Do not hardcode the fee or the minimum; read them from the quote.

`503 route_fees_unavailable` means Circle's fee API was unreachable. Retry.

## Execute

`POST /bridge`

```ts
const { status, body } = await dolaramp<BridgeResponse>("/bridge", {
  method: "POST",
  body: JSON.stringify({
    from: "base",
    to: "arbitrum",
    amount: "50000000",
    wallet_secret: config.walletSecret,
    idempotency_key: "rebalance-2026-10-03-01",
    // to_address: "0x...",   // omit to land on your own hotwallet on `to`
    // external_ref: "user_100",
  }),
});
```

- Funds leave the **hotwallet** on `from`. Sweep children first.
- Without `to_address` the USDC lands on your own hotwallet on `to` (rebalancing). With it, the USDC lands on that address (a cross-chain payout). Validate a third-party address with `POST /address/validate` for the destination network first.
- On Solana the destination token account is created automatically if missing.
- `idempotency_key` makes a retry replay the outcome instead of bridging twice.

## The answer is `accepted`, not final

A successful call returns `status: "accepted"` with `operation_id`, `burn_tx`, `rail`, `recipient`, `recipient_receives_estimate`, `our_fee_usd` and `network_fee_usd`. The burn has landed; the mint is pending.

Confirmation arrives as:

- webhook `bridge.confirmed` (or `bridge.failed`), and
- `GET /operations` going from `pending` to `confirmed`.

```ts
if (status === 200 && body.status === "accepted") {
  await markBridgeInFlight(body.operation_id, body.burn_tx);
  // settle on the bridge.confirmed webhook
}
```

A burn always ends in a mint: attestations do not expire, so a slow transfer is delayed, not lost. Do not re-send a bridge that is in flight.

## Errors

| HTTP | Code | Meaning |
|---|---|---|
| `400` | `same_network`, `below_minimum` (carries `minimum`), `insufficient_balance`, `invalid_recipient`, `no_hotwallet` | Fix the request |
| `401` | `invalid_wallet_secret` | Wrong secret |
| `409` | `op_in_progress` | Another operation holds this hotwallet. Retry shortly, same key |
| `502` | `bridge_failed` | Did not go through (`stage` says where). Check `GET /operations` before retrying |
| `503` | `sponsor_out_of_gas`, `relayer_not_configured`, `route_fees_unavailable` | Retry later, same key |

## When to use it

- A customer paid in USDC on Solana and a supplier invoices on Arbitrum.
- Hotwallet balances drifted and one network needs funding for payouts.

For USDT, or for moving value into or out of TRON and TON, there is no bridge route: use the networks' own deposits and withdrawals.

## Common mistakes

- Bridging an amount below the minimum without checking the quote.
- Treating `accepted` as settled.
- Retrying a timed-out call without the same `idempotency_key`.
- Passing `mode: "standard"`. Only `fast` executes; anything else is refused with `400`.
- Trying to bridge USDT or PYUSD.
- Forgetting that the EVM hotwallet has one address on both Base and Arbitrum: the default destination between them is the same `0x` address on the other network.
