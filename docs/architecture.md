# Architecture

Every device file is four substitutions and one include. All the configuration
lives in `config/common/`, composed in layers, so changing one shared package
changes all ~70 proxies at once.

```
device file            substitutions: mac, area, [module]
  └─ family package    esphome.name, board pins, relays, status LED
       ├─ djc-common.yaml         ESP32 baseline: IDF, ota, api, mdns, web_server,
       │                          safe_mode, buttons, uptime/temp/version sensors
       ├─ ble-proxy.yaml          the proxy itself + filtering + tuning entities
       ├─ esp32-ble-platform.yaml sdkconfig trims shared by the C3/C5/C6 families
       ├─ wifi.yaml  OR  eth.yaml transport
       ├─ gateway-watchdog.yaml   reachability watchdog (WiFi families only)
       └─ debug-lite.yaml         reset-reason diagnostic
```

## The layers

| File | Role |
|---|---|
| `djc-common.yaml` | ESP32 baseline. ESP-IDF **pinned 6.1.0**, `min_version 2026.9.0`, 10 s task watchdog, logger WARN, `ota` (`id_ota`, partition access), `api` (`reboot_timeout: 0s`), mdns, web_server v2, safe_mode, restart / safe-mode / factory-reset buttons, uptime / internal-temperature / version sensors, Home Assistant as the only time source. |
| `ble-proxy.yaml` | The whole proxy: the forked component, `esp32_ble_tracker`, `bluetooth_proxy` with its filter config, the iBeacon for BPS calibration, and the five diagnostic sensors + two tuning entities. **The one file worth reading in full.** |
| `esp32-ble-platform.yaml` | Shared sdkconfig trim for the C3 / C5 / C6 proxy families. Variant-agnostic on purpose: the family sets `esp32.variant` / `flash_size` / `cpu_frequency`, this only tunes the framework. |
| `wifi.yaml` | WiFi STA, WPA2 minimum, BTM/RRM on, 17 dB, no power save, 5 min `reboot_timeout`. RSSI as dB (internal) and %; MAC/BSSID internal. |
| `eth.yaml` | W5500 SPI on the Waveshare ESP32-S3-ETH pinout, plus `ethernet_info` sensors. Boards with different wiring override the pins in their device file. |
| `gateway-watchdog.yaml` | Pings the default gateway, reboots on link-but-no-reachability. Capped at 2 reboots per 1 h budget. WiFi families only — see below. |
| `debug-lite.yaml` | `debug:` at 24 h exposing only Reset Reason. `debug.yaml` (full heap/loop set) is opt-in and not in this repo. |
| `irk-capture-base.yaml` | Standalone tool package, not part of a proxy. Flash to a spare ESP32/S2/S3 to read IRKs off a phone. |

## Family packages

One per hardware type. Each sets `esphome.name`, the project name/version, the
board variant, pins, and any relays or buttons the hardware has.

| Family | Devices | Transport | Composition |
|---|---:|---|---|
| `rrn00.yaml` | 38 | WiFi | gwwd + djc + platform + wifi + ble + debug; `module: 3｜6｜c6` selects pins and variant |
| `s2224-esp32c3.yaml` | 11 | WiFi | same; `module: xh-c3f｜esp8685` |
| `shelly-ble.yaml` | 6 | WiFi | gwwd + djc + ble + wifi + debug (classic ESP32, unicore, 160 MHz) |
| `esp32-eth-ble.yaml` | 5 | **Ethernet** | djc + ble + eth + debug — **no wifi, no gateway watchdog** |
| `esp32s3-ble.yaml` | 3 | WiFi | djc + ble + wifi + debug + gwwd |
| `cf-sw1-esp8685.yaml` | 2 | WiFi | same as s2224, plus the SW1's relay and button |

The C5 and XIAO-C6 proxies have no family package — too few of them, and each
carries its own status-LED wiring. They compose the layers directly in the
device file; `config/devices/esp32c5-a1b2c3.yaml` is the worked example.

## Conventions the packages rely on

### Override, don't fork

Family files adjust shared defaults with `!extend <id>` and `!remove`, never by
copying the package and editing it:

```yaml
ota:
  - platform: esphome
    id: !extend id_ota      # add an on_begin hook to the shared ota block
    on_begin:
      then:
        - esp32_ble_tracker.stop_scan:
```

```yaml
ethernet:
  reset_pin: !remove        # this board has no reset line
```

The ids exist purely so this works: **`id_ota`, `factory_reset_btn`,
`reboot_btn`, `esp32_uptime`, `project_name_sensor`,
`bootloader_version_sensor`** in `djc-common.yaml`; **`ble_tracker`,
`ble_proxy`, `sel_ble_scan_profile`, `num_ble_rssi_threshold`,
`sw_ble_active_scan`** in `ble-proxy.yaml`. Do not rename them.

### Version everything

`common/ble-proxy.yaml` carries its own version:

```yaml
substitutions:
  ble_proxy_version: "2026.09.17.1"
```

Every family appends it to its project version:

```yaml
version: "2026.09.17.0+ble${ble_proxy_version}"
```

so a change to the shared BLE package shows up in `sensor.*_project_version` on
every proxy without editing 70 files. **Bump it on any change to
`ble-proxy.yaml`.** The trailing digit is a same-day revision counter — without
it a second edit on the same date produces an identical version, and devices
look up to date when they are not.

The one exception: if the fork's *compiled code* did not change (as between
v1.3.2 and v1.4.0, where upstream moved one Python type annotation and nothing
else), don't bump it. A bump would force ~70 byte-identical OTAs.

### Reboot policy

Deliberately inconsistent, and the inconsistency is the point:

| Node type | `api.reboot_timeout` | Also has |
|---|---|---|
| WiFi proxies | **`0s`** | `wifi.reboot_timeout: 5min` + gateway watchdog |
| **Ethernet proxies** | **`300s`** | neither of the above |

`api.reboot_timeout: 0s` is the default because "Home Assistant went away for
five minutes" is not "this node is broken" — a core upgrade or recorder
migration once rebooted the whole fleet at once. WiFi nodes still recover from a
wedged radio via `wifi.reboot_timeout`, and from "associated but unreachable"
via the gateway watchdog.

Ethernet nodes have **neither** — no WiFi component, no watchdog — so the API
timeout is their only automatic recovery. The cost is a reboot during any
5-minute Home Assistant outage, which is accepted. **This is intentional; do not
"fix" it.**

### The gateway watchdog

ESPHome ships no watchdog for "link up, gateway unreachable". `wifi`'s
`reboot_timeout` is an *association* watchdog — it refreshes on every loop while
the radio is associated, so a dead VLAN, gateway or uplink never trips it.

[`davidcoulson/esphome-gateway-watchdog`](https://github.com/davidcoulson/esphome-gateway-watchdog)
follows whichever netif holds the default route and pings the DHCP-supplied
gateway, so the include needs no per-node configuration.

It is capped at **2 reboots per 1 h budget** for a reason: a fleet-wide OTA once
put ~40 nodes into a rolling reboot loop. A deliberate 55-device reproduction
produced zero packet loss on 35 control nodes, so congestion is not the cause
and the real mechanism is still unknown. `max_reboots` is the backstop that does
not care — two reboots is recovery, the third is a loop, and a node that stays
up reporting 100% loss is far more useful than one power cycling every few
minutes.

### Diagnostic trim

The defaults are trimmed because ~90 nodes were flooding the recorder. Keep new
entities `internal: true` or `entity_category: diagnostic` unless Home Assistant
genuinely needs them.

Notable: Uptime publishes on an **adaptive interval** — 10 s for the first five
minutes, 30 s to fifteen, 60 s to an hour, then 600 s. A fixed 900 s meant a
node that had just rebooted sat on a stale value for a quarter of an hour, which
is exactly the window in which a reflash is worth watching.

Gateway RTT is internal for a specific reason: unlike packet loss (which sits at
0.0 almost always, so no state change and no recorder row), RTT is a float
average that is unique on **every** publish. At 60 s × ~90 nodes that was
~130k rows/day plus five-minute long-term statistics.

### External components: git URL, pinned tag

Never `type: local`. Two reasons:

1. **Remote builders.** Builds get offloaded to paired build servers; a local
   path only exists on the Home Assistant box, so the build would fail to
   resolve the component. Each builder clones it itself.
2. **Reproducibility.** `ref: main` meant every build picked up whatever was on
   the default branch at that moment — two devices flashed an hour apart could
   run different filter code with no way to tell from the version string.

`refresh:` is kept short (1 min for the proxy fork, `always` for the watchdog)
despite the pin, because the cached clone under `.esphome/external_components/`
is keyed **per build host** and a stale one has already served a build the wrong
code. A tag is immutable, so re-checking costs nothing and closes that hole.

If a build refuses to pick up a new tag, delete
`.esphome/external_components/<hash>` — `refresh:` does not always suffice.

### sdkconfig philosophy

- BLE nodes: coexistence **on**, `PREFER_WIFI` on the rrn00/s2224 families.
  (Light nodes elsewhere in the fleet compile BT out entirely and turn coex off.)
- Prefer the first-class `esp32:` / `framework.advanced:` keys —
  `watchdog_timeout`, `cpu_frequency`, `assertion_level`,
  `enable_lwip_mdns_queries`, `compiler_optimization` — over raw
  `sdkconfig_options` whenever one exists.
- Don't restate ESPHome's own defaults.
- Verify any change by comparing the RAM/flash byte counts in the compile output.

### Comments are the documentation

The packages carry long rationale comments, including incident history — which
threshold starved which radio, which OTA caused which reboot loop. That history
is the most valuable thing in the repo. **Preserve it and add to it.** Don't
strip comments to tidy a file.
