---
name: usdt-tron-gasless
description: "Work with USDT on TRON through dolaramp without holding TRX: the gasless deposit_address, the network fee charged in USDT, one-time account activation, fees that accrue and are collected in a batch, the retryable 503 gasfree_unavailable on address creation, and asynchronous sweeps. Use when: integrating USDT TRC-20 with dolaramp, explaining TRON fees, sizing a TRON payout, or handling TRON-specific errors. Also for requests such as \"USDT TRC20 sem TRX\", \"USDT TRC20 sin TRX\"."
---

## Overview

Most USDT moves on TRON, and TRON normally needs TRX for energy. Through dolaramp, a TRON address is always a **gasless account**: transfers out of it are paid in USDT inside the transfer itself. Neither the business nor its customers ever hold TRX.

This changes four things compared with the other networks: the address to show, the fee model, the timing of sweeps, and one retryable error.

Base URL `https://api.dolaramp.com/v1`, header `X-Api-Key`. The examples assume a `config` object with `apiKey` and `walletSecret`, loaded from your secret store on the server.

## Addresses

Every TRON hotwallet and child has two fields:

- `deposit_address`: the gasless account. **This is the address to show, store and hand to customers.**
- `address`: the signing key behind it. Do not show it.

Any wallet or exchange can send USDT to a `deposit_address`; the sender pays their own network fee as usual. TRON accepts USDT only here.

## Creating addresses can answer 503

`POST /hotwallets` with `network: "tron"` and `POST /hotwallets/tron/children` can answer:

```json
{ "error": "service_unavailable", "reason": "gasfree_unavailable", "retry": true }
```

with HTTP `503`. The gasless address could not be resolved at that moment. **Nothing was created**, and a batch of children is all or nothing. Repeat the same call:

```ts
async function createTronChildren(count: number, refs: string[]) {
  for (let attempt = 0; attempt < 5; attempt++) {
    const { status, body } = await dolaramp<any>("/hotwallets/tron/children", {
      method: "POST",
      body: JSON.stringify({ wallet_secret: config.walletSecret, count, external_refs: refs }),
    });
    if (status === 201) return body.children;
    if (status === 503 && body.retry) { await sleep(2000 * (attempt + 1)); continue; }
    throw new Error(`children failed: ${status} ${body.error} ${body.reason ?? ""}`);
  }
  throw new Error("gasless address still unavailable, try again later");
}
```

Create addresses ahead of need (at signup, or keep a small pool) rather than at the moment a customer is waiting on a checkout page.

## Fees

Two separate amounts apply to every transfer out of a TRON address (a sweep or a withdrawal):

1. **Network fee**, charged by the gasless rail in USDT inside the transfer. It never passes through dolaramp. A first transfer from an address also pays a one-time **activation**.
2. **dolaramp fee**, the margin of your plan.

Read both from `GET /quote?network=tron`: `fee_usdt` is the dolaramp fee, `network_fee_usdt` is the network's, each given for the typical case and for `worst_case_new_accounts` (first transfer, with activation). Sweep and withdraw responses carry `network_fee_usdt` too, and `GET /ledger` itemizes `fee_usd`, `network_fee_usd` and `activation_usd`.

Do not hardcode these values. Quote them.

Because the network fee is per transfer and activation is per address:

- **Reuse addresses.** One address per customer, reused, pays activation once.
- **Batch small deposits.** Sweeping a child after several deposits costs one network fee instead of several.

## The dolaramp fee accrues

On TRON the dolaramp fee is not taken on each operation. It **accrues**, stays reserved in the hotwallet, and is collected in one batch transfer once it reaches a threshold. The network fee of that collecting transfer is on dolaramp.

Consequences for code:

- `GET /me` exposes `accrued_fees_usd`.
- What a TRON withdrawal can send is `balance − accrued fees − network fee of that transfer`.
- A payout above that answers `400 insufficient_balance` with `balance`, `available` and `reserved_for_fees`. Size payouts from `available`.
- `402 reserved_for_fees` means the balance covers only the reserve.
- `allow_partial: true` sends what fits (`capped_for_fees: true` in the answer). Off by default.

## Timing

- **Deposits** are picked up within seconds of the block and reported as detected; `deposit.confirmed` follows when the network confirms, about a minute later. Credit on `deposit.confirmed`.
- **Sweeps** are asynchronous: each child answers `accepted` with a `trace_id`, then `sweep.confirmed` or `sweep.failed` arrives by webhook and `GET /operations` flips from `pending` to `confirmed`. See `sweep-stablecoin-deposits`.
- **Withdrawals** allow one pending transfer per hotwallet. Run the payout queue serially. See `send-stablecoin-payouts`.

## Checking a TRON address before paying out

`POST /address/validate` with `{ "address": "T...", "network": "tron" }`. TRON addresses start with `T` and carry a checksum, so a typo is caught. An `0x...` address is not a TRON address: refuse it rather than guess.

## Common mistakes

- Showing `address` instead of `deposit_address`.
- Treating `503 gasfree_unavailable` as a permanent failure instead of retrying.
- Hardcoding the network fee.
- Offering USDC on TRON. It is USDT only.
- Sweeping every small deposit individually.
- Assuming the whole hotwallet balance can be paid out (part is reserved for accrued fees).
- Running TRON payouts in parallel.
