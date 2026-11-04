# Home Assistant sync configuration

## One-time setup: configuration.yaml

The blueprint below calls two RESTful commands, `odin_get` and `odin_set`. They are
not part of the blueprint file — Home Assistant reads them from
`configuration.yaml`, so add them once and restart Home Assistant. They are
generic: `{{ host }}` is filled in by each automation from its **ODIN Address**
field, so one pair serves every ODIN on the network.

```yaml
rest_command:
  odin_get:
    url: "http://{{ host }}/dashboard/state"
    method: GET
  odin_set:
    url: "http://{{ host }}/dashboard/set"
    method: POST
    payload: >-
      {%- if ts is defined and ts | int(0) > 0 %}
      {"key": "{{ key }}", "value": {{ value }}, "ts": {{ ts | int }}}
      {%- else %}
      {"key": "{{ key }}", "value": {{ value }}}
      {%- endif %}
    content_type: "application/json"
```

Without these two commands the automation fails with
`Service rest_command.odin_set not called`. Editing the file needs a Home Assistant
restart to take effect; the `rest_command.reload` action also picks them up, the
plain "Reload YAML" button does not.

The `ts` member is optional and this blueprint never sends it: ODIN stamps each write with
its own arrival, and that stamp is what makes a pushed room number a measurement — for the
hour after it. A writer that does send `ts` wins over the arrival time; a room value of `0`
clears the override together with its stamp.

### How does this work?
1. Ensure your Home Assistant instance is running and the two commands above are loaded.
2. Click the blue button at the bottom of this page.
3. Home Assistant will automatically open the import dialog.
4. You will see a preview of the blueprint and can click **"Import Blueprint"**.

---

## Sync Battery, Room Temperature, Buffer Discharge or an Electric Meter to ODIN (REST)
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
| `consumption_override_kwh` | energy, kWh — a cumulative meter total | 0 (off) .. 1000000 |

Create **one automation per field** — seven automations if you feed all of them. Each
one pushes on a change of its source **and once when Home Assistant starts up**, so a
restart of Home Assistant — or an ODIN that was reflashed or reset while Home
Assistant was down — is caught up without waiting for the sensor to move. A push that
would change nothing is skipped.
The **Room Temperature Override** is the room temperature of that zone, from the
same channel as a Bluetooth sensor or the pump's own reading: the number replaces
whatever the pump reports for its boiler room wherever the ODIN plans, controls or
records the room — the history graph included — and the row under the box says which
input is in use and how old its last fix was. Every write is stamped with its arrival, so
ODIN counts the pushed value as a measurement for an hour and the physics learners may use
it — that is what the five-minute beat is for (one typed number is stamped too, but an hour
of it cannot teach tau). A reading between 0 and 5 °C is not forwarded (the
ODIN refuses that gap: it is the difference between "override off" and "a heated
room"); set the field back to 0 in the ODIN dashboard to hand the zone back to
whatever is left.
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

The **Electric Meter** replaces the pump's own kWh counter for everything ODIN books:
the day totals, the hour rows, the COP the planner learns and the plan-vs-actual
columns. It is there for the installs where the pump cannot answer the consumption
question — the dialect has no energy block, the S0 wire sits on another circuit, or the
only meter in the house is on the P1 port. Forward whatever that meter reads: a lifetime
total or a since-midnight counter both work, because ODIN differences consecutive
readings and treats a drop as the meter resetting instead of energy given back, so a
counter that zeroes at midnight needs no extra automation. The one value that is not a
reading is exactly `0`: 0 is off, so a since-midnight counter that reports `0.000` at
midnight switches the override off for that report and the next non-zero one switches it
back, and the step itself books nothing either way. A `ts` is meaningless here and
is refused on this key: a meter's total is a level an automation re-reads, not a sample
that ages. Set the field back to 0 to hand the bookkeeping back to the pump, whose own
counter stays readable on the same card (Physics Data → Consumption) while it is
overridden.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fgekkekoe%2Fheatpump-optimizer%2Fraw%2Fmain%2Fautomations%2Fsync_odin.yaml)
