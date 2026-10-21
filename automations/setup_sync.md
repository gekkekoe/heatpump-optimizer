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

The `ts` member is optional and only the two room fields send it: it is the time the
source took its reading, and it is what tells ODIN that this number is a measurement
from a sensor rather than a value somebody typed into the Overrides box. Everything
else — a plain `{"key", "value"}` write, and any other field — is unchanged.

### How does this work?
1. Ensure your Home Assistant instance is running and the two commands above are loaded.
2. Click the blue button at the bottom of this page.
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

Create **one automation per field** — six automations if you feed all of them. Each
one pushes on a change of its source **and once when Home Assistant starts up**, so a
restart of Home Assistant — or an ODIN that was reflashed or reset while Home
Assistant was down — is caught up without waiting for the sensor to move. A push that
would change nothing is skipped.
The **Room Temperature Override** is the room temperature of that zone, from the
same channel as a Bluetooth sensor or the pump's own reading: the number replaces
whatever the pump reports for its boiler room wherever the ODIN plans, controls or
records the room — the history graph included — and the row under the box says which
input is in use and how old its last fix was. Because this blueprint also sends the
time its sensor took the reading, ODIN treats the pushed value as a measurement: it
ages like any sensor (the row then says it is holding the last fix), and the physics
learners may use it. A number you type into the box yourself has no such time, so it
never goes stale — and nothing is learned from it, because a typed temperature cannot
measure how fast a house cools. A reading between 0 and 5 °C is not forwarded (the
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

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fgekkekoe%2Fheatpump-optimizer%2Fraw%2Fmain%2Fautomations%2Fsync_odin.yaml)
