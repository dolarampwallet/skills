# dolaramp skills: build a stablecoin payment gateway with Claude

Skills that teach Claude how to build stablecoin payments on the [dolaramp API](https://dolaramp.com/docs/). Accept **USDT, USDC and PYUSD**, pay out, and reconcile on **Solana, TRON, TON, Base and Arbitrum**, under one API key and without ever holding a gas token.

Made for teams building a payment gateway, a checkout, an exchange deposit flow, mass payouts or a fintech backend on stablecoins, in Latin America and anywhere else.

With the plugin installed, ask in plain words and Claude writes code that follows the API as it actually behaves:

- "add USDT deposits on TRON to my checkout"
- "accept USDC on Solana and credit the customer when it confirms"
- "verify the payment webhook in my Express app"
- "send stablecoin payouts from my queue without paying twice"
- "aceitar USDT no meu sistema e creditar o cliente"
- "agregar pagos en USDT a mi pasarela"

## Skills

### stablecoin-gateway-quickstart
Start a stablecoin gateway: authentication, the hotwallet, the one-time wallet secret, API key scopes, networks and tokens, and the receive, sweep and pay out flow.

### accept-stablecoin-payments
Accept USDT, USDC and PYUSD: one deposit address per customer or order, your own reference on each address, detected versus confirmed, and crediting the customer safely.

### verify-payment-webhooks
Receive payment webhooks: signature over the raw body, replay protection, retries and de-duplication, with an Express handler and notes for other frameworks.

### sweep-stablecoin-deposits
Consolidate deposit addresses into the main wallet in batch, with the fee quoted first and the asynchronous confirmation on TRON handled.

### send-stablecoin-payouts
Send payouts and withdrawals: idempotency keys, confirmed versus pending answers, insufficient balance, and a queue that never pays twice.

### usdt-tron-gasless
USDT TRC-20 without TRX: the gasless deposit address, the network fee paid in USDT, fees that accrue, and the retryable error on address creation.

### bridge-usdc
Move USDC between Solana, Base and Arbitrum: quote, execute, and settle on the confirmation webhook.

### reconcile-stablecoin-payments
Accounting and back office: the ledger with itemized fees, operations by batch, deposits, and the monthly statement.

## Networks and tokens

| Network | Tokens |
|---|---|
| Solana | USDT, USDC, PYUSD |
| TRON | USDT |
| TON | USDT |
| Base | USDC |
| Arbitrum | USDC, USDT |

Only the issuers' official contracts: [dolaramp.com/contratos](https://dolaramp.com/contratos/).

## What the plugin does and does not do

The plugin contains skills only. It runs no code on your machine, starts no server and sends nothing anywhere. The code Claude writes calls the API from your own application, with your own key.

## Install

In Claude Code:

```
/plugin marketplace add dolarampwallet/skills
/plugin install dolaramp@dolaramp
```

## What you need

- An API key. Sign up in the panel at [dolaramp.com/painel](https://dolaramp.com/painel/) with Google, GitHub or an email code. The free plan includes the key.
- The API reference: [dolaramp.com/docs](https://dolaramp.com/docs/) (OpenAPI at `https://dolaramp.com/docs/openapi.yaml`).
- Integration flows by business type (store, gateway, exchange, on-ramp, off-ramp, mass payouts, card): [dolaramp.com/docs/fluxos](https://dolaramp.com/docs/fluxos/).
- Pricing: [dolaramp.com/pricing](https://dolaramp.com/pricing/).

There is no sandbox. Test on mainnet with small amounts.

## Keeping secrets out of the conversation

Never paste an API key or a wallet secret into a chat. The skills tell Claude to load them from your application's own secret store or server-side configuration, and to keep them out of the browser, the logs and the source code.

## Links

- For developers: [dolaramp.com/developers](https://dolaramp.com/developers/)
- Changelog: [dolaramp.com/changelog](https://dolaramp.com/changelog/)
- Security contact: [dolaramp.com/security](https://dolaramp.com/security/)
- Support: [dolaramp.com/support](https://dolaramp.com/support/) or hello@dolaramp.com

## License

Apache-2.0. See [LICENSE](LICENSE).
