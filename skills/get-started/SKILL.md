---
name: get-started
description: "Start an integration with the dolaramp API: authentication, creating a hotwallet, handling the one-time wallet_secret, API key scopes, amounts in micro-units and the overall receive, sweep and pay out flow. Use when: the user wants to accept or send stablecoins (USDT, USDC, PYUSD) through dolaramp, mentions api.dolaramp.com, a dr_live_ key, a ws_ wallet secret, or asks how the dolaramp API works."
---

## Overview

dolaramp is non-custodial stablecoin wallet infrastructure for businesses. One API key covers five networks. The business never holds a gas token: dolaramp sponsors the gas and takes its fee in stablecoin inside the transaction.

| Network (`network`) | Tokens (`token`) |
|---|---|
| `solana` | `USDT`, `USDC`, `PYUSD` |
| `tron` | `USDT` |
| `ton` (branded gram) | `USDT` |
| `base` | `USDC` |
| `arbitrum` | `USDC`, `USDT` (the USDT0 deployment) |

The flow is always the same:

1. Create one **hotwallet** per network. It returns a `wallet_secret` once.
2. Generate **child wallets**: one deposit address per customer or per order.
3. Receive deposits: `deposit.confirmed` webhook, or poll `GET /deposits`.
4. **Sweep** children into the hotwallet.
5. **Withdraw** from the hotwallet to any address.

Reference: https://dolaramp.com/docs/ (OpenAPI: https://dolaramp.com/docs/openapi.yaml). Flows by business type: https://dolaramp.com/docs/fluxos/.

## Basics

- Base URL: `https://api.dolaramp.com/v1`
- Auth: header `X-Api-Key: dr_live_...` on every call.
- Amounts in request bodies are **strings in micro-units**, 6 decimals: `"10000000"` is 10.00.
- Errors are `{ "error": "<code>", "reason"?: "<detail>" }` with a matching HTTP status.
- Rate limit: 120 requests per minute per key (`429`).
- There is no sandbox. Test on mainnet with small amounts.

## Secrets

Two secrets exist, and neither belongs in source code, logs, a browser or a chat:

- **The API key** (`dr_live_...`). Issued in the panel (https://dolaramp.com/painel/) and by `POST /keys`. The plaintext is shown once.
- **The wallet secret** (`ws_...`), returned when a hotwallet is created. It encrypts the hotwallet key and every child key. dolaramp stores only ciphertext and cannot recover it. Sweep, withdraw, bridge, export and child generation all require it in the request body.

Load both from the application's secret manager or server-side configuration at startup, and pass them in as `config.apiKey` and `config.walletSecret` (the examples in these skills assume that `config` object). If the user pastes one into the conversation, tell them to rotate it.

Keys have a scope. Use the smallest one that works:

- `read`: list, quote, poll. Enough for dashboards and reconciliation. Cannot move funds even with the wallet secret.
- `operate`: everything. A write with a `read` key answers `403 insufficient_scope`.

## A minimal client

```ts
const BASE = "https://api.dolaramp.com/v1";

export async function dolaramp<T>(path: string, init: RequestInit = {}): Promise<{ status: number; body: T }> {
  const res = await fetch(BASE + path, {
    ...init,
    headers: {
      "X-Api-Key": config.apiKey,
      "Content-Type": "application/json",
      ...init.headers,
    },
  });
  const body = (await res.json()) as T;
  return { status: res.status, body };
}
```

Return the status alongside the body. Several endpoints use the status to say something the body alone does not (`202` on a withdrawal, `503` with `retry: true`).

## Create a hotwallet

```ts
const { status, body } = await dolaramp<{ address: string; deposit_address?: string; wallet_secret?: string }>(
  "/hotwallets",
  { method: "POST", body: JSON.stringify({ network: "solana", label: "main" }) },
);
// status 201. body.wallet_secret is shown ONCE: store it in your secret manager now.
```

- One hotwallet per network: a second call answers `409`.
- **Base and Arbitrum share one address and one secret.** Creating either creates both (`evm_unified: true`). If one already existed, the other is mirrored and no new secret is returned.
- **TRON** can answer `503` with `reason: gasfree_unavailable` and `retry: true`. Nothing was created; repeat the same call. See the `tron-gasless` skill.

## Check the account

`GET /me` returns the plan (`tier`), the address usage (`addresses_used`, `addresses_limit`) and, on TRON, `accrued_fees_usd`. Check `addresses_limit` before generating addresses in bulk: over the cap, child creation answers `402 address_limit_reached`.

## What to build next

- Deposit addresses and crediting users: `receive-deposits`
- Webhook endpoint: `verify-webhooks`
- Consolidation: `sweep-funds`
- Payouts: `send-payouts`
- Accounting: `reconcile`

## Common mistakes

- Sending amounts as numbers or in whole units. They are strings in micro-units.
- Losing the `wallet_secret`. It cannot be recovered. The only way out is to reset the network (`DELETE /hotwallets/{network}`, authorised by a code emailed to the account), which destroys that hotwallet and its children: any balance still on those addresses becomes unreachable. Store the secret in a secret manager on day one.
- Using an `operate` key in a service that only reads.
- Calling the API from the browser. The key and the secret must stay on the server.
