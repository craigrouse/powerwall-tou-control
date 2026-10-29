# 5. Lessons learned

[← Back: Home Assistant setup](04-home-assistant-setup.md) | [Next: Troubleshooting →](06-troubleshooting.md)

This is the most important doc in this repo. Everything below was learned
the hard way, running this for real over several months. The details have
been generalised and all personal identifiers removed, but the underlying
causes and fixes are accurate. If you only read one doc before leaving any
of this running unattended, make it this one.

## Order of operations: tariff first, mode last

**What happened:** for a while, the working automations pushed settings in
whatever order seemed natural — often mode first, tariff last. Since a
gateway firmware update, the Powerwall started checking the tariff *before*
honouring reserve, export, or mode changes. With the old ordering, reserve
and mode changes were landing against the *old* tariff and being silently
ignored — a forced fill one evening simply never started, until someone
noticed and manually toggled the mode.

**The rule:** always push the tariff first, wait about 15 seconds for it to
settle, and only then set grid charging → reserve → export rule → operation
mode, **with mode last**. The mode write is what actually makes the gateway
re-plan against the new tariff. This pattern has held up ever since, and
community reports (see GitHub issue `alandtse/tesla#1171`) describe the
same underlying behaviour change.

## Never trust a pushed setting — verify, then nudge

**What happened:** more than once, every command in a sequence succeeded
(no errors at all) and the gateway still didn't do the thing. It had
silently reverted to its previous mode, or just never acted on the new
plan.

**The rule:** after pushing a forced window, wait about 3 minutes and check
the relevant power sensor (see the next two lessons for *which* sensor).
If it still isn't doing what you asked, "nudge" the gateway: switch mode to
`self_consumption`, wait **45 seconds**, then switch back to `autonomous`.
Short toggles (5–10 seconds) were tried first and made things worse — they
raced Tesla's own cloud sync, and a stale write would apply itself minutes
later, undoing the nudge. 45 seconds has worked reliably.

## Test export by grid power, not battery power

**What happened:** during a forced export, the battery power sensor showed
the battery discharging at roughly 1 kW. It looked like it was working. It
wasn't — that 1 kW was just covering the house's own electricity use. No
export was actually happening at all, but the "is it working?" check (based
on battery power) said everything was fine, so the automation didn't nudge
anything, and the export never started.

**The rule:** when testing whether a forced export is actually working,
check **grid/site power**, not battery power. House load can make the
battery look busy even when nothing at all is reaching the grid.

## Cloud sensors lag — use local sensors to test "did it act"

**What happened:** the cloud-polled (Teslemetry) power sensors can lag the
real, physical state of the gateway by about a minute. On one occasion, a
stale cloud reading made an automation believe an export hadn't started
when it actually had — and it triggered a needless mode toggle right in
the middle of a perfectly healthy export.

**The rule:** the local gateway integration's sensors update roughly every
30 seconds and reflect the gateway's real state much more promptly. Use
**those** for any "did the gateway actually do what I asked" check. Use the
cloud entities for the things you write to (mode, reserve, export rule,
grid charging) and for anything that doesn't need split-second accuracy.

## The gateway is slow — start time-critical windows early

**What happened:** the gateway typically takes somewhere around 3–4
minutes to actually react to a new tariff and settings, even when
everything is pushed correctly and promptly.

**The rule:** if you're trying to line a forced window up with something
time-critical (an external tariff event starting at an exact time, for
example), start your own window about 5 minutes early, so the gateway has
time to react before the moment that actually matters.

## Tesla rate-limits commands — budget your pushes

**What happened:** a burst of about six tariff pushes within sixteen
minutes was enough to get `"Account or server is rate limited"` back from
every subsequent command for the next hour or so. Worse, the automation
that triggered this aborted at its very first action when the rate limit
hit, and left the Powerwall grid-charging at a peak rate for sixteen
minutes before anyone noticed.

**The rule:** budget well under about five tariff pushes per fifteen
minutes. Never wire an automation's trigger to a helper that gets rewritten
periodically by something else (that's an easy way to rack up pushes
without meaning to). Only push a new tariff when the tariff actually needs
to change — compare a hash (or similarly cheap fingerprint) of the tariff
you're about to push against the last one you successfully pushed, and skip
the push if nothing's changed.

## Fail safe — every Tesla command needs `continue_on_error: true`

**What happened:** directly related to the rate-limit incident above — the
very first command in the sequence (the tariff push) hit the rate limit,
threw an error, and the whole automation stopped dead. Every command after
it — including the ones that would have turned grid charging back off and
dropped the reserve — never ran. The battery sat grid-charging at peak rate
for sixteen minutes as a result.

**The rule:** mark every single Tesla command `continue_on_error: true`.
That way, even if one command fails (rate limit or otherwise), the rest of
the sequence still runs — and a stale tariff with grid charging switched
off safely is a far better failure mode than an aborted sequence that
leaves grid charging on at a bad moment.

## The gateway can overrun a cheap window

**What happened:** rather than charging steadily throughout a cheap window,
the gateway sometimes finishes its charge plan with a full-power sprint
right at the end — and that sprint can straddle the boundary into the
*expensive* period that follows, costing real money at the worst possible
rate.

**The rule:** end your pushed cheap window about 15 minutes before the real
cheap window actually ends, so any end-of-window sprint lands safely inside
the genuinely cheap period rather than spilling over into the expensive
one.

## One writer at a time

**The rule:** only one automation should be in charge of the tariff for any
given situation at any given moment, gated by clear boolean conditions (an
`input_boolean` or similar) so it's obvious which automation is "holding
the pen". Two automations racing to push different tariffs at the same time
is a recipe for exactly the kind of silent failure described throughout
this doc.

## Near-simultaneous triggers — use `restart` mode and dedupe

**What happened:** two separate sensors that were logically related (for
example, a price sensor and a related dispatch/status sensor) updated
roughly 7 seconds apart rather than at exactly the same instant. With a
normal (`single`) automation mode, the second trigger got dropped while the
first run was still inside its 15-second tariff-settle delay — leaving the
wrong posture active for several minutes until something else rescued it.

**The rule:** use Home Assistant automation `mode: restart` for these
situations (so a newer trigger cancels and replaces an older, still-running
one) combined with a dedupe check (don't re-push an unchanged posture) so
that the newest trigger always wins and you still only push once.

## Start long scripts "fire-and-forget"

**What happened:** the push script waits about four minutes (15-second
settle, 3-minute check, 45-second nudge). When an automation called it
directly and then restarted because a new trigger arrived, the half-finished
run could be cut off, or a late "nudge" could fire after the battery had
already gone back to normal.

**The rule:** start the long scripts with `script.turn_on` (the calling
automation does not wait), and have "back to normal" stop any push that is
still running before it does anything else. All the examples in this repo
do both.

## Exporting while an EV charges

**What happened:** exporting from the battery while an electric vehicle is
simultaneously drawing a charge from the same supply doesn't actually get
you anything — the electricity meter nets the two flows together
instantaneously. You're not selling power to the grid at all; you're just
moving energy from your battery into your car, at your export tariff's
(fake) sell price, which is not what you wanted.

**The rule:** pause any forced export while a car is actively charging.

## Measure your real numbers, don't trust the spec sheet

**What happened:** the source system's battery is rated for a certain
export limit, but its actual measured net export under real conditions came
in noticeably lower than the rated figure (roughly 11.5 kW against a
14 kW rated limit). Discharge rate also slows further at low state of
charge (roughly 6.5 kW, well below the normal rate, once the battery
dropped under about 13%).

**The rule:** when you're planning the *timing* of a forced window (how
long will it take to export/import a given amount of energy?), use numbers
you've actually measured on your own system, not the numbers from a spec
sheet. They can be meaningfully different, especially near the empty end of
the battery's range.

## The Tesla app can fight you

**The rule:** check the Tesla app for anything that might conflict with an
API-pushed tariff — in particular, any linked energy-provider tariff or
"automatic rate plan" feature, and any Wall Connector rate-plan charging
schedule. Turn these off. They and your pushed fake tariff will otherwise
compete with each other, and you won't always be able to tell which one
"won" at any given moment.

## Integration updates can break this

**What happened:** an update to the Tesla integration being used broke the
`time_of_use` call outright on one occasion.

**The rule:** pin a known-good version of your integration rather than
auto-updating it. Read release notes before updating deliberately, and
test that a forced charge/export still works straight after any update,
before you trust it unattended again.

## Firmware updates change behaviour

**What happened:** the gateway firmware update that introduced the
"tariff checked before reserve/mode" behaviour (see the very first lesson
above) was itself a firmware change, not a Home Assistant or integration
change. A later major firmware version introduced a further round of
stalls that needed new workarounds.

**The rule:** keep your own notes on what firmware version you're on and
what behaviour you've seen. Watch the first few sessions closely after any
gateway firmware update — don't assume last month's working recipe still
behaves identically.

## Local control doesn't cover the tariff

**What happened:** where a local (LAN) connection to the gateway is
available, it can set mode, reserve, export rule and grid charging directly
— fast, and without touching Tesla's cloud command rate limit at all. The
tariff itself, however, could not be set this way; the underlying local API
call for it simply isn't implemented, and pushing a tariff still has to go
via the cloud regardless.

**The rule:** if you're using local control for the other three levers,
remember the tariff push is still a cloud operation, with cloud lag and the
cloud rate limit both still applying to it.

## The `input_number.set_value` trap

**What happened:** a helper value was being set with `number.set_value`
targeting an `input_number.*` entity. This call matched nothing and
silently did nothing at all — no error, just no effect — which meant a
dependent value was stuck at its default for a long time before anyone
noticed.

**The rule:** `input_number`, `input_datetime`, and `input_select` helpers
each need their *own* domain's `set_value` / `set_datetime` /
`select_option` action — never the plain `number.set_value`,
`datetime.set_value`, or similar. The plain domain action is for entities an
*integration* created (like the real Powerwall `number.*` entities), not
for helpers you created yourself.

## Confirm via the tariff calendar first, then the power sensors

**The rule:** after a push, check the tariff calendar entities
(`calendar.my_home_buy_tariff` / `calendar.my_home_sell_tariff` — see
[doc 4](04-home-assistant-setup.md)) to confirm the new tariff actually
landed on the cloud side, *before* worrying about whether the power
sensors look right. If the calendar still shows the old tariff, nothing
downstream is going to behave correctly no matter what else you check.

## Supplier APIs fail too — don't let them abort the battery automation

**What happened:** calls to an electricity supplier's own API (for example,
to adjust a smart-charging schedule) sometimes failed with errors that
Home Assistant didn't treat as a normal, catchable automation error — so
`continue_on_error: true` on that specific call did nothing, and the whole
automation aborted partway through, before it had even reached the
Powerwall commands.

**The rule:** wrap any non-essential, third-party (supplier) writes so
their errors genuinely can't abort the battery-control part of your
automation — for example, by calling them through a separate script
(fired "and forget", not waited on) rather than inline, and by putting any
non-essential actions like this **last** in your sequence, well after the
Powerwall commands that actually matter have already run.

---

Next: [Troubleshooting →](06-troubleshooting.md) is a quick-reference table
for when something above still isn't fixing it. See also
[doc 7, Octopus integration](07-octopus-integration.md) for lessons
specific to combining this with an Octopus Energy tariff.
