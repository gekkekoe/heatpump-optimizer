# Home Assistant sync configuration


### How does this work?
1. Ensure your Home Assistant instance is running.
2. Click the blue button above for the version you need.
3. Home Assistant will automatically open the import dialog.
4. You will see a preview of the blueprint and can click **"Import Blueprint"**.

---

## Sync Battery, Room Temperature or Buffer Discharge to ODIN (REST)
This blueprint pushes a Home Assistant sensor value straight into an ODIN on the
LAN over its REST API — no MQTT needed. Pick the field it feeds:

| ODIN field | Source sensor | Accepted values |
| --- | --- | --- |
| `battery_soc_kwh` | battery energy, kWh | 0..2000 |
| `battery_max_discharge_kw` | power, kW | 0..100 |
| `room_override_z1` | room temperature, °C | 0 (off) or 5..40 |
| `room_override_z2` | room temperature, °C | 0 (off) or 5..40 |
| `discharge_override_z1` | on/off entity, or a numeric sensor (pump power, flow) | on / above threshold → 1, off / at or below → 0 |
| `discharge_override_z2` | idem, for zone 2 | idem |

Create **one automation per field** — six automations if you feed all of them.
The **Room Temperature Override** is for an ODIN that has no room sensor of its
own, or one that reads the boiler room instead of the living room: the number
replaces the pump's own room reading wherever the ODIN plans, controls or records the
room — the history graph included — and the dashboard says which of the two is in use.
Only the building-physics learner leaves the number alone: a typed temperature cannot
measure how fast the house cools. A reading between 0 and 5 °C is not forwarded (the
ODIN refuses that gap: it is the difference between "override off" and "a heated
room"); set the field back to 0 in the ODIN dashboard to hand control back to the
sensor.
The **Discharge Override** switch tells the ODIN that heat is leaving the store into a
zone's loop while the pump's own zone-pump report says nothing is moving — an emitter
loop driven from outside the pump. It is a switch on the ODIN, so the source is read as
a boolean: an `on`/`off` entity (binary sensor, input boolean, switch) maps to 1/0, a
numeric sensor — the external pump's power, a flow sensor — counts as discharging above
the **Discharge Threshold**. The flag is OR'd with what the pump reports, so it can add
a discharge and never hide one. While it is 1, no idle stretch of that zone counts as
free drift and the ODIN stops learning that zone's heat-loss time constant (tau) and
passive solar gain; that is the argument for driving it from the pump's own sensor and
letting the automation write 0 again when it stops, instead of leaving the dashboard
switch on. Values are only written when they differ from what the ODIN holds, so the
flash is not written on every sensor update.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fgekkekoe%2Fheatpump-optimizer%2Fraw%2Fmain%2Fasgard%2Fha_blueprints%2Fsync_odin.yaml)
