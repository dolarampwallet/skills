---
name: sweep-funds
description: "Consolidate dolaramp child wallets into the hotwallet with POST /sweep: batch sweeps, the fee quote, idempotency keys, per-child results, and the asynchronous accepted then sweep.confirmed pattern on TRON. Use when: moving deposited funds from deposit addresses to the hotwallet, scheduling consolidation, handling below_min_sweep, or reading sweep results and fees."
---

## Overview

Deposits land in child wallets. A **sweep** moves the balance of one token from children into the hotwallet, in batch. dolaramp sponsors the gas; the fee is taken in stablecoin inside the transaction. A child can only ever be swept into its own hotwallet.

Base URL `https://api.dolaramp.com/v1`, header `X-Api-Key` (an `operate` key). Amounts are strings in micro-units (6 decimals).

## Quote first

`GET /quote?network=tron` returns what one sweep or withdraw costs right now:

- `typical.fee_usdt`: the dolaramp fee.
- `typical.network_fee_usdt`: TRON only. Charged by the network inside the transfer, separate from the fee.
- `worst_case_new_accounts`: the same for a first-time address (token account creation on Solana, account activation on TRON).
- `min_sweep_usdt`: balances below this are skipped.

Do not hardcode fees. Read them from the quote.

## Sweep

`POST /sweep`

```ts
const { body } = await dolaramp<SweepResponse>("/sweep", {
  method: "POST",
  body: JSON.stringify({
    network: "solana",
    token: "USDT",
    wallet_secret: process.env.DOLARAMP_WALLET_SECRET,
    child_addresses: ["ANjK0x...ktZ"],          // optional, up to 20
    idempotency_key: "sweep-2026-10-03-batch-7", // up to 64 chars
  }),
});
```

- Without `child_addresses` the call scans the 20 most recent children. To sweep specific customers, pass their addresses (up to 20 per call) and loop.
- `idempotency_key` makes a retry safe: the same key returns the original batch with `replayed: true` and moves nothing.
- `409 op_in_progress`: another sweep or withdraw is running for that address. Retry shortly.
- `401 invalid_wallet_secret`; repeated wrong secrets lock the account for a while (`429 wallet_secret_locked`, `retry_in_s`).

## Read the results

The response carries `batch_id`, `hotwallet` and a `results` array, one entry per child:

```json
{ "child": "ANjK0x...ktZ", "swept": "12500000", "tx_hash": "3xR...", "status": "confirmed" }
{ "child": "ANjK1x...ktZ", "skipped": true, "reason": "below_min_sweep", "balance": "40000" }
```

Handle three shapes:

| Shape | Meaning |
|---|---|
| `status: "confirmed"` + `tx_hash` | Done. Solana, TON, Base and Arbitrum answer this way |
| `status: "accepted"` + `trace_id` | TRON. Accepted, will confirm in seconds. See below |
| `skipped: true` + `reason` | Nothing moved for this child (`below_min_sweep`, `sweep_pending`, an error) |

A skipped child is not a failure of the batch. Report it and move on.

## TRON is asynchronous

On TRON each child returns `accepted` with a `trace_id` in one or two seconds, and the transfer confirms a little later. The outcome arrives as:

- webhook `sweep.confirmed` or `sweep.failed`, and
- `GET /operations?batch_id=<batch_id>` going from `pending` to `confirmed`.

Do not treat `accepted` as final in your books. A child with a pending sweep is skipped (`sweep_pending`) if swept again, so do not loop on it.

```ts
for (const r of body.results) {
  if (r.status === "confirmed") await markSwept(r.child, r.tx_hash);
  else if (r.status === "accepted") await markSweepPending(r.child, r.trace_id, body.batch_id);
  else if (r.skipped) log.info({ child: r.child, reason: r.reason }, "sweep skipped");
}
```

## When to sweep

- **On deposit:** call the sweep from the `deposit.confirmed` handler with that child's address. Funds reach the hotwallet quickly; one fee per deposit.
- **In batches:** sweep on a schedule. On TRON the network fee is per transfer, so letting small deposits accumulate in a child before sweeping costs less.

## Common mistakes

- Sweeping balances smaller than the fee. Check `min_sweep_usdt` and the quote.
- Treating a TRON `accepted` as confirmed.
- Retrying a timed-out sweep without the same `idempotency_key`.
- Expecting more than 20 children per call.
- Sending the wrong `token` for the network (TRON and TON are USDT only, Base is USDC only).
