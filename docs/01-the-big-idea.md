# 1. The big idea

[← Back to README](../README.md) | [Next: Anatomy of a tariff →](02-anatomy-of-a-tariff.md)

## There is no "charge now" button

If you look at the Tesla API (directly, or through a Home Assistant
integration like Teslemetry), you won't find a service called
"charge the Powerwall now" or "export now". It doesn't exist.

What you do get is a way to tell the Powerwall what tariff — what
time-of-use price schedule — you are on. The Powerwall's own software then
plans its behaviour to get you the best deal on that tariff: it charges up
when electricity is cheap, and if you've told it exporting is allowed, it
sells electricity back to the grid when the price is good.

The trick this whole repository is built on is simple: **you control the
Powerwall's behaviour by controlling what it believes the tariff is**, not
by telling it directly what to do. Push it a fake tariff where right now is
"off-peak" (very cheap to buy) and it will charge. Push it a fake tariff
where right now is a brilliant sell price, and it will export. This is
sometimes called a "fake tariff" or "isolated spike" trick — you'll see both
terms used in this repo.

This is exactly what the real controllers in the source system for this repo
do, and it works. But it comes with real gotchas, which is why most of this
repo is about getting the details right (see
[Lessons learned](05-lessons-learned.md)) rather than just the trick itself.

## The four levers

Once the Powerwall has a tariff, four more settings decide exactly what it
does with that belief:

| Lever | What it controls | Typical entity (Teslemetry naming) |
|---|---|---|
| **Tariff** (time of use) | What the Powerwall *believes* electricity costs, hour by hour | Pushed via the `time_of_use` service call — see [doc 2](02-anatomy-of-a-tariff.md) |
| **Operation mode** | The overall strategy the gateway follows | `select.my_home_operation_mode` |
| **Backup reserve** | The minimum state of charge (SoC — how full the battery is, as a percentage) the Powerwall won't go below | `number.my_home_backup_reserve` |
| **Grid charging allowed** | Whether the Powerwall is allowed to charge *from the grid* at all (as opposed to only from solar) | `switch.my_home_allow_charging_from_grid` |
| **Export rule** | Whether the Powerwall may only export surplus solar, or may also export from the battery | `select.my_home_allow_export` |

Operation mode has three options that matter here:

- **`autonomous`** — called "Time-Based Control" in the Tesla app. The
  Powerwall actively plans charging and discharging around the tariff you've
  given it. This is the mode you want switched on for forced charge/export
  to actually happen.
- **`self_consumption`** — called "Self-Powered" in the app. The Powerwall
  just tries to cover your house load from solar and battery, and ignores
  the tariff for planning purposes. You'll see this mode used briefly as a
  "nudge" (see [Lessons learned](05-lessons-learned.md)) to wake up a
  gateway that's ignoring a new tariff.
- **`backup`** — keeps the battery topped up for power cuts and otherwise
  does very little. Not used by the recipes in this repo.

Export rule has two options:

- **`pv_only`** — only solar surplus can be exported; the battery itself
  won't be drained to export.
- **`battery_ok`** — the battery itself can be discharged to export, not
  just solar surplus. You need this for a forced export.

## What combination does what

| Goal | Tariff shape | Grid charging | Reserve | Export rule | Mode |
|---|---|---|---|---|---|
| Force charge from the grid | Now = very cheap to buy | On | 100 | `pv_only` (we don't need to export while charging) | `autonomous` |
| Force export (including from battery) | Now = very lucrative to sell | On* | 0 | `battery_ok` | `autonomous` |
| Hold / don't discharge | Doesn't matter much | Off | Current SoC | either | either |
| Back to normal | Your real tariff | Your normal setting | Your normal setting | Your normal setting | `autonomous` |

\* The source system this repo is drawn from leaves grid charging on even
during a forced export. That looks odd at first, but it's deliberate: with
the tariff making "now" wildly expensive to buy, the Powerwall won't
actually import at that price, and leaving the switch in its normal "on"
position avoids an extra write (and extra risk of the write failing) on the
way back to normal afterwards. See [doc 3](03-recipes.md) for the full
recipes.

## Glossary

- **TOU (time of use)** — a tariff where the price of electricity changes
  depending on the time of day, rather than being a single flat rate.
- **Tariff** — the price schedule itself: what you pay to buy electricity,
  and what you're paid to sell it, at each time of day.
- **Peak / off-peak** — "peak" is the expensive period (usually daytime or
  early evening); "off-peak" is the cheap period (classically overnight).
- **Buy tariff / sell tariff** — what you pay to *import* electricity from
  the grid (buy), versus what you're paid to *export* electricity to the
  grid (sell). The Powerwall tracks these separately.
- **Reserve (backup reserve)** — the minimum battery charge the Powerwall
  will protect. Set it to 100 and the Powerwall will try to charge all the
  way to full; set it to 0 and it's allowed to use the whole battery.
- **Gateway** — the physical controller unit that comes with the Powerwall
  and talks to Tesla's cloud (and, optionally, to your local network). When
  this repo says "the gateway reacted" or "the gateway ignored it", this is
  what's meant.
- **SoC (state of charge)** — how full the battery is, as a percentage.
- **kW vs kWh** — kW (kilowatts) is a rate — how fast power is flowing right
  now. kWh (kilowatt-hours) is an amount — how much energy has flowed over
  time. A battery discharging at 5 kW for one hour delivers 5 kWh.

---

Next: [Anatomy of a tariff →](02-anatomy-of-a-tariff.md) walks through the
actual JSON you push, field by field.
