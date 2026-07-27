# Unattended: an engineering log
I run a Claude-in-the-loop trading agent 24/7 in unattended production with my
own money on the line. This is the engineering diary — what broke, what the
guardrails caught, what a fix cost, and what I got wrong.

No hype and no growth-hacking. It's a log. The interesting parts are the
failures, because the failures are where the design gets tested.

**Why a trading bot:** not because I'm selling a strategy. Because it's the
cheapest way I know to put an LLM somewhere consequences are immediate,
measurable, and unforgiving. A support-ticket agent that fails silently costs
you a customer three weeks later. This one tells me within a candle.

## Running tally

Live since **2026-07-11**. Last updated **2026-07-27**.

| | |
|---|---|
| Days unattended | 16 |
| Automated tests gating every deploy | 934 |
| Material incidents | 2 — **$0 lost to either** |
| Monitoring-feed outages | 1 (19 hours — post #1) |
| LLM spend, all-time | $8.09 across 3,202 calls (~$0.51/day), every call priced and attributed |
| Where that spend goes | 63% narrator, 17% gate-2 vetting, 20% everything else (post #2) |
| Infrastructure | ~$11.50/month, one small VPS |
| Positions opened and managed end-to-end | 46 |
| Positions adopted from outside the signal path | 9 — each auto-fitted with an exchange-side stop at adoption |
| Strategy P&L | **Negative.** Profit factor 0.44 over 55 trades, ~66% below high-water mark on a deliberately small stake |
| Losses caused by an engineering failure | **0** |

Those last two rows are the whole point, and they belong next to each other.
The strategy is losing money — that's a research problem, and research isn't
what this log is about. Every dollar that left the account left the way it was
designed to: through a stop sitting on the exchange from the moment of fill, at
the distance the risk engine computed, on a position sized to a set fraction of
equity. Nothing was lost to a crash, a hung feed, a provider error, or an LLM
doing something surprising.

A losing strategy is actually a better test of the engineering than a winning
one. It exercises the loss paths constantly.

## Posts

- [#2 — The $0.00 line item that wasn't zero](posts/002-the-zero-that-wasnt.md) · 2026-07-27
- [#1 — The reconnect watchdog that would have failed on the exact outage it was built for](posts/001-watchdog-that-would-have-failed.md) · 2026-07-25

## The rest of it

- **[The checklist](https://github.com/amitsmarty1123-dotcom/agent-production-hardening-checklist)**
  — everything learned here, distilled into a production-hardening checklist for
  LLM agents. MIT. Section 4 names my own open gap.
- **[The case study](https://github.com/amitsmarty1123-dotcom/claude-agent-production-case-study)** — the
  full architecture write-up: asymmetric authority, fail-closed gates, metered
  spend, out-of-process protection, and the incident log.

If you have a failure mode this log hasn't hit yet, open an issue on the
checklist. I'd rather learn it from you than from production.
