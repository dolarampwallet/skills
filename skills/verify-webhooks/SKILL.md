---
name: verify-webhooks
description: "Receive and verify dolaramp webhooks: register an endpoint, check X-DolaRamp-Signature or the timestamped X-DolaRamp-Signature-V1 over the raw body, refuse replays, de-duplicate by X-DolaRamp-Delivery-Id and handle the retry schedule. Use when: writing a webhook handler for deposit, sweep, withdraw or bridge events from dolaramp, debugging a signature mismatch, or reviewing webhook security."
---

## Overview

dolaramp sends a `POST` with `Content-Type: application/json` and the body `{ "event": "<name>", "data": { ... } }`. Answer any `2xx` within 8 seconds. Redirects are not followed: a `3xx` counts as a failure.

## Register an endpoint

`POST /webhooks` with `{ "url": "https://...", "events": [...] }`. HTTPS only, up to 5 endpoints per account. The response returns `id` and `secret`. **The secret is shown once**: store it in your secret manager and load it into your app config at startup.

Events:

| Event | When |
|---|---|
| `deposit.detected` | A transfer was seen. Informational |
| `deposit.confirmed` | Settled. **The one to credit on** |
| `deposit.abandoned` | Seen but never settled |
| `sweep.confirmed` / `sweep.failed` | A child to hotwallet sweep settled or failed |
| `withdraw.confirmed` / `withdraw.failed` | A payout landed or did not |
| `bridge.confirmed` / `bridge.failed` | A cross-network move settled or failed |

Manage with `GET /webhooks`, `DELETE /webhooks/{id}`. Inspect attempts with `GET /webhooks/deliveries`.

## Headers on every delivery

| Header | Value |
|---|---|
| `X-DolaRamp-Signature` | hex HMAC-SHA256 of the raw body, keyed with the secret |
| `X-DolaRamp-Signature-V1` | `t=<unix seconds>,v1=<hex HMAC-SHA256 of "<t>.<raw body>">` |
| `X-DolaRamp-Event` | the event name |
| `X-DolaRamp-Delivery-Id` | stable across retries: use it to de-duplicate |
| `X-DolaRamp-Attempt` | `1` on the first try, up to `8` |

## Verify

Compute the HMAC over the **bytes received**, before parsing JSON, and compare in constant time. Prefer the V1 header: it also binds a timestamp, so an old capture cannot be replayed.

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

const TOLERANCE_S = 300;

export function verifyDolaramp(rawBody: string, headers: Record<string, string | undefined>, secret: string): boolean {
  const header = headers["x-dolaramp-signature-v1"];
  if (!header) return false;
  const parts = new Map(header.split(",").map((p) => {
    const i = p.indexOf("=");
    return [p.slice(0, i).trim(), p.slice(i + 1).trim()] as const;
  }));
  const t = Number(parts.get("t"));
  const v1 = parts.get("v1");
  if (!Number.isInteger(t) || !v1 || !/^[0-9a-f]{64}$/.test(v1)) return false;
  if (Math.abs(Date.now() / 1000 - t) > TOLERANCE_S) return false;
  const expected = createHmac("sha256", secret).update(`${t}.${rawBody}`).digest();
  const given = Buffer.from(v1, "hex");
  return expected.length === given.length && timingSafeEqual(expected, given);
}
```

## An Express handler

The raw body is the part people get wrong. A JSON body parser that runs first re-serializes the payload and the signature no longer matches.

```ts
import express from "express";

const app = express();

app.post("/webhooks/dolaramp", express.raw({ type: "application/json" }), async (req, res) => {
  const raw = req.body.toString("utf8");
  if (!verifyDolaramp(raw, req.headers as Record<string, string>, config.webhookSecret)) {
    return res.status(400).json({ error: "invalid_signature" });
  }

  const deliveryId = req.header("x-dolaramp-delivery-id")!;
  if (await alreadyProcessed(deliveryId)) return res.json({ received: true });

  const { event, data } = JSON.parse(raw);
  await enqueue(deliveryId, event, data); // persist first, work later
  res.json({ received: true });
});
```

In other frameworks, get the raw body the same way: Next.js route handlers use `await request.text()`; Fastify needs a raw-body content parser; in Python read `request.get_data()` (Flask) or `await request.body()` (FastAPI) before any JSON parsing.

## Retries and duplicates

A delivery that fails (timeout, connection error, non-2xx) is retried up to 8 attempts in total: 30 s after the first failure, then doubling (1, 2, 4, 8, 16, 32 minutes). After the last attempt it is marked failed.

So the handler must be:

- **Fast.** Verify, store, answer `2xx`. Do the work in a queue or a background job.
- **Idempotent.** The same `X-DolaRamp-Delivery-Id` can arrive more than once. For deposits, also key the credit on `tx_hash` + `address` + `token`.

A missed webhook is recoverable: `GET /deposits` and `GET /operations` hold the same facts. Poll them on a schedule as a safety net.

## Common mistakes

- Verifying against a re-serialized body instead of the raw bytes.
- Comparing signatures with `===` instead of a constant-time compare.
- Doing slow work before answering, which causes a timeout and a retry.
- Treating `deposit.detected` as a payment.
- Returning `200` on a bad signature. Return `400` so a misconfiguration is visible in the delivery log.
- Logging the webhook secret or the full headers.
