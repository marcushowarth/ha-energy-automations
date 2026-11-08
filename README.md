# Home Assistant energy and battery automations

![Home Assistant](https://img.shields.io/badge/Home%20Assistant-41BDF5?logo=home-assistant&logoColor=white)
![Fox ESS](https://img.shields.io/badge/Fox%20ESS-Modbus-E60012)
![Octopus Flux](https://img.shields.io/badge/Octopus-Flux-5840FF)
![Solcast](https://img.shields.io/badge/Solcast-PV%20forecast-F5A623)
![YAML](https://img.shields.io/badge/YAML-CB171E?logo=yaml&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-green)

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
4. **Peak (16:00 to 19:00).** If the rest of today's forecast is above a threshold, *set flag to force peak export* turns on peak export. The inverter then Force Discharges down to a target SoC, holds in Feed-in First, and goes back to Self Use when SoC drops below target (*Battery Switch Work mode ...*).
   - **Tuning, being monitored.** After comparing against [Predbat](https://github.com/springfall2008/batpred)'s plans, the forecast threshold is now very low (0.25 kWh) and the export target SoC lower (65%). That still exports at the peak on most days, even in October, and leaves enough for the evening so the house runs on the battery until the eco rate starts.

### Charge offsets

The charge periods switch on when SoC is at or below the target **minus an offset** (`input_number.eco_charge_offset`, `input_number.daytime_charge_offset`, both 2%), and switch off only once the target itself is reached. Without the gap, a battery sitting at or near its target would start a grid charge for a fraction of a percent, stop, drift down with the house load, and start again. The free electricity slot uses the same idea (`input_number.free_electricity_pre_soc_offset`, 3%) before it Force Discharges.

### Throughout the day

These aren't tied to one point in the day; they react whenever their condition is met.

- **Tariff tracking:** *Octopus Flux Tariffs* and the *apply tariff for the current time* script switch the utility meters between eco, day and peak, so the Energy dashboard prices every kWh correctly.
- **Charge reduction:** charge current drops to 4 A above 93% SoC to be kind to the cells. The idea came from [this Fox ESS community thread](https://foxesscommunity.com/viewtopic.php?t=946).
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
