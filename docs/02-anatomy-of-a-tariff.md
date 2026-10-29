# 2. Anatomy of a tariff

[← Back: The big idea](01-the-big-idea.md) | [Next: Recipes →](03-recipes.md)

This doc walks through the actual JSON payload you push to the Powerwall,
field by field, based on the two working automations this repo is drawn
from (one forces a charge, one forces an export — see
[examples/tariffs/](../examples/tariffs/) for complete, valid examples you
can read without any Home Assistant knowledge at all).

## The shape of the payload

The service call is `teslemetry.time_of_use` (the exact service name depends
on your integration). It takes a device id and a `tou_settings` field, which
is a JSON string shaped like this:

```json
{
  "break_even_cost_to_charge": 0,
  "mode": "time_of_use",
  "tariff_content_v2": {
    "code": "MY-TARIFF-CODE",
    "currency": "GBP",
    "daily_charges": [{"amount": 0, "name": "Charge"}],
    "demand_charges": {"ALL": {"rates": {"ALL": 0}}},
    "energy_charges": {
      "ALL": {"rates": {"ALL": 0}},
      "AllYear": {"rates": {"OFF_PEAK": 0.00, "ON_PEAK": 1.50}}
    },
    "name": "My Tariff",
    "seasons": {
      "AllYear": {
        "fromDay": 1, "fromMonth": 1, "toDay": 31, "toMonth": 12,
        "tou_periods": {
          "OFF_PEAK": {"periods": [ /* ... */ ]},
          "ON_PEAK": {"periods": [ /* ... */ ]}
        }
      }
    },
    "sell_tariff": { /* same shape again, for export prices */ },
    "utility": "My Supplier",
    "version": 1
  }
}
```

It looks intimidating, but almost all of it is boilerplate you copy
unchanged. Only a handful of fields actually matter for forcing a charge or
export. Here's what each one does.

## Field by field

- **`mode`** — always `"time_of_use"` for this trick. This tells the gateway
  to plan around the tariff rather than some other built-in mode.
- **`break_even_cost_to_charge`** — a cost figure Tesla's planner uses
  internally for some of its own optimisation decisions. The source system
  sets this to `0` and hasn't needed to touch it further.
- **`tariff_content_v2`** — the actual tariff. Everything interesting is in
  here.
  - **`code`** and **`name`** — free-text labels. Give your fake tariffs
    obvious names (the source system uses things like
    `MANUAL-IMPORT-ISOLATED-SPIKE`) so you can recognise them later if you
    check what's currently pushed.
  - **`currency`** — your currency code, e.g. `"GBP"`.
  - **`utility`** — free text naming your electricity supplier. Doesn't
    need to be accurate for the trick to work, but keeping it truthful
    makes the tariff calendar entities (see [doc 4](04-home-assistant-setup.md))
    easier to read later.
  - **`version`** — a version number for the tariff schema; `1` is what the
    source system uses throughout.
  - **`daily_charges`** / **`demand_charges`** — standing charges and
    demand charges. The source system always zeroes these out (`0`), since
    they're irrelevant to a short forced window and you don't want them
    distorting the Powerwall's planning.
  - **`energy_charges`** — the actual buy prices. This is a nested
    structure: `AllYear` (see `seasons` below) maps rate *names*
    (`OFF_PEAK`, `ON_PEAK`) to prices. This is where you set your fake
    cheap or expensive price.
  - **`seasons`** — when each set of rates applies across the year. The
    source system always uses a single season called `AllYear` running
    1 January to 31 December, because the trick only needs to hold for a
    short window, not actually model seasons.
    - **`tou_periods`** — within that season, this is where you say *which
      hours of the week* count as `OFF_PEAK` versus `ON_PEAK`. This is the
      part that actually creates your "spike".
  - **`sell_tariff`** — a second, complete tariff structure (same fields:
    `energy_charges`, `seasons`, `utility`, etc.) for what you're paid to
    *export*. It mirrors the buy side but with its own rates.

## The `tou_periods` structure — how a "period" is defined

Each entry in a `periods` list looks like this:

```json
{"fromDayOfWeek": 0, "fromHour": 2, "fromMinute": 0, "toDayOfWeek": 6, "toHour": 4, "toMinute": 0}
```

- `fromDayOfWeek` / `toDayOfWeek` — 0 to 6, covering Monday through Sunday
  (in the source system's convention; check your own integration's
  numbering if in doubt, but the key point is the whole week needs covering,
  which the examples below do by spanning day 0 to day 6).
- `fromHour` / `fromMinute` and `toHour` / `toMinute` — the start and end
  time of the period, in 24-hour clock.

**The periods for `OFF_PEAK` and `ON_PEAK` together must cover the entire
week, with no gaps and no overlaps.** The Powerwall is matching "now" against
these periods to decide which rate applies; if there's a gap, behaviour for
that gap is undefined, and if there's an overlap, you don't know which rate
wins. Every period list in this repo's examples is built so that `OFF_PEAK`
and `ON_PEAK` periods between them exactly tile the week.

### Midnight wrap

A period that crosses midnight (for example, a charge window from 23:00 one
day to 01:00 the next) can't be written as a single period, because
`toHour`/`toMinute` on their own don't say "the next day". Instead, you
split it into two periods: one running from the start time to midnight, and
one running from midnight to the end time.

`toHour: 0, toMinute: 0` always means "midnight" (the very end of the day).
So a window from 23:00 to 01:00 becomes:

```json
[
  {"fromDayOfWeek": 0, "fromHour": 23, "fromMinute": 0, "toDayOfWeek": 0, "toHour": 0, "toMinute": 0},
  {"fromDayOfWeek": 1, "fromHour": 0, "fromMinute": 0, "toDayOfWeek": 1, "toHour": 1, "toMinute": 0}
]
```

The working automations in the source system handle this with a template
that checks whether the forced window's end time falls on the same calendar
day as the start time, and builds either one period (same day) or two
periods (crossing midnight) accordingly. The reusable script in
[examples/home-assistant/script_powerwall_push_window.yaml](../examples/home-assistant/script_powerwall_push_window.yaml)
does the same thing.

## The isolated spike trick

The whole point of this tariff is to make one window — your forced
charge or export period — stand out so strongly against everything else
that the Powerwall's planner has no sensible reason to do anything except
exactly what you want, exactly when you want it. In practice that means
three rates: one for your window, and a very different one for the rest of
the week. The labels `OFF_PEAK` and `ON_PEAK` are just names for "whichever
set of periods" — which one you attach to your forced window (and which
rate you give it) is entirely up to you, as the two recipes below show.

**Force charge** (from the working import controller): here the window
itself is labelled `OFF_PEAK` and everything else is `ON_PEAK`.

- Buy, inside the window (`OFF_PEAK`): **0.00** — free to buy, so the
  Powerwall has every reason to fill up.
- Buy, outside the window (`ON_PEAK`): **1.50** — absurdly expensive, so it
  won't import a single extra watt outside your window.
- Sell, everywhere: **0** — there's no reason to export at all while you're
  trying to charge, so selling is worthless in this tariff.

**Force export** (from the working export controller): here it's the other
way round — the window itself is labelled `ON_PEAK` and everything else is
`OFF_PEAK`.

- Buy, inside the window (`ON_PEAK`): **3.00** — absurdly expensive, so the
  Powerwall has no reason to import while it's meant to be exporting.
- Buy, outside the window (`OFF_PEAK`): **0.07** — a normal-ish price. The
  Powerwall doesn't need an incentive to import here; it's not the point of
  this tariff.
- Sell, inside the window (`ON_PEAK` in the `sell_tariff`): **2.90** — a
  spectacular sell price, so the Powerwall wants to export as hard as it's
  allowed to.
- Sell, outside the window (`OFF_PEAK`): **0** — no incentive to export
  outside the window.

Notice the slightly confusing detail: which label (`OFF_PEAK` or
`ON_PEAK`) you attach to your forced window is arbitrary — what matters is
which numeric rate ends up attached to which actual time period, not what
the label is called. The working export controller reuses the same pair of
period-list variables for both the buy and sell side of the tariff, so the
window and the "everything else" periods only need to be worked out once;
only the rates attached to each label differ between buy and sell. As long
as the periods themselves are correct and cover the whole week, the trick
works regardless of which label you've used for which.

The general principle, for your own variations: **make the window you want
unmistakably the best option, and make the rest of the time unmistakably a
bad idea.** Don't be subtle about it — a £0.01 price difference is not
enough of a signal; the source system uses differences of more than ten
times between the forced window and everything else.

---

Next: [Recipes →](03-recipes.md) puts this into four complete,
step-by-step procedures.
