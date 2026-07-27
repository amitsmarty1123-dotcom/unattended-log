# #2 — The $0.00 line item that wasn't zero

*2026-07-27 · sixteen days unattended*

Zero is a plausible number. That's what makes it dangerous in a cost ledger.

My agent prices every model call and writes it to a ledger row. The pricing
table is keyed by model name. Miss a key, and the lookup returns nothing — and
the obvious thing for a cost function to do with "I don't know what this costs"
is return `0.0`, because what else would it return?

So the call happens, the money is spent, the row says **$0.00**, and the total
at the bottom of the page is wrong in the one direction you'll never
investigate. Nobody audits a bill that's lower than expected.

## The shape of the bug

This isn't really a pricing bug. It's a **missing-data-rendered-as-a-value**
bug, and it shows up everywhere once you start looking:

- Unknown cost → `0`
- Missing latency sample → `0ms`
- Absent error count → `0 errors`
- Unparsed usage payload → no tokens, so no spend

Each of those is a number that means "I don't know", wearing the costume of a
number that means "there was none." The first is an alarm. The second is
reassurance. They render identically.

The fix isn't to make the cost function throw — that would take down a live
trading path over a bookkeeping concern, which is a worse trade. The fix is that
**the ignorance has to be visible somewhere the total isn't.** The cost function
still returns zero and the call still proceeds, but the sink that writes the
ledger row checks: cost came out zero, tokens were actually used, and this model
isn't in the pricing table. That combination isn't a free call. That's a gap in
my table, and it logs a warning saying exactly that, naming the model.

The row still gets written with cost zero. I'd rather have an
undercounted row than a missing one — a missing row loses the token counts too,
and those are the thing you need to reconstruct the real number later.

## The latch, again

Here's the part that connects to [post
#1](001-watchdog-that-would-have-failed.md). That warning fires **once per
model, per process lifetime.** There's a set of already-warned models, and the
second occurrence is silent.

Post #1 was me getting burned by exactly that pattern: an alert that fired once
and latched while a dead feed stayed dead for 19 hours.

So is this the same mistake twice? I don't think so, and the distinction is
worth being precise about, because "never latch alerts" is the wrong lesson to
take from post #1.

**Latch on facts. Escalate on states.**

"Model `X` is not in the pricing table" is a *fact about configuration*. It is
equally true on the first call and the ten-thousandth. Repeating it ten thousand
times adds zero information and actively harms you — it buries the next real
warning under noise. One clear statement, then silence, is correct.

"The websocket feed is dead while money is at risk" is a *state of the running
system*. It has a duration. Its meaning changes as that duration grows: dead for
30 seconds is a blip, dead for 19 hours is an emergency. An alert that can't
express duration can't express the difference, which is precisely how I ended up
with the outage in post #1.

The test I use now: **if the same alert firing again tomorrow would mean
something different than it means today, it must be able to fire again
tomorrow.** Config facts fail that test. Live states pass it.

## The other way a ledger goes quietly wrong

While I was in there: prompt caching bills at rates that aren't the input rate.
Cache reads bill at a fraction of input; cache writes bill at a premium over it.
And — the part that actually bites — those cached tokens are reported in their
*own* fields, excluded from the normal input-token count.

Price them at the input rate and you overcharge yourself on reads. Ignore the
fields entirely and you undercount writes, which cost *more* than input. Both
errors are silent, both compound per call, and they push in opposite directions
so they can partially mask each other in the total.

If you use prompt caching and your ledger only reads `input_tokens` and
`output_tokens`, your numbers are wrong right now.

## What the ledger actually told me

Sixteen days in, total model spend is **$8.09** across **3,202 calls** — about
fifty cents a day. Small enough that the absolute number is uninteresting.

The *distribution* is not:

| Subsystem | Spend | Calls |
|---|---|---|
| Narrator (writes status prose I read on a dashboard) | **$5.12** | 2,779 |
| Trade proposer | $0.63 | 324 |
| Self-improvement sessions | $0.92 | 11 |
| **Gate 2** (vets every entry, can reject or shrink it) | **$1.39** | **80** |
| Scan / auto-approver | $0.03 | 8 |

**63% of my model spend, and 87% of my calls, go to a feature that produces
commentary.** The subsystem that actually stands between a bad signal and my
money is $1.39.

I'm not arguing that's wrong — the narrator runs on a timer regardless of
whether anything happens, and Gate 2 only runs when a signal fires, so the call
counts follow directly from that design. But I didn't *know* it until the ledger
could tell me, and at ten times the scale that ratio is the difference between a
$50 month and a $500 one. If I ever need to cut spend, I now know exactly where
to cut and — more importantly — exactly which $1.39 to never touch.

That's the actual argument for per-call attribution. Not the total. **A total
tells you whether you can afford it. Attribution tells you what you're buying,
and whether the expensive thing is the important thing.**

Mine wasn't. Yours probably isn't either.

---

*Next: what happens when a fallback model provider rejects a request your
primary would have accepted — and why the entry it killed was the right
outcome.*
