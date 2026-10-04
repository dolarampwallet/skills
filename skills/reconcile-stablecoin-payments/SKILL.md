---
name: reconcile-stablecoin-payments
description: "Reconcile and account for dolaramp activity: GET /ledger for every on-chain movement with itemized fees, GET /operations for sweeps, withdrawals and bridges by batch, GET /deposits, GET /statement for the monthly export, balances and the webhook delivery log. Use when: building accounting, a back-office report, a daily reconciliation job, a CSV export, or finding out what happened to a specific deposit, sweep or payout. Also for requests such as \"conciliação\", \"extrato mensal\", \"conciliación de pagos\"."
---

## Overview

Webhooks tell you things as they happen. These read endpoints are the record to check them against. All of them work with a `read` key, which cannot move funds: use one for reporting services.

Base URL `https://api.dolaramp.com/v1`, header `X-Api-Key`.

| Question | Endpoint |
|---|---|
| Everything that moved, with fees | `GET /ledger` |
| The state of a sweep, payout or bridge | `GET /operations` |
| Which deposits arrived and their state | `GET /deposits` |
| The month, for accounting | `GET /statement?month=YYYY-MM` |
| What is in an address now | `GET /balance/{network}/{address}/{token}` |
| Did my webhook endpoint receive it | `GET /webhooks/deliveries` |
| Plan, address usage, fees accrued | `GET /me` |

## The ledger

`GET /ledger` merges deposits with sweeps, withdrawals and fee settlements into one feed.

Query: `day` (`YYYY-MM-DD`, UTC), `network`, `token`, `kind` (`deposit`, `sweep`, `withdraw`, `fee_collection`), `sort` (`date_desc`, `amount_desc`, `amount_asc`), `limit` (up to 200), `offset`.

Each row has `kind`, `datetime`, `network`, `token`, `amount_usd`, `fee_usd` (dolaramp), `network_fee_usd` and `activation_usd` (charged by the network on TRON), `tx_hash`, `ref` (the child's `external_ref`, the child address or the destination), `batch_id` and `status`. The answer carries `has_more`.

```ts
async function ledgerForDay(day: string) {
  const rows = [];
  for (let offset = 0; ; offset += 200) {
    const { body } = await dolaramp<{ rows: any[]; has_more: boolean }>(
      `/ledger?day=${day}&limit=200&offset=${offset}`,
    );
    rows.push(...body.rows);
    if (!body.has_more) break;
  }
  return rows;
}
```

Deposits are free. `fee_collection` rows are the batch in which fees accrued on TRON were collected.

## Operations

`GET /operations?limit=&batch_id=` lists sweeps, withdrawals and bridges with their fee and network fee breakdown. Use it to:

- settle anything you marked in flight (`pending` becomes `confirmed` or `failed`), and
- read one sweep batch by its `batch_id`.

Withdrawal and bridge responses carry `operation_id`; match on it, or on your `idempotency_key` from the webhook.

## Deposits

`GET /deposits?limit=` returns `state` (`arriving`, `confirmed`, `failed`), `detected_at`, `confirmed_at`, `address`, `external_ref`, `amount` (micro-units), `amount_usd`, `tx_hash`, `from`. Credit only `confirmed`. See `accept-stablecoin-payments`.

## Monthly statement

`GET /statement?month=2026-09` (UTC) returns `period`, a `summary` (deposit count and total, sweeps, withdrawals, `fees_total`, `network_fees_total`), `addresses`, and the line items in `deposits` and `operations`. The summary is exact even when the line items are capped, so use the summary for totals and page the ledger for full detail.

## A daily reconciliation job

```ts
async function reconcile(day: string) {
  const rows = await ledgerForDay(day);

  for (const r of rows.filter((r) => r.kind === "deposit" && r.status === "confirmed")) {
    if (!(await db.hasDeposit(r.tx_hash, r.ref))) await alert("deposit missing in our books", r);
  }
  for (const r of rows.filter((r) => r.kind === "withdraw")) {
    const ours = await db.findPayoutByTx(r.tx_hash);
    if (!ours) await alert("payout on chain that we did not record", r);
    else if (ours.status !== r.status) await db.updatePayoutStatus(ours.id, r.status);
  }

  const stale = await db.payoutsInFlightOlderThan("15 minutes");
  if (stale.length) await alert("payouts still in flight", stale);
}
```

Compare in both directions: what the ledger has that your books lack, and what your books claim that the ledger does not show.

## Money math

- `amount` fields on webhooks and operation responses are strings in micro-units. Use `BigInt` or a decimal type.
- `*_usd` fields are for display and reports.
- Sum fees per kind: `fee_usd` is dolaramp's, `network_fee_usd` is the network's (TRON), and `activation_usd` is part of `network_fee_usd`, not an extra.

## Webhook delivery log

`GET /webhooks/deliveries?limit=` shows each delivery with `event`, `url`, `status_code`, `attempts` and `success`. When an event seems missing, look here first: a failing endpoint shows as a non-2xx status and a growing attempt count.

## Common mistakes

- Using an `operate` key in a reporting service.
- Trusting only webhooks, with no scheduled reconciliation.
- Adding `activation_usd` on top of `network_fee_usd`.
- Reading days in local time. `day` and `month` are UTC.
- Summing `*_usd` display strings with floats for accounting.
