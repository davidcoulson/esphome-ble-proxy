# Hardware

## What to buy

Ranked by how well they actually work as proxies, best first.

### 1. ESP32-S3 + W5500 Ethernet — the best option

No radio contention at all. Wired uplink, no WiFi component, no gateway
watchdog needed. Runs the **Aggressive** scan profile (94% duty) and carries 5
BLE connection slots instead of the usual 3.

Two common variants, and they are **not** pin-compatible:

| | Waveshare ESP32-S3-ETH | XIAO ESP32-S3 + W5500 |
|---|---|---|
| CLK | GPIO13 | GPIO7 |
| MOSI | GPIO11 | GPIO9 |
| MISO | GPIO12 | GPIO8 |
| CS | GPIO14 | GPIO2 |
| INT | GPIO10 | GPIO10 |
| RESET | GPIO9 | *none* — `!remove` it |

`common/eth.yaml` defaults to the **Waveshare** pinout. On a XIAO, override in
the device file:

```yaml
ethernet:
  clk_pin: GPIO7
  mosi_pin: GPIO9
  miso_pin: GPIO8
  cs_pin: GPIO2
  interrupt_pin: GPIO10
  reset_pin: !remove
```

Remember these nodes set `api.reboot_timeout: 300s` — it is their only automatic
recovery. See [architecture.md](architecture.md#reboot-policy).

### 2. ESP32-C5 on 5 GHz WiFi

The only board here with a 5 GHz radio. Putting WiFi on 5 GHz takes it off the
band BLE scans, which removes the contention that forces every other WiFi proxy
down to 37.5%. A C5 on 5 GHz can run **Aggressive**.

```yaml
wifi:
  band_mode: 5GHZ
```

Note `CONFIG_SOC_RMT_SUPPORTED` must stay **enabled** if the board drives a
WS2812 status LED (the C5 dev boards do) — unlike the rrn00/s2224 families,
which disable RMT to save space.

### 3. ESP32-S3, WiFi

Dual core, so BLE and WiFi are not fighting for one CPU even though they share
the antenna. A dedicated S3 doing nothing but proxying is a good, boring proxy.

### 4. Classic ESP32 (including reflashed Shellys)

Works fine. Configured unicore at 160 MHz here — `cpu_frequency: 160MHz` as the
first-class key, which replaced a `platformio_options` `board_build.f_cpu` that
had become a no-op under the ESP-IDF toolchain. Verified byte-identical binary
size before and after, so that was a spelling change, not a clock change.

**Caution:** the original ESP32 supports BLE 4.2 only. `ble-proxy.yaml` sets
`CONFIG_BT_BLE_50_FEATURES_SUPPORTED: y`, which is correct for derivatives
(S3/C3/C5/C6) but not the original. It is harmless on the classic ESP32 — the
symbol is simply not honoured — but don't read it as "this board does BLE 5".

### 5. ESP32-C3 / C6 / ESP8685 — workable, with a caveat

Cheap, tiny, and what most in-wall devices actually contain, so in practice a
lot of the fleet is these. **Single core, one radio shared between WiFi and
BLE.** They must run the **Balanced** (37.5%) profile. At 75% they starved WiFi
badly enough to break ping and OTA.

If one of these will not accept an OTA, that is the reason — see
[operations.md](operations.md#a-node-will-not-take-an-ota).

The C6 is WiFi 6 capable; `CONFIG_ESP_WIFI_11AX_SUPPORT: y` shortens airtime per
frame, which genuinely matters on a shared radio.

### Not usable for IRK capture: C3 / C6 / H2

`irk-capture-base.yaml` needs the **Bluedroid** stack. Those chips are
NimBLE-only. Use a classic ESP32, S2 or S3 for that job.

---

## Placement

Proxies are only useful where they hear things. Some rules of thumb from this
fleet:

- **Three minimum** for Bermuda to trilaterate meaningfully. More helps more
  than moving existing ones.
- **One per room** is the realistic target for room-level presence. This is why
  so much of the fleet is in-wall outlets — they are already everywhere, already
  powered, and nobody has to look at them.
- **Not in metal, not in a cupboard.** An outlier low **BLE Adverts Forwarded**
  reading almost always means placement, not a fault.
- Tile-style tags that rotate their address need **≥2 proxies** hearing both the
  old and new address for Bermuda to bind them across the rotation.

## Existing fleet

| Family | Count | Board | Transport |
|---|---:|---|---|
| `rrn00` | 38 | ESP32-C3 WROOM-03/06, ESP32-C6 WT0132C6 | WiFi |
| `s2224` | 11 | ESP32-C3 XH-C3F, ESP8685 | WiFi |
| `shelly` | 6 | ESP32 classic | WiFi |
| `esp32-eth` | 5 | ESP32-S3 + W5500 | Ethernet |
| `esp32c5` | 4 | ESP32-C5 | WiFi 5 GHz |
| `esp32s3` | 3 | ESP32-S3 | WiFi |
| `cf-sw1` | 2 | ESP8685 (CloudFree SW1) | WiFi |
| `xiao` | 1 | XIAO ESP32-C6 | WiFi |

## Pinouts in the family packages

`rrn00.yaml` and `s2224-esp32c3.yaml` carry pin maps selected by the `module:`
substitution, so one package serves several board revisions:

**rrn00** — `module: 3` (WROOM-03), `6` (WROOM-06), `c6` (WT0132C6)

| | `3` | `6` | `c6` |
|---|---|---|---|
| upper relay | GPIO3 | GPIO10 | GPIO7 |
| lower relay | GPIO5 | GPIO7 | GPIO4 |
| upper button | GPIO6 | GPIO6 | GPIO10 |
| lower button | GPIO1 | GPIO3 | GPIO2 |
| upper LED | GPIO7 | GPIO1 | GPIO8 |
| lower LED | GPIO4 | GPIO19 | GPIO6 |
| status LED | GPIO2 | GPIO2 | GPIO5 |

GPIO2 on the WROOM-03/06 modules is a strapping pin and the board wires the
status LED there, so the validator's warning is expected noise —
`ignore_strapping_warning: true` is set for exactly that.

**s2224** — `module: xh-c3f` or `esp8685`

| | `xh-c3f` | `esp8685` |
|---|---|---|
| upper relay | GPIO19 | GPIO3 |
| lower relay | GPIO18 | GPIO10 |
| upper button | GPIO4 | GPIO5 |
| status LED | GPIO5 | GPIO6 |

**cf-sw1** (ESP8685) — relay GPIO2, button GPIO4, status LED GPIO5.
