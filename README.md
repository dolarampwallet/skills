# dolaramp skills

Skills that teach Claude how to build on the [dolaramp API](https://dolaramp.com/docs/): stablecoin wallet infrastructure for businesses. Receive USDT, USDC and PYUSD on Solana, TRON, gram (TON), Base and Arbitrum, consolidate with gasless sweeps, and pay out, all under one API key.

With this plugin installed, ask Claude for an integration in plain words ("add USDT deposits on TRON to my checkout", "verify the dolaramp webhook in my Express app", "send payouts from my queue") and it writes code that follows the API as it actually behaves.

## What is inside

| Skill | Use it when |
|---|---|
| `get-started` | First integration: API key, hotwallet, the wallet secret, the overall flow |
| `receive-deposits` | Deposit addresses per customer, detected versus confirmed, crediting users |
| `verify-webhooks` | Signature check, replay protection, retries, de-duplication |
| `sweep-funds` | Consolidating deposit addresses into the hotwallet |
| `send-payouts` | Withdrawals with idempotency, `202 pending`, insufficient balance |
| `tron-gasless` | USDT on TRON without TRX: addresses, fees, the retryable `503` |
| `bridge-usdc` | Moving USDC between Solana, Base and Arbitrum |
| `reconcile` | Ledger, operations and the monthly statement |

The plugin contains skills only. It runs no code on your machine, starts no server and sends nothing anywhere. The code Claude writes calls the API from your own application, with your own key.

## Install

In Claude Code:

```
/plugin marketplace add dolarampwallet/skills
/plugin install dolaramp@dolaramp
```

## What you need

- An API key (`dr_live_...`). Sign up at [dolaramp.com/painel](https://dolaramp.com/painel/) with Google, GitHub or an email code. The free plan includes the key.
- The API reference: [dolaramp.com/docs](https://dolaramp.com/docs/) (OpenAPI at `https://dolaramp.com/docs/openapi.yaml`).
- Integration flows by business type: [dolaramp.com/docs/fluxos](https://dolaramp.com/docs/fluxos/).

There is no sandbox. Test on mainnet with small amounts.

## Keeping secrets out of the conversation

Never paste an API key or a `wallet_secret` into a chat. The skills tell Claude to read them from environment variables (`DOLARAMP_API_KEY`, `DOLARAMP_WALLET_SECRET`) and to keep them on the server side of your application.

## Links

- Changelog: [dolaramp.com/changelog](https://dolaramp.com/changelog/)
- Pricing: [dolaramp.com/pricing](https://dolaramp.com/pricing/)
- Security contact: [dolaramp.com/security](https://dolaramp.com/security/)
- Support: hello@dolaramp.com

## License

Apache-2.0. See [LICENSE](LICENSE).
