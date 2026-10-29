# 3. Recipes

[← Back: Anatomy of a tariff](02-anatomy-of-a-tariff.md) | [Next: Home Assistant setup →](04-home-assistant-setup.md)

Four recipes: force a charge, force an export, hold the battery where it
is, and get back to normal. Each one lists the settings **in order** —
the order matters a great deal; see
[Lessons learned](05-lessons-learned.md) for why.

The first two recipes are drawn directly from the working automations this
repo is based on. The third ("hold") is a simpler pattern the source system
doesn't actually use day-to-day — it's included because it's a useful and
obvious variation, but it is **not** something that's been proven in the
source system, so treat it as a reasonable starting point rather than a
battle-tested recipe.

All entity names below use the placeholder scheme from
[doc 4](04-home-assistant-setup.md) — replace `my_home` with your own
entity names.

## Recipe 1: Force charge

**Goal:** make the Powerwall charge from the grid right now, for a chosen
number of hours, regardless of what it would otherwise do.

**Tariff shape:** an isolated spike where *now* is free to buy (`0.00`) and
everything else is very expensive to buy (`1.50`); selling is worthless
everywhere (`0`) since you're not trying to export. See
[doc 2](02-anatomy-of-a-tariff.md) for the full tariff, and
[examples/tariffs/force_charge_example.json](../examples/tariffs/force_charge_example.json)
for a ready-made example.

Steps, in order:

1. **Push the tariff.** Call the `time_of_use` service with the fake
   charge tariff.
2. **Wait about 15 seconds** for the gateway to settle on the new tariff.
3. **Turn grid charging on** — `switch.my_home_allow_charging_from_grid` →
   on.
4. **Set reserve to 100** — `number.my_home_backup_reserve` → `100`. This
   tells the Powerwall it's allowed (and wants) to fill all the way up.
5. **Set export rule to `pv_only`** — `select.my_home_allow_export` →
   `pv_only`. You don't need to export anything while forcing a charge, so
   keep it conservative.
6. **Set mode to `autonomous` — last.** `select.my_home_operation_mode` →
   `autonomous`. This is what makes the gateway actually re-plan against
   the new tariff and settings.

**How to check it worked:** within a few minutes,
`sensor.powerwall_battery_power` (the local gateway sensor — see
[doc 4](04-home-assistant-setup.md)) should be clearly negative (charging).
The source system treats anything above −2 kW after 3 minutes as "not
charging" and triggers a nudge (see
[Lessons learned](05-lessons-learned.md)).

## Recipe 2: Force export

**Goal:** make the Powerwall export — including from the battery, not just
surplus solar — right now, for a chosen number of hours.

**Tariff shape:** an isolated spike where *now* is a brilliant sell price
(`2.90`) and a so-so buy price (`0.07`); everywhere else is an expensive buy
price (`3.00`) and a worthless sell price (`0`). See
[doc 2](02-anatomy-of-a-tariff.md), and
[examples/tariffs/force_export_example.json](../examples/tariffs/force_export_example.json).

Steps, in order:

1. **Push the tariff.**
2. **Wait about 15 seconds.**
3. **Turn grid charging on** — `switch.my_home_allow_charging_from_grid` →
   on. This looks odd for an export recipe, but it's what the working
   source automation does: with the tariff making "now" expensive to buy,
   the Powerwall won't actually import, and leaving the switch on avoids an
   extra write (and an extra point of failure) on the way back to normal.
4. **Set reserve to 0** — `number.my_home_backup_reserve` → `0`. This
   allows the battery to be drained, not just solar surplus.
5. **Set export rule to `battery_ok`** — `select.my_home_allow_export` →
   `battery_ok`. This is the setting that actually permits exporting from
   the battery itself, not just from excess solar.
6. **Set mode to `autonomous` — last.**

**How to check it worked:** check **grid power**
(`sensor.powerwall_site_power`), not battery power. See
[Lessons learned](05-lessons-learned.md) for why battery power is
misleading here — house load can make the battery look like it's doing
something useful when the export itself has actually stalled. The source
system treats site power above −1 kW after 3 minutes as "not exporting".

## Recipe 3: Hold / don't discharge

**Goal:** stop the battery discharging any further, without forcing a
charge or export — just freeze it roughly where it is.

*This is a simpler pattern, not taken from the source automations — treat
it as an untested suggestion and verify it behaves as expected on your own
system before relying on it.*

Steps, in order:

1. **Read the current state of charge** (SoC) from
   `sensor.my_home_percentage_charged`.
2. **Set reserve to that current SoC** — `number.my_home_backup_reserve` →
   whatever the battery is at right now. The Powerwall won't discharge
   below its reserve, so this should stop further discharge.
3. **Turn grid charging off** — `switch.my_home_allow_charging_from_grid` →
   off, so it doesn't start topping up from the grid either.

You probably don't need to touch the tariff or mode for this one — it's a
much smaller intervention than a full forced charge or export. But if your
Powerwall is already mid-way through a planned charge or export against a
tariff, consider whether you also want to go "back to normal" (Recipe 4)
rather than leaving an old forced tariff in place with a frozen reserve.

**How to check it worked:** watch `sensor.my_home_percentage_charged` for a
few minutes — it should stop falling (and not necessarily rise, since
you've only set a floor, not demanded a charge).

## Recipe 4: Back to normal

**Goal:** undo a forced charge or export and return to whatever your
Powerwall normally does.

Steps, in order:

1. **Push your real tariff** — whatever your actual electricity tariff is.
   See [examples/tariffs/normal_overnight_example.json](../examples/tariffs/normal_overnight_example.json)
   for a sample UK-style overnight-cheap tariff shape you'll need to adapt
   to your own real rates and times.
2. **Wait about 15 seconds.**
3. **Restore your normal grid charging setting.**
4. **Restore your normal reserve.**
5. **Restore your normal export rule** (most households should default to
   `pv_only` unless they have a specific reason and the right permissions
   to export from the battery routinely — see the disclaimer in the
   [README](../README.md)).
6. **Set mode to `autonomous` — last**, so the Powerwall re-plans against
   the restored tariff and settings.

**How to check it worked:** check the tariff calendar entities
(`calendar.my_home_buy_tariff` / `calendar.my_home_sell_tariff` — see
[doc 4](04-home-assistant-setup.md)) show your real tariff's current
period and price, not the fake one. Then check the power sensors are
behaving the way your normal tariff would suggest (for example, not
importing heavily if it's meant to be a cheap overnight period you're not
forcing anything during).

---

Next: [Home Assistant setup →](04-home-assistant-setup.md) covers getting
the integration installed and finding your own entity ids.
