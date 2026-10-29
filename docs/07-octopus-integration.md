# 7. Octopus integration

[← Back: Troubleshooting](06-troubleshooting.md) | [Back to README](../README.md)

This doc covers combining the fake-tariff trick with the
[BottlecapDave Octopus Energy integration](https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy)
(available via HACS), for anyone on an Octopus Energy tariff in Great
Britain. If you're not on Octopus, skip this doc — the recipes in
[doc 3](03-recipes.md) don't need it.

Octopus runs several kinds of event that are worth reacting to with a
forced charge or export: smart car-charging dispatches, paid "Saving
Sessions", and free-electricity "Power Ups". Each maps onto a recipe from
[doc 3](03-recipes.md) slightly differently, and each has its own sharp
edges.

## Entity placeholders used in this doc

As with the rest of this repo, these are placeholders — replace the
capitalised parts with your own device and account identifiers, which
you'll find in **Settings → Devices & Services → Entities**, filtered to
the Octopus Energy integration.

| Placeholder | What it is |
|---|---|
| `binary_sensor.octopus_energy_YOUR_DEVICE_intelligent_dispatching` | On while a smart (Intelligent Octopus Go) car-charging dispatch is active |
| `event.octopus_energy_YOUR_ACCOUNT_octoplus_power_down_events` | Paid Saving Session data, via its `joined_events` attribute |
| `calendar.octopus_energy_YOUR_ACCOUNT_octoplus_power_down` | Calendar view of upcoming/current paid Saving Sessions |
| `event.octopus_energy_YOUR_ACCOUNT_octoplus_power_up_events` | Free-electricity session data, via its `events` attribute |
| `calendar.octopus_energy_YOUR_ACCOUNT_octoplus_power_up` | Calendar view of upcoming/current free sessions |
| `sensor.octopus_energy_electricity_YOUR_METER_current_rate` | The current unit rate, useful for Agile-style tariffs |

## 1. Intelligent Octopus Go dispatches

Intelligent Octopus Go gives you a cheap rate for your whole house whenever
your car's smart charging schedule triggers a dispatch — not just for the
car's own charging. `binary_sensor.octopus_energy_YOUR_DEVICE_intelligent_dispatching`
turns on for the duration.

That makes a dispatch a good moment to run **Recipe 1: Force charge**
(see [doc 3](03-recipes.md)) for the Powerwall too — the rate is cheap for
everything right now, not just the car, so you may as well fill the battery
at the same time.

Two things to watch:

- **Never export during a dispatch.** The electricity meter nets import and
  export instantaneously, so exporting while the car (or anything else) is
  drawing a dispatch-rate charge just moves battery energy into the car
  instead of onto the grid. You lose the stored energy and earn nothing. See [Exporting while an EV charges](05-lessons-learned.md#exporting-while-an-ev-charges).
- **Check your account's smart-charge cap, if you have one.** Some Octopus
  accounts have a daily cap on how many hours of smart charging count as
  cheap. If you're near or over that cap, a dispatch-triggered slot may not
  actually be at the cheap rate any more — check your own account settings
  rather than assuming every dispatch is always cheap.

[examples/home-assistant/octopus/automation_charge_during_dispatch.yaml](../examples/home-assistant/octopus/automation_charge_during_dispatch.yaml)
shows the pattern: dispatch turns on → force-charge for the rest of
the slot (or until the dispatch ends); dispatch turns off → back to normal.
It's guarded so it won't fight a Saving Session that happens to be running
at the same time.

## 2. Saving Sessions (paid Power Downs) — force export

A paid Saving Session is Octopus paying you to use *less* electricity
during a specific window — which, from a Powerwall's point of view, is a
great time to run **Recipe 2: Force export** (see [doc 3](03-recipes.md)):
you get paid both by Octopus for the session and, depending on your export
tariff, for the electricity itself.

Saving Session data lives in the `joined_events` attribute of
`event.octopus_energy_YOUR_ACCOUNT_octoplus_power_down_events`, and is also
shown as a calendar, `calendar.octopus_energy_YOUR_ACCOUNT_octoplus_power_down`.

A reliable pattern, matching the working source automation this repo is
drawn from:

1. **Start 5 minutes early.** Trigger on the calendar's `start` event with
   a `-00:05:00` offset, so the forced export tariff is already pushed and
   settling before the session itself begins — the gateway's own reaction
   time (see [doc 5](05-lessons-learned.md#the-gateway-is-slow--start-time-critical-windows-early))
   means a push at the exact session start is already too late.
2. **Re-check every 3 minutes while the session runs.** Don't just push
   once and hope — keep checking **grid power**, not battery power (see
   [Test export by grid power](05-lessons-learned.md#test-export-by-grid-power-not-battery-power)),
   and nudge (`self_consumption` → 45 s → `autonomous`) if export looks
   stalled. A session that silently stalls partway through and is never
   rechecked can end up exporting nothing at all.
3. **Restore your normal settings when the session ends**, using
   **Recipe 4: Back to normal**.

[examples/home-assistant/octopus/automation_saving_session_export.yaml](../examples/home-assistant/octopus/automation_saving_session_export.yaml)
implements this pattern. Its 3-minute re-check loop is a reasonable
simplification of the source system's more elaborate version, but hasn't
been proven over as many real sessions — treat it as a solid starting
point rather than a fully battle-tested recipe, and watch your first few
sessions closely.

## 3. Power Ups (free electricity) — optionally dump, then force charge

A free-electricity "Power Up" session means the whole house (not just the
car) gets free electricity for the window. This is the one case in this
doc where it's worth checking **two** things, not one: which event entity
you're actually reading, and how much the battery can realistically refill.

**Read the right entity.** Free sessions live in the `events` attribute of
`event.octopus_energy_YOUR_ACCOUNT_octoplus_power_up_events`, and as the
calendar `calendar.octopus_energy_YOUR_ACCOUNT_octoplus_power_up`. They do
**not** appear anywhere in the Power Down `joined_events` used for paid
Saving Sessions. If your automation is filtering the Power Down data for
"zero points" as a way of spotting free sessions, that filter will **never
match** — free sessions simply aren't in that data at all, and you'll miss
every single one.

**What to do with a Power Up:**

- **Optionally, dump surplus beforehand.** If the battery is already quite
  full, consider exporting some of it just before the free session starts,
  to make room — but only dump what the session can actually refill. The
  battery can only recharge so fast (bounded by your grid charging limit),
  so a session only has so many kWh of genuine "refill capacity" in it. If
  you dump more than the session can put back, you'll end the session with
  less energy stored than you started with, for no benefit — this is an
  easy way to lose value rather than gain it. Work out your refill capacity
  from your own measured charge rate (see
  [Measure your real numbers](05-lessons-learned.md#measure-your-real-numbers-dont-trust-the-spec-sheet))
  multiplied by the session length, and only dump down to that floor.
- **Force charge during the session** using Recipe 1, since electricity is
  free for the duration.

[examples/home-assistant/octopus/automation_charge_during_dispatch.yaml](../examples/home-assistant/octopus/automation_charge_during_dispatch.yaml)'s
pattern (force-charge start/stop around a window) also applies here; the
dump-before step is left out of the examples in this repo since it depends
heavily on your own measured numbers, but the principle is described above.

## 4. Agile-style tariffs — charge when the rate is low

If you're on a half-hourly variable tariff (such as Octopus Agile), you
already have a sensor with the current unit rate:
`sensor.octopus_energy_electricity_YOUR_METER_current_rate`. A simple
variation on Recipe 1 is to force-charge whenever that rate drops below a
threshold you choose, and go back to normal when it rises above it again.

**This is an untested suggestion** — it isn't something the source system
actually runs, so treat [examples/home-assistant/octopus/automation_agile_cheap_charge.yaml](../examples/home-assistant/octopus/automation_agile_cheap_charge.yaml)
as a starting point, not a proven recipe. Two things worth building in from
the start:

- **A minimum "on" time**, so a rate that hovers right around your
  threshold doesn't flap the Powerwall in and out of a forced charge every
  half hour, which would also eat into your Tesla command rate-limit budget
  (see [doc 5](05-lessons-learned.md#tesla-rate-limits-commands--budget-your-pushes)).
- **An awareness of how many pushes a day this adds up to.** A half-hourly
  rate sensor changing a lot, combined with a naive "push whenever the
  condition is true" automation, is exactly the kind of thing that trips
  the rate limit — dedupe so you only push when you're actually changing
  state (cheap → not-cheap or back), not on every rate update.

## Octopus-specific lessons

These are additional lessons, specific to combining this project with the
Octopus integration, on top of everything in [doc 5](05-lessons-learned.md).

**Octopus's own API fails too, especially at busy moments.** Right at
session boundaries (session starting or ending), Octopus's backend can be
under heavy load, and writes to it sometimes fail. Critically, the Octopus
integration raises its own error type for these failures, which is **not**
a standard Home Assistant error — `continue_on_error: true` on the call
itself does **not** catch it, and the automation calling it aborts anyway,
exactly like the Tesla rate-limit case in [doc 5](05-lessons-learned.md).
**The fix:** route every Octopus write through a separate script, started
with a fire-and-forget call (so the caller doesn't wait for it and can't be
aborted by its failure), and verify the result afterwards by reading the
entity back. See
[examples/home-assistant/octopus/script_octopus_safe_write.yaml](../examples/home-assistant/octopus/script_octopus_safe_write.yaml)
for a generalised version of this pattern.

**Put Octopus writes last.** Since they're more failure-prone than the
Powerwall commands, and less important to the actual energy outcome, order
your sequence so the Powerwall settings (the part that actually matters)
are written first, and any Octopus writes (like adjusting a smart-charging
target) come after.

**Target and ready-by writes are read-modify-write — queue them, don't
parallelise them.** Setting things like a smart-charging target percentage
or a "ready by" time works by reading the current schedule, changing one
field, and sending the whole thing back. If two such writes run at the same
time, both read the same starting point, and whichever one's write lands
last silently overwrites the other's change — the value that "loses" can
appear to revert on its own a minute or two later, which is confusing to
debug if you don't know this is happening. **The fix:** run this class of
Octopus write through a script using Home Assistant's `queued` mode (not
`parallel`), so writes are serialised rather than racing each other.

**Rate and dispatch sensors update a few seconds apart.** At a slot
boundary, the current-rate sensor and a related dispatch/status sensor
don't always update in the same instant — a gap of several seconds between
them is normal. With a plain `single`-mode automation, the second trigger
can be dropped while the first run is still inside its tariff-settle delay,
leaving the wrong posture in place for longer than intended. **The fix:**
the same one described in [doc 5](05-lessons-learned.md#near-simultaneous-triggers--use-restart-mode-and-dedupe)
— use `mode: restart` with a dedupe check, so the newest trigger always
wins and you still only push once.

---

Back to [README](../README.md), or [Troubleshooting](06-troubleshooting.md)
if something here isn't working as expected.
