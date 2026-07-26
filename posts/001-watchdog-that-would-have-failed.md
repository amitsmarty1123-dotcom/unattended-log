# #1 — The reconnect watchdog that would have failed on the exact outage it was built for

*2026-07-25 · two weeks into running unattended*

Three things happened in about 36 hours: my monitoring went dead and stayed
quiet about it for 19 hours, I built the fix, and the fix turned out to have the
same bug class as the thing it was fixing. The third one is the interesting one.

## The outage

A deploy restart on the 24th brought the private websocket feed back up, and
then the exchange dropped it — a transient auth error at 07:26:45Z, followed by
a "goodbye" frame. There was no auto-reconnect, so the socket just stayed dead.
(Not clock skew, which was my first guess. NTP was synced.)

It stayed dead for roughly **19 hours** before I noticed it during a routine
check the next day.

Here's the part that stung. **The dead-man alert had fired.** It worked exactly
as written: it detects a silent feed with money at risk, it pages me, and it
latches so it doesn't spam. One alert, in the middle of a lot of other
notifications, and then silence for the rest of the outage.

Alert-once is completely reasonable logic. It reads fine in review. It is also
how a sustained outage stays quiet — because the alert's job isn't to tell you
something happened, it's to make sure you *know* while it's still happening. A
latch that never re-escalates is a notification, not an alarm.

What didn't happen: the open position was never unprotected. Every entry places
its stop-loss on the exchange itself at fill time, so protection doesn't depend
on my process being alive, or connected, or correct. I confirmed the stop and
the trailing stop were both still sitting there on the exchange side. The
monitoring layer failed; the layer underneath it held; **$0 lost to the
outage.**

That's not luck, it's the one design rule I'd fight hardest for: whatever
protects you from the worst case has to live in the external system, not in your
event loop.

## The fix

A supervisor task that probes socket health every 20 seconds and, when the
socket reads dead for a sustained 90-second window, rebuilds it with bounded
exponential backoff — 6 attempts, 5 seconds ramping to 120. Feed lost, feed
recovered, and gave-up-call-a-human all page me. After it gives up it stops
trying and stays on the independent 60-second reconciler until I intervene.

Two deliberate constraints:

**It keys off socket health only, never the heartbeat.** A quiet market produces
no events, and a feed with nothing to say looks identical to a feed that's dead
if you're measuring message arrival. That's a false-positive generator.

**The sustained window matters.** The vendor SDK does its own transient
reconnects; a 90-second dead window absorbs those instead of fighting them.

The reconciler's dead-man check underneath is unchanged. I didn't want the new
clever layer replacing the old dumb layer — I wanted two independent ones. This
is also the single feature I've shipped that defaults **on** in the public repo
config, where everything else optional defaults off. "Off" reproduces the
outage, so off isn't a sane default.

## The bug in the fix

Final review before deploy caught a real one, and it's worth spelling out
because I'd have shipped it.

The reconnect logic scored success by asking the socket whether it was connected
immediately after rebuilding it. Sounds right. But the SDK reports the TCP and
websocket handshake — which goes true *before* the exchange returns its async
auth verdict.

So: a **persistent** auth failure — the exact failure class that caused the
19-hour outage — would rebuild the socket, read "connected," score that as a
recovery, reset the give-up counter, and do it again forever. Flapping
indefinitely. The bounded retry would never bound. The give-up page that's
supposed to summon a human would never fire.

The watchdog written specifically for that outage would have silently failed on
that outage.

The fix: move both the recovery check and the give-up decision to the top of the
*next* supervision tick — a full check interval after any connect attempt, which
puts it past the auth verdict. Health check runs before the give-up decision
each tick, so a slow-but-genuine reconnect still counts as recovery rather than
tripping a false give-up. Then a regression test with a socket that connects and
immediately drops: it must not score recovery, and it must still reach give-up.
That test fails against the old code, which is the only reason I trust it.

Merged at 934 tests, up 20 from baseline. Deployed the same morning, no config
change needed. The websocket came up clean and the open position rode through
it.

## What I actually took from this

**Test your recovery path against the failure that motivated it.** Not against
a generic version of it. My reconnect logic handled "socket dropped" fine. It
did not handle "socket drops, reconnects, and gets rejected again," which was
the only failure I'd actually observed. I'd written the fix against my mental
summary of the incident rather than its mechanism.

**"Is it connected?" is rarely one question.** Handshake, authenticated,
subscribed, and receiving are four different states, and libraries will happily
answer a different one than the one you meant. If a health check is load-bearing
— and a watchdog's health check is the most load-bearing line in it — find out
which state you're actually reading.

**Latching alerts need an escalation path.** Fire-once-and-latch is correct for
avoiding spam and wrong for sustained bad states. Something has to re-raise, or
the absence of alerts becomes indistinguishable from health.

Next post will probably be about the cost ledger, which found a $0.00 line item
that was quietly not $0.00.
