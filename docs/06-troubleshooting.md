# 6. Troubleshooting

[← Back: Lessons learned](05-lessons-learned.md) | [Next: Octopus integration →](07-octopus-integration.md)

A quick-reference table. Find your symptom, check the likely cause, try the
fix. Most of these are explained in more depth in
[Lessons learned](05-lessons-learned.md) — follow the links for the full
story.

| Symptom | Likely cause | Fix |
|---|---|---|
| Not charging after a push | Mode/reserve were written before the tariff had settled, so they landed against the old tariff | Re-push in the right order: tariff → wait 15 s → grid charging → reserve → export rule → mode (last). See [Order of operations](05-lessons-learned.md#order-of-operations-tariff-first-mode-last) |
| Not charging after a push, and the order was correct | Gateway accepted everything but didn't act | Wait 3 minutes, check `sensor.powerwall_battery_power` (local, not cloud). If still not charging, nudge: `self_consumption` → wait 45 s → `autonomous`. See [Never trust a pushed setting](05-lessons-learned.md#never-trust-a-pushed-setting--verify-then-nudge) |
| Export reads as working but nothing reaches the grid | Checked battery power instead of grid/site power — house load is masking a stalled export | Check `sensor.powerwall_site_power` instead. See [Test export by grid power](05-lessons-learned.md#test-export-by-grid-power-not-battery-power) |
| "Account or server is rate limited" error | Too many tariff pushes in a short window | Wait at least 15 minutes before pushing again. Going forward, dedupe pushes (only push when the tariff actually changed) and keep well under ~5 pushes per 15 minutes. See [Tesla rate-limits commands](05-lessons-learned.md#tesla-rate-limits-commands--budget-your-pushes) |
| Automation stops halfway through, settings left in a bad state | A command threw an error with no `continue_on_error`, or a non-Tesla call (e.g. a supplier API) threw an error that wasn't caught | Add `continue_on_error: true` to every Tesla command. Move any supplier/third-party calls to a separate fire-and-forget script, placed last. See [Fail safe](05-lessons-learned.md#fail-safe--every-tesla-command-needs-continue_on_error-true) |
| Charge overruns the cheap window into an expensive period | Gateway finished the charge plan with a full-power sprint that crossed the boundary | End your pushed cheap window about 15 minutes before the real one ends. See [The gateway can overrun a cheap window](05-lessons-learned.md#the-gateway-can-overrun-a-cheap-window) |
| Flapping between operation modes | Two automations (or an automation and a watchdog) are both trying to manage the same situation, or a nudge toggle is too fast and racing Tesla's cloud sync | Make sure exactly one automation is "in charge" at a time, gated by a boolean. Use a 45-second gap for any mode nudge, not 5–10 seconds. See [One writer at a time](05-lessons-learned.md#one-writer-at-a-time) and [Never trust a pushed setting](05-lessons-learned.md#never-trust-a-pushed-setting--verify-then-nudge) |
| Tariff in the Tesla app doesn't match what you pushed | A linked energy-provider tariff or a Wall Connector rate-plan schedule is overriding your push, or the integration version you're on has a known bug with `time_of_use` | Check and disable any linked tariff / rate-plan features in the Tesla app. Check your integration's release notes for known `time_of_use` regressions, and consider pinning a known-good version. See [The Tesla app can fight you](05-lessons-learned.md#the-tesla-app-can-fight-you) and [Integration updates can break this](05-lessons-learned.md#integration-updates-can-break-this) |
| A helper (`input_number`/`input_datetime`/`input_select`) silently doesn't update | Used the plain domain's `set_value` (e.g. `number.set_value`) instead of the helper's own domain action | Use `input_number.set_value`, `input_datetime.set_datetime`, or `input_select.select_option` as appropriate — never the plain `number`/`datetime`/`select` version on an `input_*` entity. See [The `input_number.set_value` trap](05-lessons-learned.md#the-input_numberset_value-trap) |
| Export worked, but you got less benefit than expected while a car was charging | The meter nets import/export instantly, so exporting while a car charges just moves battery energy into the car rather than onto the grid | Pause forced exports while a car is actively charging. See [Exporting while an EV charges](05-lessons-learned.md#exporting-while-an-ev-charges) |

---

Next: [Octopus integration →](07-octopus-integration.md) covers combining
all of this with the BottlecapDave Octopus Energy integration — smart
dispatches, Saving Sessions, and free-electricity events.
