# esphome-ble-proxy

The ESPHome Bluetooth proxy fleet — configuration, the reasoning behind it, and
a browser flasher so you can have one running without reading any of it.

Roughly 70 nodes feed Bluetooth advertisements into Home Assistant here, where
[Bermuda](https://github.com/agittins/bermuda) turns them into room-level
presence. The interesting part is not that they proxy BLE — stock ESPHome does
that in four lines — it is that they **filter on-device**, before a packet ever
crosses the network. A stock proxy forwards every advertisement it hears. In a
dense house that is a firehose of other people's AirPods.

**→ [Flash one from your browser](https://davidcoulson.github.io/esphome-ble-proxy/) — pick a board, plug it in over USB, click. Wi-Fi is set up over Improv; you never see any YAML.**
**→ [QUICKSTART.md](QUICKSTART.md) — the ten-minute version if you'd rather write the config yourself.**
**→ [The config generator](https://davidcoulson.github.io/esphome-ble-proxy/#advanced) — answer six questions, get YAML.**

---

## What makes these different

| | Stock `bluetooth_proxy` | This |
|---|---|---|
| Advertisement filtering | none — forwards everything | RSSI floor + per-category limits, IRK matching, MAC allow/blocklist, manufacturer blocklist, service-UUID passthrough |
| Other people's phones | forwarded | dropped (RPA resolves to none of your IRKs) |
| Apple noise (~69% of traffic) | forwarded | dropped, with HomeKit / FindMy / iBeacon carve-outs |
| Visibility | none | forwarded / dropped / drop-rate sensors per proxy |
| Tuning | reflash | Home Assistant number + select entities, live |

Measured effect: **~38% fewer bytes** from the RSSI filter alone, and **56–66%
of all advertisements dropped** once IRK filtering is on.

That filtering comes from a fork of the core component,
[`esphome-bluetooth-proxy-filter`](https://github.com/davidcoulson/esphome-bluetooth-proxy-filter),
pulled in by pinned tag. See [docs/filtering.md](docs/filtering.md) for the
filter chain in execution order and what every knob does.

## The fleet

| Family | Count | Board | Link | Notes |
|---|---:|---|---|---|
| `rrn00` | 38 | ESP32-C3 WROOM-03/06, ESP32-C6 | WiFi | In-wall dual outlets. Proxy runs alongside two relays |
| `s2224` | 11 | ESP32-C3 XH-C3F / ESP8685 | WiFi | Dual outlets |
| `shelly` | 6 | ESP32 (unicore, 160 MHz) | WiFi | Reflashed Shellys |
| `esp32-eth` | 5 | ESP32-S3 + W5500 | **Ethernet** | Best hardware here — no radio contention |
| `esp32c5` | 4 | ESP32-C5 | WiFi **5 GHz** | 5 GHz takes WiFi off BLE's band |
| `esp32s3` | 3 | ESP32-S3 | WiFi | Dedicated proxies, nothing else on them |
| `cf-sw1` | 2 | ESP8685 | WiFi | CloudFree SW1 switches |
| `xiao` | 1 | XIAO ESP32-C6 | WiFi | |

If you are placing new hardware: **prefer Ethernet, then a C5 on 5 GHz, then a
dedicated S3.** Everything else shares one 2.4 GHz radio between WiFi and BLE,
and that trade-off is the single biggest constraint in this whole repo —
[docs/tuning.md](docs/tuning.md) has the numbers.

## Two ways in

**Quick flash.** Prebuilt firmware for seven boards, installed from the browser
over Web Serial, with Wi-Fi provisioned over Improv down the same USB cable. No
ESPHome install, no YAML, no secrets file.

The wizard asks two questions, because both are compile-time and cannot be
changed afterwards:

| | |
|---|---|
| **Block Apple noise?** | *Yes* drops Apple manufacturer data — ~69% of what a proxy hears — while keeping HomeKit accessories and FindMy tags. *No* forwards it. |
| **Relay connections?** | *Yes* lets Home Assistant reach locks, LED strips, SwitchBots through the proxy. *No* is advertisements only: lighter, and the radio is never pulled off scanning. |

That is 4 variants × 7 boards, all built ahead of time. Everything else — RSSI
threshold, RSSI floor, scan profile — is a Home Assistant entity you can change
live without reflashing.

**IRKs come from Home Assistant, not the build.** Blocking Apple would otherwise
drop your own iPhones too: an advertisement is protected from the blocklist
only when its rotating address matches an IRK. Every proxy subscribes to one HA
entity holding the list, with a name next to each key. Adding or replacing a
phone is an edit in HA; every proxy reloads within a second, with no reflash.
[docs/irks-from-ha.md](docs/irks-from-ha.md).

**Bring your own ESPHome.** Write a device file, use the shared packages, keep
full control. That is the rest of this repo.

## Repo layout

```
config/
  common/          the shared packages — this is the actual configuration
  devices/         one example device file per family (they are 4–8 lines each)
  adopt/           secret-free packages behind the prebuilt firmware, and the
                   dashboard_import target a flashed device is adopted from
  secrets.yaml.example
firmware/          board configs, variants/, and compose.sh — CI builds board x variant
docs/
  architecture.md  how the packages compose, and why it is layered this way
  filtering.md     the fork: filter chain, every option, what to allowlist
  tuning.md        RSSI thresholds, scan profiles, the radio-sharing problem
  hardware.md      boards, pinouts, what to buy
  operations.md    flashing, OTA at scale, safe mode, the watchdog
  troubleshooting.md  symptom → cause → fix
  home-assistant.md   Bermuda, BPS, private_ble_device, the entities you get
  upgrading.md     the ESPHome bump runbook (re-sync the fork!)
  irks-from-ha.md  change IRKs in Home Assistant instead of reflashing
wizard/            the browser flasher + config generator, served by GitHub Pages
```

## How a device file works

Every node is a handful of substitutions and one include. All the real
configuration lives in `config/common/`, composed in layers:

```yaml
substitutions:
  mac: a1b2c3
  area: "Kitchen"

packages:
  device: !include ../common/esp32s3-ble.yaml
```

That family package pulls in the ESP32 baseline, WiFi, the BLE proxy package,
the gateway watchdog and the diagnostic set. Change `common/ble-proxy.yaml` once
and all ~70 proxies change. [docs/architecture.md](docs/architecture.md)
explains the layering and the `!extend` / `!remove` conventions that let a
family override a shared default without forking it.

## Conventions worth knowing before you edit anything

- **Comments are the documentation.** The packages carry long rationale
  comments, including incident history. Preserve them; add to them.
- **Bump `ble_proxy_version`** in `common/ble-proxy.yaml` on *any* change there.
  Families append it to their project version, so Home Assistant shows which
  filter build each node is running.
- **External components are pinned by tag, sourced by git URL** — never
  `type: local`. A local path only exists on one box, and builds get offloaded
  to remote builders.
- **Re-sync the fork on every ESPHome version bump.** It is a copy of a
  fast-moving core component. See [docs/upgrading.md](docs/upgrading.md).
- **Never mass-flash.** A fleet-wide OTA once put ~40 nodes into a reboot loop.
  Batch by family, a handful at a time. See [docs/operations.md](docs/operations.md).

## Secrets

Nothing in this repo contains a credential — every key is a `!secret`
reference. Copy `config/secrets.yaml.example` to `secrets.yaml` next to your
device files and fill it in.

The IRK keys are named `ble_irk_phone_1..4` / `ble_irk_watch_1..4` here. The
live fleet names them per person; if you are syncing between the two, that is
the one difference to reconcile.

## Credits

- [ESPHome](https://esphome.io) and its `bluetooth_proxy` component, which this
  forks rather than replaces
- [Bermuda BLE Trilateration](https://github.com/agittins/bermuda)
- [DerekSeaman/irk-capture](https://github.com/DerekSeaman/irk-capture) — the
  IRK capture component behind `common/irk-capture-base.yaml`
