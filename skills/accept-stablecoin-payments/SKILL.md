---
name: accept-stablecoin-payments
description: "Accept stablecoin payments (USDT, USDC, PYUSD) with the dolaramp API: generate child wallets as per-customer deposit addresses, tag them with external_ref, tell detected from confirmed, and credit users from the deposit.confirmed webhook or GET /deposits. Use when: building a checkout, a top-up, a gateway or an exchange deposit flow on dolaramp, generating deposit addresses, or deciding when it is safe to release goods. Also for requests in Portuguese or Spanish such as \"receber USDT\", \"gerar endereço de depósito\", \"aceptar pagos en USDT o USDC\"."
---

## Overview

A **child wallet** is a deposit address that belongs to your hotwallet. Give one to each customer or each order. Deposits to it are detected and reported to you; later you sweep the child into the hotwallet (see `sweep-stablecoin-deposits`).

Base URL `https://api.dolaramp.com/v1`, header `X-Api-Key`. Amounts are strings in micro-units (6 decimals). The examples assume a `config` object with `apiKey` and `walletSecret`, loaded from your secret store on the server.

## Generate deposit addresses

`POST /hotwallets/{network}/children`

```ts
const res = await dolaramp<{ children: Array<{ address: string; deposit_address: string; external_ref: string | null }> }>(
  "/hotwallets/tron/children",
  {
    method: "POST",
    body: JSON.stringify({
      wallet_secret: config.walletSecret,
      count: 3,
      external_refs: ["user_100", "user_101", "user_102"],
    }),
  },
);
```

- `count` is 1 to 100 per request. Loop for more.
- `external_refs` is optional, one per child, and its length must equal `count`. Use your own customer or order id (up to 128 characters). It comes back on the deposit, so no lookup table is needed.
- **Always hand the customer `deposit_address`, not `address`.** They are the same on most networks. On TRON `deposit_address` is the gasless account and is the only one to show.
- On Base and Arbitrum a child is one address valid on both networks, counted once against the cap.
- `402 address_limit_reached` means the plan's address cap is full (`tier`, `limit`, `current` are in the body).
- TRON can answer `503` with `reason: gasfree_unavailable`, `retry: true`. The batch is all or nothing: no child was created. Repeat the same call.

List them with `GET /hotwallets/{network}/children?limit=&offset=&ref=`. `ref` filters by `external_ref` (substring). `with_balance=1` adds `balance_usd` (pages of 50 or fewer).

Addresses are permanent. Reuse a customer's address for every deposit instead of creating a new one each time: it saves the address cap and, on TRON, the one-time activation fee.

## Detected versus confirmed

A deposit is reported twice, and the difference is money:

| State | Webhook | Meaning | What to do |
|---|---|---|---|
| `arriving` | `deposit.detected` | A transaction was seen on-chain | Show "payment on its way". **Do not release goods.** |
| `confirmed` | `deposit.confirmed` | Settled by that network's own rule | Credit the customer |
| `failed` | `deposit.abandoned` | Seen but never settled within the deadline | Treat as not received |

An arriving transfer can still disappear. Build the crediting logic on `deposit.confirmed` only.

## Credit from the webhook

Register an endpoint with `POST /webhooks` (see `verify-payment-webhooks` for the signature). The body is `{ "event": "...", "data": { ... } }`. For deposits `data` carries:

`network`, `token`, `address`, `external_ref`, `amount` (micro-units, string), `amount_usd`, `tx_hash`, `from`, `detected_at`.

```ts
async function onDepositConfirmed(data: {
  network: string; token: string; address: string; external_ref?: string | null;
  amount: string; tx_hash: string;
}) {
  // One credit per (tx_hash, address, token). The same event can be delivered more than once.
  const key = `${data.tx_hash}:${data.address}:${data.token}`;
  await db.transaction(async (tx) => {
    const fresh = await tx.insertDepositIfAbsent(key, data);
    if (!fresh) return;
    await tx.creditCustomer(data.external_ref, BigInt(data.amount), data.token);
  });
}
```

Use `BigInt` (or a decimal type) for `amount`. Never floats for money.

## Poll as a safety net

Webhooks can fail on your side. Reconcile with `GET /deposits?limit=200` on a schedule and credit anything `confirmed` that you have not recorded. Each row has the same fields as the webhook (`network`, `token`, `address`, `external_ref`, `amount` in micro-units, `amount_usd`, `tx_hash`, `from`, `detected_at`) plus `confirmed_at` (null while arriving) and `state`.

Because both sources carry `tx_hash`, use the same key for both: `tx_hash` + `address` + `token`. A deposit credited from the webhook is then skipped by the poll, and the other way round.

A `read` key is enough for polling.

## After the deposit

The funds sit in the child wallet until you sweep. A common pattern is to call the sweep from the `deposit.confirmed` handler, or in batches on a schedule. See `sweep-stablecoin-deposits`.

## Common mistakes

- Releasing goods on `deposit.detected`.
- Showing `address` instead of `deposit_address` on TRON.
- Crediting twice because the handler is not idempotent.
- Creating a new address per payment when one per customer would do.
- Telling a customer to send a token the network does not support here (for example USDC on TRON). Tokens per network: Solana USDT/USDC/PYUSD, TRON USDT, TON USDT, Base USDC, Arbitrum USDC and USDT.
