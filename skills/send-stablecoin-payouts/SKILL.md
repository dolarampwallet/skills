---
name: send-stablecoin-payouts
description: "Send stablecoin payouts with the dolaramp API using POST /hotwallet/withdraw: idempotency keys, the 200 confirmed and 202 pending answers, withdraw.confirmed and withdraw.failed webhooks, insufficient_balance, allow_partial and the one-at-a-time rule on TRON. Use when: building withdrawals, mass payouts, an off-ramp or a payout queue on dolaramp, or handling a payout that timed out or failed. Also for requests in Portuguese or Spanish such as \"saque em USDT\", \"pagamento em massa em stablecoin\", \"retiros en USDT\", \"pagos masivos\"."
---

## Overview

A withdrawal sends a token from the hotwallet to any address. dolaramp sponsors the gas and takes its fee in stablecoin inside the transfer. Children must be swept first: only the hotwallet pays out.

Base URL `https://api.dolaramp.com/v1`, header `X-Api-Key` (an `operate` key). Amounts are strings in micro-units (6 decimals). The examples assume a `config` object with `apiKey` and `walletSecret`, loaded from your secret store on the server.

## Before sending

1. Validate the destination: `POST /address/validate` with `{ "address": "...", "network": "tron" }`. A valid address on the wrong network loses the funds.
2. Read the cost: `GET /quote?network=...`. `amount` is **gross**: the recipient receives `amount − fee`. To deliver an exact amount, add the quoted fee.
3. Generate an `idempotency_key` per payout and store it with the payout record **before** calling.

## Withdraw

`POST /hotwallet/withdraw`

```ts
const { status, body } = await dolaramp<WithdrawResponse>("/hotwallet/withdraw", {
  method: "POST",
  body: JSON.stringify({
    network: "solana",
    token: "USDT",
    to: "9xQeWvG816bUx9EPjHmaT23yvVM2ZWbrrpZb9PusVFin",
    amount: "10000000",
    wallet_secret: config.walletSecret,
    idempotency_key: "payout-inv-10432", // up to 64 chars
    external_ref: "user_100",            // optional, travels on the webhooks
  }),
});
```

## Read the answer by status

| HTTP | Meaning | What to do |
|---|---|---|
| `200` `status: confirmed` | Landed. `tx_hash`, `operation_id`, `recipient_receives` | Mark paid |
| `202` `status: pending` | Handed to the network, landing not seen yet. `operation_id`, `tx_hash` or (TRON) `trace_id` | Mark **in flight**. Wait for the webhook. Do not resend with a new key |
| `400 insufficient_balance` | The hotwallet cannot cover it. Nothing left | Top up or resize. On TRON the body has `balance`, `available`, `reserved_for_fees` |
| `400` other | `amount_too_small`, `amount_below_fee`, `invalid_recipient`, `unsupported_token_network_pair` | Fix the input |
| `402 reserved_for_fees` | TRON: the balance covers only fees owed | Top up |
| `409 op_in_progress` | Another operation holds this hotwallet | Retry shortly, same key |
| `502 withdraw_failed` | Refused by the rail after validation (`reason`). Nothing was sent | Retry with the same key |
| `503` | Sponsor or gasless rail unavailable | Retry later, same key |

**A `202` is never a failure.** The final outcome arrives, usually within a minute, as `withdraw.confirmed` or `withdraw.failed` with a `reason` (`not_landed`, `onchain_reverted`, `onchain_error`, `gasfree_failed`). After a `withdraw.failed`, nothing left the hotwallet and the same key may be used again.

## Idempotency

The key is reserved before anything is sent. Two calls with the same key produce one transfer; a retry returns the original withdrawal with `replayed: true` and its current `status`. This is what makes a timeout safe:

```ts
async function payout(p: Payout) {
  // p.idempotencyKey was stored when the payout was created
  for (let attempt = 0; attempt < 5; attempt++) {
    const { status, body } = await withdraw(p);
    if (status === 200) return markPaid(p, body.tx_hash);
    if (status === 202) return markInFlight(p, body.operation_id);
    if (status === 409 || status === 502 || status === 503) { await sleep(5000 * (attempt + 1)); continue; }
    return markRejected(p, body); // 400, 401, 402, 403
  }
  return markNeedsReview(p); // still unknown: reconcile via GET /operations, never resend with a new key
}
```

Never generate a fresh key for a retry of the same payout.

## Let the webhook settle the queue

Subscribe to `withdraw.confirmed` and `withdraw.failed`. `data` carries `operation_id`, `tx_hash`, `status`, `to`, `requested_amount`, `recipient_receives`, `fee_usdt`, your `idempotency_key` and `external_ref`. Match on `idempotency_key`. Reconcile with `GET /operations` as a safety net.

## TRON specifics

- **One pending transfer per hotwallet.** Firing the next payout right after the previous one is fine: the API waits for the previous transfer to clear (up to about 25 s). If it still cannot, the call fails with `502 withdraw_failed` / `reason: previous_transfer_pending`. Nothing was sent; retry with the same key a few seconds later. Process a TRON payout queue **serially**.
- **What can be sent** is `balance − accrued fees − network fee`. Size payouts from `available` in the error body, or from `GET /me` (`accrued_fees_usd`).
- **`allow_partial: true`** sends what fits instead of refusing. The response and the webhook then carry `recipient_receives` and `capped_for_fees: true`. Without the flag a payout is never sent partially. Only use it when the user explicitly wants "send what is available".

See the `usdt-tron-gasless` skill for the fee model.

## Common mistakes

- Treating `202` or a timeout as failed and sending again with a new key.
- Running TRON payouts in parallel.
- Assuming the recipient receives `amount`. They receive `amount − fee`.
- Paying out to an address without validating its network.
- Turning on `allow_partial` by default.
- Putting the `wallet_secret` in a job payload or a log line.
