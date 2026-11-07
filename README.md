# Home Assistant energy and battery automations

The automations and scripts that run a home battery on a time-of-use tariff, taken from a real, daily-running Home Assistant setup. The battery charges cheaply overnight, by an amount set from tomorrow's solar forecast, tops up before the evening peak, sells at the peak when the forecast allows, and takes full advantage of Octopus free electricity sessions.

`automations.yaml`, `scripts.yaml` and `ENTITIES.md` are **generated** from a private config (see [How this repo is published](#how-this-repo-is-published)), so they are exactly what runs, with personal IDs replaced by placeholders.

## The setup

| Part | What it is |
|---|---|
| Inverters | Two Fox ESS: an H1-5.0-E hybrid with 7 HV2600 batteries (13.75 kWh usable), and an H1-3.7-E for PV only |
| Inverter control | [foxess_modbus](https://github.com/nathanmarlor/foxess_modbus), local Modbus: work mode, charge periods, min/max SoC, charge current |
| Tariff | Octopus **Flux**: eco (overnight), day and peak (16:00 to 19:00) rates for import and export |
| Rates | [Octopus Energy](https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy) integration, copied into `input_number` helpers daily |
| Solar forecast | [Solcast](https://github.com/BJReplay/ha-solcast-solar) |
| Logging | Google Sheets integration (one row per event), a Google calendar, and email |

## How a day runs

1. **Evening, tomorrow's forecast.** *set Max SoC ECO based on tomorrow's PV forecast* picks how full to charge overnight: a sunny day needs less grid charge.
2. **Overnight (eco rate).** *Enable / Disable 1st charge period during Eco Tariff period* turn on the grid charge while SoC is below that target, and off once it's reached. *Battery Max SoC / Min SoC setting* move the inverter's limits at each tariff boundary.
3. **Before the peak.** *Enable / Disable 2nd charge period before Peak Tariff period* top up from the grid at the day rate if the battery is short of the peak target.
4. **Peak (16:00 to 19:00).** If the rest of today's forecast is good, *set flag to force peak export* turns on peak export. The inverter then Force Discharges down to a target SoC, holds in Feed-in First, and goes back to Self Use when SoC drops below target (*Battery Switch Work mode ...*).
5. **All day.**
   - **Tariff tracking:** *Octopus Flux Tariffs* and the *apply tariff for the current time* script switch the utility meters between eco, day and peak, so the Energy dashboard prices every kWh correctly.
   - **Charge reduction:** charge current drops to 4 A above 93% SoC to be kind to the cells.
   - **Cooking bump:** *alternative cooking value* raises the day Max SoC when the oven or a heater is drawing a lot.
   - **Full-battery log:** a row goes into the sheet the first time the battery hits 100% each day.

### Free electricity sessions

*Energy: Free electricity slot (armed)* is the most involved one. You set a slot's start and end on the dashboard and arm it, then:

| When | What happens |
|---|---|
| Night before | Cap the overnight charge at the pre-slot target |
| Before the slot | Force Discharge down to the target. The lead time comes from the current SoC (13.75 kWh, about 3 kW discharge, plus 30 minutes). Then hold in Feed-in First |
| During the slot | Grid charge to 100%; the workshop and heat pump meters are priced as free |
| At the end | Restore Self Use, the normal charge period and Max SoC, disarm, re-apply the tariff, and email a start and end summary |

Two scripts, *Free Electricity Day Tomorrow* and *Free Electricity Day Today*, are the older, manual way to do the same.

### Other

- *Update Octopus Energy Tariffs with Change Detection* copies the latest rates into helpers, then logs and emails which ones changed.
- *Sensors: Weekly battery report* emails the battery levels of every battery-powered sensor and flags low or silent ones.

## Using it yourself

- **Placeholders:** `YOUR_INVERTER_DEVICE_ID`, `YOUR_GOOGLE_SHEETS_ENTRY_ID` and the `meterserial_*` part of the Octopus sensor names stand in for the real IDs. `notify.email_me` is any notify entity.
- **Entities:** [ENTITIES.md](ENTITIES.md) lists every helper and sensor referenced. Create the helpers, or rename them to yours.
- **Fox ESS naming:** the inverter is named `workshop` in Home Assistant, so its entities are `sensor.workshop_*`, `select.workshop_work_mode` and so on.
- **Testing:** read an automation before turning it on. They change real inverter settings.

## How this repo is published

The full config lives in a private repo. Automations and scripts get a `public` **label** in the Home Assistant UI, and an export script:

1. reads which entities carry the label from Home Assistant's entity registry,
2. pulls those blocks out of `automations.yaml` / `scripts.yaml` without reformatting them,
3. replaces known personal IDs (inverter, config entries, meter serials, notify targets) with placeholders,
4. runs a leak check for email addresses, postcodes, street addresses, private IPs, device and config entry IDs, long meter numbers and internal hostnames, and **writes nothing if any match**.

So what's public is chosen in the Home Assistant UI with one label, and everything stays editable there.

## Licence

[MIT](LICENSE)
