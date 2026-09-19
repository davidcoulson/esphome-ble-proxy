# IRKs from Home Assistant

Every proxy subscribes to one Home Assistant entity and reloads its IRK list
whenever that entity changes. A new phone becomes an edit in HA: no reflash and
no reboot, and every proxy picks it up within a second.

Needs the `bluetooth_proxy` fork **v1.6.0+** (`set_irks()`). The ESPHome side is
`config/common/irks-from-ha.yaml`, which is already included by
`common/ble-proxy.yaml` and by the quick-flash firmware.

## Home Assistant side

The list lives in HA's `secrets.yaml`, with a name next to each key:

```yaml
# /config/secrets.yaml (Home Assistant's, not ESPHome's)
ble_proxy_irks: |
  David phone:   00112233445566778899aabbccddeeff
  David watch:   ffeeddccbbaa99887766554433221100
  Jack phone:    0123456789abcdef0123456789abcdef
```

A template sensor publishes it as an **attribute**:

```yaml
# /config/packages/ble_proxy_irks.yaml
template:
  - sensor:
      - name: "BLE Proxy IRKs"
        unique_id: ble_proxy_irks
        icon: mdi:key-chain
        # A fixed word, deliberately. The keys go in the attribute; the state
        # is what shows up in lists and logbooks.
        state: "configured"
        attributes:
          irks: !secret ble_proxy_irks

# Keep the keys out of history. Without this every change is written to the
# database, and ends up in backups.
recorder:
  exclude:
    entities:
      - sensor.ble_proxy_irks
```

If you already have a `recorder:` block in `configuration.yaml` and HA
complains about the package, move that `exclude` entry into it.

Reload template entities (Developer Tools → YAML → Template entities), or
restart HA. Then check any proxy's **BLE IRKs Loaded** sensor: it should show
the number of keys you listed.

### Why an attribute and not the state

HA caps an entity's state at 255 characters. Eight IRKs are 256 hex characters
before any separator or name. Attributes have no such cap, and ESPHome's
`homeassistant` text sensor can subscribe to one.

## Format

The proxy takes every run of **exactly 32 hex characters** as a key and ignores
everything else. So the names, colons, commas and newlines are for you, not for
the proxy. Anything readable works:

```
David phone: 0011…eeff
0011…eeff, ffee…1100
{"David phone": "0011…eeff", "David watch": "ffee…1100"}
```

The rules that follow from that:

- **Names must not contain 32 hex digits in a row.** "Dad", "Faded" and "AC"
  are fine; a name that is itself a hex string is not.
- **Keys need a separator.** Two keys written back to back are 64 digits, which
  is rejected rather than split in half.
- **Hex only.** Same format as the `ble_irk_*` values in ESPHome's
  `secrets.yaml`, and parsed by the same function, so the bytes are identical.
  If a device gave you a base64 IRK, convert it first.
- **Up to 16 keys** survive a reboot (that is what the flash copy holds).

## Replacing one phone

Edit its line in HA's `secrets.yaml`, then reload template entities. Every proxy
reloads its list; **BLE IRKs Loaded** confirms the count. Because each line is
named, you can see exactly which key you're replacing.

Keep `private_ble_device` in step: it's the same key doing the other half of the
job (tracking the phone in HA, where the proxy only decides whether to forward
it).

## What happens when things go wrong

| Situation | Proxy behaviour |
|---|---|
| HA restarting, entity `unavailable` | Ignored. Keeps its current list. |
| Attribute contains no valid key (typo, empty) | Ignored. Keeps its current list. |
| Some lines valid, some junk | Loads the valid keys; the junk is dropped. |
| Proxy boots before HA connects | Uses the last list HA delivered, saved in flash. |
| Brand-new proxy, HA never delivered anything | Uses the compile-time `irks:` list, if any. |
| You actually want no IRKs | Set the attribute to the word `clear`. |

Refusing to clear on bad input is the important one. With the Apple
manufacturer blocklist on, a wiped IRK list doesn't fail loudly: it silently
drops your own phones, because the IRK match is what protects them from that
blocklist.

## Precedence

1. The HA entity, whenever it delivers at least one valid key
2. The last list HA delivered, from flash
3. The compile-time `irks:` list, on first boot only

Once HA has delivered a list, that list is the source of truth, even across
reflashes. Editing `irks:` in ESPHome's `secrets.yaml` no longer changes
anything on a proxy that has already heard from HA. **Edit the list in HA.**

## Using a different entity

Override the substitutions, per device or in a family package:

```yaml
substitutions:
  irk_entity: sensor.my_irks
  irk_attribute: keys
```

## Security

The keys now exist in three places: HA's `secrets.yaml`, the template sensor's
attribute, and each proxy's flash.

- **The attribute is visible** to any HA admin in Developer Tools → States. It
  is kept out of the recorder by the exclude above. It can still appear in a
  diagnostics download (the same exposure as
  [agittins/bermuda#839](https://github.com/agittins/bermuda/issues/839)).
- **It travels over the ESPHome API.** The fleet encrypts that. Quick-flash
  firmware does not: it is a generic binary with no key to pre-share. Adopting
  it in Home Assistant does not change that; adopting it into an ESPHome
  dashboard and building it with `api: encryption:` does. On an untrusted
  network, do that before pointing it at your IRKs.
- **An IRK identifies a person's phone.** Anyone holding one can recognise that
  phone's rotating address. Treat the list like a password.
