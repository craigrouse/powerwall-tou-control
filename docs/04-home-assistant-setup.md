# 4. Home Assistant setup

[← Back: Recipes](03-recipes.md) | [Next: Lessons learned →](05-lessons-learned.md)

This doc covers getting the pieces in place in Home Assistant: installing
the integration that lets you push fake tariffs, finding your own entity
ids, understanding the sensor sign conventions, and the helpers the example
automations expect.

## Install Teslemetry

[Teslemetry](https://teslemetry.com/) is a paid cloud service and matching
Home Assistant integration that proxies commands to Tesla's official Fleet
API — including the `time_of_use` call this whole repo is built on. It's
available either through [HACS](https://hacs.xyz/) (Home Assistant Community
Store — install HACS first if you don't have it, then add Teslemetry as a
custom integration) or, depending on your Home Assistant version, it may be
available directly from the core integrations list. Check the Teslemetry
website for the current recommended install method, since this changes
faster than this doc will stay up to date.

Teslemetry requires an active subscription to use (it's what pays for
Tesla's own API access on your behalf). Budget for that as an ongoing cost
of this whole approach.

## Find your Powerwall's device id

The `time_of_use` service call needs a Tesla **device id** for your
Powerwall (a long hexadecimal string — this repo uses the placeholder
`YOUR_POWERWALL_DEVICE_ID` throughout). To find yours:

1. In Home Assistant, go to **Settings → Devices & Services → Devices**.
2. Find your Powerwall device (it'll be listed under the Teslemetry
   integration).
3. Open it, and look at the URL in your browser's address bar — the device
   id is the long string of letters and numbers at the end.

## Find your entity ids

Teslemetry creates entities named after your energy site (often based on
whatever name you gave the site when you set it up, or your address). The
exact names will be specific to your installation, so this repo uses
placeholder names throughout. Here's the "rename these" table:

| Placeholder used in this repo | What it is | Typical pattern on your system |
|---|---|---|
| `select.my_home_operation_mode` | Operation mode (`autonomous` / `self_consumption` / `backup`) | `select.<your_site_name>_operation_mode` |
| `number.my_home_backup_reserve` | Backup reserve, 0–100% | `number.<your_site_name>_backup_reserve` |
| `select.my_home_allow_export` | Export rule (`pv_only` / `battery_ok`) | `select.<your_site_name>_allow_export` |
| `switch.my_home_allow_charging_from_grid` | Whether grid charging is allowed | `switch.<your_site_name>_allow_charging_from_grid` |
| `sensor.my_home_grid_power` | Grid power, cloud-polled (Teslemetry) | `sensor.<your_site_name>_grid_power` |
| `sensor.my_home_battery_power` | Battery power, cloud-polled (Teslemetry) | `sensor.<your_site_name>_battery_power` |
| `sensor.my_home_percentage_charged` | State of charge (SoC) % | `sensor.<your_site_name>_percentage_charged` |
| `calendar.my_home_buy_tariff` | Current buy-tariff period/price | `calendar.<your_site_name>_buy_tariff` |
| `calendar.my_home_sell_tariff` | Current sell-tariff period/price | `calendar.<your_site_name>_sell_tariff` |
| `sensor.powerwall_site_power` | Site (grid) power, **local gateway**, ~30 s | from the core `powerwall` integration, see below |
| `sensor.powerwall_battery_power` | Battery power, **local gateway**, ~30 s | from the core `powerwall` integration, see below |
| `notify.my_phone` | Wherever you want nudge/failure alerts sent | your own notify service |
| `YOUR_POWERWALL_DEVICE_ID` | Teslemetry device id | see above |

Find the exact names on your own system via **Settings → Devices &
Services → Entities**, filtering by your Powerwall device, or by searching
for a distinctive fragment like `backup_reserve` or `allow_export`.

## Optional: the local gateway integration

Alongside Teslemetry (cloud), you can also add the core Home Assistant
`powerwall` integration, which talks to your Powerwall gateway directly
over your local network (LAN). This gives you a second, independent set of
sensors that update roughly every 30 seconds, rather than the Teslemetry
cloud sensors, which can lag by about a minute.

This matters more than it sounds like it should — see
[Lessons learned](05-lessons-learned.md) for a real incident caused by
trusting a laggy cloud reading. The short version: **use the local gateway
sensors whenever you're checking "did the gateway actually do the thing I
asked"**, and use the cloud (Teslemetry) entities for everything else,
including all of the settings you write to (mode, reserve, export rule,
grid charging).

To set it up, add the **Powerwall** integration from
**Settings → Devices & Services → Add integration**, and point it at your
gateway's local IP address. You'll need your gateway's Wi-Fi network
password (or whatever local credential your gateway uses) to pair it —
check Tesla's own documentation for your specific hardware, since this
varies by Powerwall generation.

Some local gateway setups can also control mode, reserve, export rule and
grid charging directly over the LAN, which avoids Tesla's cloud command
rate limit entirely (see [Lessons learned](05-lessons-learned.md)) — but
the tariff itself (the `time_of_use` call) is a cloud-only operation even
then. Whether local control is available, and how to pair it, depends on
your integration version; check its documentation.

## Sign conventions — the bit that trips people up

Power sensors have a sign (positive or negative), and the convention is not
always intuitive:

- **Site power** (`sensor.powerwall_site_power` / the Teslemetry
  equivalent): **positive means importing** from the grid, **negative
  means exporting** to the grid.
- **Battery power** (`sensor.powerwall_battery_power` / the Teslemetry
  equivalent): **positive means discharging** (the battery is supplying
  power), **negative means charging** (the battery is taking power in).

So, for example, "is it charging?" means "is battery power clearly
negative?", and "is it exporting?" means "is site power clearly negative?".
Get this backwards and you'll convince yourself a charge recipe is doing
nothing when it's actually working perfectly (or vice versa).

## Confirming a tariff push landed

Teslemetry exposes the current tariff as calendar entities —
`calendar.my_home_buy_tariff` and `calendar.my_home_sell_tariff` — showing
the current period's name and price. After pushing a fake tariff, checking
these is the quickest way to confirm the push actually reached the cloud
and took effect, before you even look at the power sensors. See
[doc 6, Troubleshooting](06-troubleshooting.md) for what to do if the
calendar still shows your old tariff.

## Helpers you'll want

The example automations in [examples/home-assistant/](../examples/home-assistant/)
expect a small set of Home Assistant **helpers** (user-configurable
entities you create yourself, as opposed to ones an integration creates for
you):

- `input_boolean.powerwall_force_charge` — turn on to start a forced
  charge.
- `input_boolean.powerwall_force_export` — turn on to start a forced
  export.
- `input_number.powerwall_force_hours` — how many hours to force for (0.5
  to 6, in half-hour steps).
- `timer.powerwall_force_timer` — counts down the forced window and ends
  it automatically.

See [examples/home-assistant/helpers.yaml](../examples/home-assistant/helpers.yaml)
for the exact YAML-mode definitions, and
[examples/home-assistant/dashboard_card.yaml](../examples/home-assistant/dashboard_card.yaml)
for a simple dashboard card to control them.

---

Next: [Lessons learned →](05-lessons-learned.md) — the most important doc
in this repo. Read it before you leave any of this running unattended.
