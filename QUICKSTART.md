# Quickstart — a filtering BLE proxy in ten minutes

Six steps. Nothing here needs you to understand the filtering; the defaults are
tuned and you can leave them alone. When you want to change something, every
option is explained in [docs/filtering.md](docs/filtering.md) and
[docs/tuning.md](docs/tuning.md).

**Don't want to do any of this?** [Quick flash](https://davidcoulson.github.io/esphome-ble-proxy/)
installs prebuilt firmware straight from your browser and sets up Wi-Fi over
Improv — no ESPHome, no YAML, no secrets file. It asks two questions (block
Apple? relay connections?) and flashes the matching build. What it can't do is
Apple filtering *and* phone tracking at once — that needs your IRKs, which are
compile-time — but the device is adoptable afterwards so you can add them.

**Halfway house:** the [config generator](https://davidcoulson.github.io/esphome-ble-proxy/#advanced)
asks you six questions and writes steps 2 and 3 for you.

---

## What you need

- An ESP32 with Bluetooth. Any of these work; they are listed best-first:
  **ESP32-S3 + W5500 Ethernet** → **ESP32-C5 on 5 GHz WiFi** → **ESP32-S3** →
  ESP32-C3 / C6 / ESP8685 / classic ESP32. See [docs/hardware.md](docs/hardware.md)
  for why the order matters — short version, WiFi and BLE share one radio on the
  cheap boards.
- ESPHome (the Home Assistant add-on, or the CLI). **2026.9.0 or newer** — the
  packages use keys that do not exist before it.
- A USB cable for the first flash. Every flash after that is over the air.

## 1. Get the files onto your ESPHome box

```bash
git clone https://github.com/davidcoulson/esphome-ble-proxy
cp -r esphome-ble-proxy/config/common /config/esphome/
cp esphome-ble-proxy/config/secrets.yaml.example /config/esphome/secrets.yaml
```

If `/config/esphome/common/` already exists, copy the files in rather than over
it — and check `docs/architecture.md` first, because the shared packages assume
a particular set of ids (`id_ota`, `ble_tracker`, `ble_proxy`) exist.

## 2. Fill in `secrets.yaml`

Open `/config/esphome/secrets.yaml`. The only two lines you *must* set are
`wifi_ssid` and `wifi_password` (skip both on an Ethernet board).

Everything else can wait:

- **`ble_irk_*`** — leave the placeholders or delete those lines entirely. An
  empty `irks:` list just turns IRK filtering off; the proxy works fine without
  it, it simply forwards other people's phones too. Come back and fill these in
  once you have something working — see [docs/filtering.md#getting-your-irks](docs/filtering.md#getting-your-irks).
- **`ble_pet_tag_macs`** — the MACs of tags you actually want to track. Leave
  the examples if you have none yet.
- **`api_key_ble_proxy`** — generate one with `openssl rand -base64 32`.

## 3. Write the device file

One file per proxy, in `/config/esphome/`. For a dedicated ESP32-S3:

```yaml
# /config/esphome/ble-esp32s3-a1b2c3.yaml
substitutions:
  mac: a1b2c3          # last 6 hex digits of the board's MAC
  area: "Kitchen"      # the Home Assistant area it lives in

packages:
  device: !include common/esp32s3-ble.yaml
```

That is the whole file. Swap the include for the family that matches your
board — `config/devices/` in this repo has a worked example for each:

| Your board | Include |
|---|---|
| ESP32-S3, WiFi | `common/esp32s3-ble.yaml` |
| ESP32-S3 + W5500 Ethernet | `common/esp32-eth-ble.yaml` |
| Classic ESP32 (incl. reflashed Shelly) | `common/shelly-ble.yaml` |
| ESP32-C3 / C6 in-wall outlet (RRN00) | `common/rrn00.yaml` + `module: 3｜6｜c6` |
| ESP32-C3 / ESP8685 outlet (s2224) | `common/s2224-esp32c3.yaml` + `module: xh-c3f｜esp8685` |
| CloudFree SW1 | `common/cf-sw1-esp8685.yaml` |
| ESP32-C5, XIAO C6, anything else | compose it — copy `config/devices/esp32c5-a1b2c3.yaml` |

Don't know the MAC yet? Put anything in, flash, and read the real one off the
ESPHome dashboard — then fix the file and re-flash over the air. `mac` is only
a naming convention; nothing depends on it matching.

## 4. Flash it once over USB

Plug the board in. In the ESPHome dashboard: **Install → Plug into this
computer**. Or from the CLI:

```bash
esphome run /config/esphome/ble-esp32s3-a1b2c3.yaml
```

First build takes a while — it compiles ESP-IDF. Later builds are minutes.

> **If it fails to build** on the version pin, see
> [docs/troubleshooting.md](docs/troubleshooting.md). The most common cause is
> an ESPHome older than 2026.9.0.

## 5. Adopt it in Home Assistant

It shows up on its own: **Settings → Devices & Services → ESPHome**, "Discovered".
Click **Configure** and paste the encryption key from your `secrets.yaml`.

Home Assistant now uses it as a Bluetooth adapter automatically. Nothing else to
configure — **Settings → Devices & Services → Bluetooth** will list it as a
remote adapter, and any BLE integration you already have can reach devices
through it.

## 6. Check it is actually working

On the new device in Home Assistant you get these:

| Entity | What it tells you |
|---|---|
| **BLE Adverts Forwarded** | advertisements/min reaching Home Assistant |
| **BLE Adverts Dropped** | advertisements/min killed on-device |
| **BLE Advert Drop Rate** | the tuning number — % suppressed |
| **BLE RPAs Dropped** | other people's phones and watches |
| **BLE RSSI Threshold** | the dial — raise it to filter harder |
| **BLE Scan Profile** | Insane / Aggressive / Balanced / Disabled |

Give it a minute (they publish every 60s). **Forwarded** above zero means it is
working. A **Drop Rate** somewhere in the 50–70% range is normal and healthy.

That's it. You have a working proxy.

---

## Then what

**Tune it.** Leave it a day, then look at the Drop Rate. `BLE RSSI Threshold`
is a live dial — no reflash — and defaults to −75 dBm, about 4.6 m of useful
range. Raise it toward −85 if devices you care about are going missing, lower it
if the drop rate is too low to matter. [docs/tuning.md](docs/tuning.md).

**Onboarding a new BLE device?** Scanning is passive, which means scan responses
are dropped — and many devices put their name *only* in the scan response, so
Home Assistant may never discover them. Flip the **BLE Active Scan (onboarding)**
switch on, add the device, flip it off. It resets to off on reboot by design.

**Add your IRKs.** This is the single biggest win — 56–66% of advertisements
dropped. [docs/filtering.md#getting-your-irks](docs/filtering.md#getting-your-irks).

**Room-level presence?** Install [Bermuda](https://github.com/agittins/bermuda).
You need at least three proxies for trilateration to mean anything.
[docs/home-assistant.md](docs/home-assistant.md).

**Adding more proxies?** From here it is: write a 5-line device file, flash over
USB once, adopt. Repeat. Do **not** mass-flash an existing fleet in one go —
[docs/operations.md](docs/operations.md) explains what happened the time that
was tried.
