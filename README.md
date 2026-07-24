# 🇭🇹 haiti-payments — a Claude Code skill for MonCash & Pay'm

A free, open-source **[Claude Code](https://claude.com/claude-code) skill** that
teaches Claude how to correctly integrate Haitian mobile-money payments:

- **MonCash native** — Digicel's official API (MonCash only)
- **Pay'm** — an aggregator for **MonCash + NatCash + Kashpaw**

It covers **cash-in** (collecting payments) and **cash-out** (payouts), with the
real-world API quirks the official docs get wrong, plus the money-safety patterns
that keep a payment integration from losing real money.

> Built from a production integration and **live-tested** against both APIs.
> Made to save the next Haitian developer the hours it took to figure this out.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Overview, provider choice, the shared mental model, money-safety rules, architecture |
| `references/moncash-native.md` | MonCash OAuth, CreatePayment/redirect, RetrieveOrderPayment, Transfert payout, prefunded balance, Node.js adapter |
| `references/paym.md` | Pay'm cash-in (create/verify polling), 3-step HMAC payout, error codes, Node.js helpers |

## Install

**As a personal skill** (Claude Code):
```bash
git clone https://github.com/<your-username>/haiti-payments.git \
  ~/.claude/skills/haiti-payments
```

**As part of a project** — drop the folder into your repo under
`.claude/skills/haiti-payments/`, or reference it from your plugin.

Once installed, just ask Claude things like *"add MonCash checkout to my app"* or
*"integrate Pay'm so users can pay with NatCash"* and the skill activates
automatically.

## Highlights (things that will save you hours)

- Pay'm's auth header is **`x-access-token`**, not `Authorization: Bearer` (docs are wrong → 401).
- **NatCash rejects decimal amounts** — always send whole gourdes.
- Pay'm has **no webhook** — you poll; MonCash's **redirect-back is not confirmation** — you verify.
- Payouts draw from a **prefunded balance** that must be activated + funded.
- **Confirm server-side, deliver exactly once** with an atomic claim — the core money-safety rule.

## Contributing

PRs welcome. If a provider changes its API or you find a new quirk, update the
relevant reference file and note the date. **Tested behavior beats the official
docs** — when they disagree, trust the live API and document it here.

## License

[MIT](LICENSE) — free to use, modify, and share.

---

*Made with ❤️ for the Haitian developer community.*
