# The Home Assistant side

A proxy on its own gives Home Assistant a remote Bluetooth adapter. Everything
interesting downstream of that is four integrations.

## Adoption

The device appears under Settings → Devices & Services → **ESPHome**,
"Discovered". Configure it, paste the encryption key from `secrets.yaml`, done.
Home Assistant registers it as a Bluetooth adapter automatically — check
Settings → Devices & Services → **Bluetooth** and it will be listed as a remote
adapter. Any BLE integration you already have can now reach devices through it.

## Entities each proxy gives you

**Tuning** (these are the two you will actually touch)

| Entity | |
|---|---|
| `number.*_ble_rssi_threshold` | −90…−40 dBm, default −75. Live, no reflash. |
| `select.*_ble_scan_profile` | Insane / Aggressive / Balanced / Disabled |
| `switch.*_ble_active_scan_onboarding` | Temporarily enable active scanning to discover a new device by name. Resets to off on reboot. |

**Filter diagnostics** (60-second deltas)

| Entity | |
|---|---|
| `sensor.*_ble_adverts_forwarded` | adv/min reaching Home Assistant |
| `sensor.*_ble_adverts_dropped` | adv/min suppressed on-device |
| `sensor.*_ble_rpas_dropped` | other people's phones and watches |
| `sensor.*_ble_service_uuid_allowed` | forwarded *only* by the UUID allowlist — zero unless something is pairing |
| `sensor.*_ble_advert_drop_rate` | % — the tuning number |

**Health**

`sensor.*_gateway_packet_loss`, `sensor.*_gateway_watchdog_reboots_used`,
`sensor.*_internal_temperature`, `sensor.*_wifi_signal`,
`text_sensor.*_uptime`, `*_reset_reason`, `*_esphome_version`,
`*_project_version`, `*_ip`.

`*_project_version` is worth a dashboard card across the fleet: it carries
`+ble<ble_proxy_version>`, so a straggler on an old filter build is visible at a
glance.

---

## Bermuda — room-level presence

[github.com/agittins/bermuda](https://github.com/agittins/bermuda) (HACS).

Bermuda takes the RSSI readings every proxy reports for a device, compares them,
and decides which proxy — and therefore which **area** — the device is nearest.
It is the reason the whole fleet exists.

Configuration that matters here:

| | This fleet | |
|---|---|---|
| `ref_power` | −55 | RSSI at 1 m |
| `attenuation` | 3 | path-loss exponent |
| `max_area_radius` | 12 m | |

Those two give `distance = 10 ^ ((-55 - rssi) / 30)`, which is where the
threshold-to-distance table in [tuning.md](tuning.md#the-rssi-threshold) comes
from. It polls every 10 s with 20-sample smoothing, so nothing responds
instantly and a 37.5% scan duty cycle is plenty.

**Bermuda has no RSSI cutoff of its own.** `max_area_radius` is applied in Home
Assistant *after* the packet has already crossed the network. The on-device
threshold is the only thing that reduces actual traffic — which is the whole
argument for the fork.

**Set each proxy's `area`.** It comes from the `area:` substitution in the
device file, so it is right by construction if you filled that in. Bermuda's
output is only as good as those labels.

**You need at least three proxies** for trilateration to mean anything, and
realistically one per room for room-level results.

---

## Private BLE Device — the IRK mirror

The core `private_ble_device` integration tracks phones and watches by their
Identity Resolving Key across address rotations.

**Keep its IRKs and the proxies' `irks:` list identical.** They are the same
values doing the same job at two different layers:

- `private_ble_device` resolves the RPA in Home Assistant, so you get a device
  tracker.
- The proxy's `irks:` resolves it *on-device*, so an RPA belonging to somebody
  else never crosses the network at all.

An IRK in one list and not the other means either a phone you can't see, or a
phone whose advertisements are being dropped before Home Assistant hears them.
`common/ble-proxy.yaml` reads them from `secrets.yaml` so both BLE packages
share one list.

---

## BPS — receiver auto-calibration

Every proxy advertises an iBeacon:

```yaml
# common/ble-proxy.yaml
substitutions:
  ble_beacon_uuid: fde3b150-2f64-43ba-aee9-867f75ee4a6f   # <- set your own
  ble_beacon_major: "1"

esp32_ble_beacon:
  type: iBeacon
  uuid: ${ble_beacon_uuid}
  major: ${ble_beacon_major}
  minor: 1
  min_interval: 500ms
  max_interval: 1000ms
```

BPS reads the resulting probe-to-probe measurements through
`bermuda.dump_devices` and fits a **per-receiver distance correction** — every
ESP32 has slightly different receive sensitivity, and comparing measured
probe-to-probe distances against their known placed positions is what reveals
each one's error. Nothing needs configuring in Bermuda.

Details worth knowing before you change any of it:

- **`uuid`/`major`/`minor` are not how BPS identifies a probe.** It matches on
  the Bluetooth MAC, which ESP-IDF derives from the same base as the scanner's
  own address. The payload is just a carrier — which is why `minor` is a
  constant rather than derived per device.
- **The beacon and the `allow_ibeacon` rule read the same two substitutions**,
  so they cannot drift apart. iBeacon is Apple manufacturer data subtype `0x02`,
  so the `0x004C` entry in `manufacturer_blocklist` drops *every* iBeacon —
  including your own probes — unless `allow_ibeacon` exempts them. Without it
  the calibration matrix comes back empty.
- **Set your own `ble_beacon_uuid`, the same on every proxy you own.** The uuid
  is what scopes the rule to *your* beacons (fork v1.7.0+). Before that the rule
  was `major: 1` alone, and major 1 is the commonest vendor default there is —
  anybody's beacon left on its defaults was forwarded too, at the loose −95
  limit. `uuidgen` makes one; the [wizard](https://davidcoulson.github.io/esphome-ble-proxy/)
  generates one and remembers it. The value shipped in this repo is this
  fleet's — harmless to reuse, but then your rule also admits beacons from
  anyone else who did the same.
- **`rssi: -95` on that rule, not −127.** −127 would forward at any strength, but
  it also disables the component's internal pre-gate: with something allowed
  through unconditionally, nothing can be rejected on RSSI alone, so every
  advertisement reaches the categoriser. That is real work on a single-core C3.
  −95 sits at the ESP32's own receive sensitivity and keeps the gate.
- **The weak cross-room pairs carry the geometry.** At the fleet's −75 threshold
  the calibration matrix came back mostly empty, which is why this rule is 20 dB
  looser.
- 500–1000 ms is slow for a beacon, deliberately — the radio is shared, and
  calibration samples over minutes.

The corresponding YAML is `esp32_ble.advertising: true` on the families that set
it (rrn00, s2224). A proxy that doesn't advertise contributes nothing to
calibration.

---

## Recorder load

~70 proxies × a handful of 60-second sensors adds up. The fleet trims hard, and
the reasoning generalises:

- Mark anything static or purely local `internal: true` — it stays on the
  device's own web UI and in lambdas, but costs no entity registry slot and no
  recorder row.
- Prefer sensors that **rarely change state**. Gateway packet loss sits at 0.0
  almost always, so it's nearly free. Gateway RTT is a float average that is
  unique on *every* publish — at 60 s × ~90 nodes that was ~130k rows/day plus
  five-minute long-term statistics, which is why it is internal.
- Anything with `state_class: measurement` also generates long-term statistics.
  Worth it for the drop-rate sensors; not worth it for RTT.
