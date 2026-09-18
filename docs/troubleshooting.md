# Troubleshooting

## Build and flash

**"The selected framework version is not the recommended one"**
Expected. ESP-IDF is pinned to **6.1.0** in `djc-common.yaml` while 2026.9.0's
"recommended" is 5.5.5. The Bootloader Version sensor uses the IDF 6 signature
of `esp_ota_get_bootloader_description()` and will not build on 5.x. Ignore it.

**Build fails on an unknown key** (`watchdog_timeout`, `assertion_level`,
`cpu_frequency`, `enable_lwip_mdns_queries`)
Your ESPHome is older than 2026.9.0. `djc-common.yaml` sets
`min_version: 2026.9.0` precisely so this fails fast rather than producing
strange firmware.

**Build picks up the wrong version of the filter component**
The per-build-host clone under `.esphome/external_components/` is stale.
`refresh:` does not always suffice — delete `.esphome/external_components/<hash>`
and rebuild. This has genuinely served a build the wrong code before.

**`bluetooth_proxy: Unknown key allow_findmy`** (or any newer filter option)
The build resolved the *stock* component, or an older fork tag. Check that
`ble-proxy.yaml`'s `external_components` block is intact and that the `ref:` tag
actually contains the option. If you switched to `type: local`, the local
fallback copy may lag the pinned tag.

**Strapping-pin warning on GPIO2** (rrn00 WROOM-03/06)
Expected — the board wires the status LED to a strapping pin.
`ignore_strapping_warning: true` is already set.

**The node will not accept an OTA at all**
Almost always a C3/C6/ESP8685 with a BLE-saturated radio. The
`ota: on_begin: stop_scan` hook cannot help — it only fires once the OTA has
begun. **Boot to safe mode and flash there.**
See [operations.md](operations.md#a-node-will-not-take-an-ota).

---

## The proxy runs but nothing works

**BLE Adverts Forwarded is 0**

1. Is **BLE Scan Profile** set to `Disabled`? That genuinely stops the radio.
2. Is Home Assistant connected? Scanning **only runs while an API client is
   connected** — `on_client_connected` starts it, `on_client_disconnected` stops
   it. This is deliberate: a proxy with nobody listening should not burn airtime.
3. Is the threshold absurd? Check **BLE RSSI Threshold**.
4. Placement — a proxy in a metal box hears nothing. Compare against its
   siblings.

**Drop Rate shows "unknown" / unavailable**
The proxy heard *nothing at all* in the interval, so the sensor returns `NAN`
rather than `0`. That is by design: an idle proxy reporting "0% dropped" would
be a lie. See the point above.

**The proxy appears in Home Assistant but no BLE devices do**
Check Settings → Devices & Services → **Bluetooth**. The proxy should be listed
as a remote adapter. If it isn't, the ESPHome device was adopted but Home
Assistant hasn't picked up the Bluetooth capability — reload the ESPHome
integration entry.

---

## Devices are missing or flapping

**A known device disappeared after enabling filtering**
Work down the filter chain — the order matters, see
[filtering.md](filtering.md#the-filter-chain-in-execution-order):

| Symptom | Likely filter | Fix |
|---|---|---|
| An iPhone/Watch of yours | `irks` missing or wrong | Add the IRK; unresolved RPAs are dropped **regardless of RSSI** |
| An AirTag / FindMy tag | `manufacturer_blocklist: 0x004C` | `allow_findmy: {rssi: -85}` |
| A HomeKit accessory | same | `allow_homekit: true` (default) |
| An iBeacon | same — `allow_homekit` does **not** cover it | add `allow_ibeacon` with its major |
| A Tile | `drop_non_resolvable` | `service_uuid_allowlist: [0xFEED, 0xFEEC]` |
| Something in pairing mode | rotating private address | `service_uuid_allowlist` — its MAC is unknowable in advance |
| A tracked tag, sometimes | RSSI floor | it's on `mac_allowlist` but still bounded by `rssi_floor: -90` |
| Anything, when far away | `rssi_threshold` | lower it, or add more proxies |

**A brand-new BLE device won't be discovered at all**
Scanning is **passive**, so scan responses are dropped — and many devices put
their local name *only* in the scan response, which is how Home Assistant
discovers them. Flip the **BLE Active Scan (onboarding)** switch on, add the
device, flip it off. It resets to off on reboot by design.

**Bermuda areas flapping between rooms**
Usually the threshold is too tight (readings Bermuda needs are being cut off) or
there are too few proxies. Lower **BLE RSSI Threshold** toward −85 and check you
have at least three proxies hearing the device. Bermuda smooths over 20 samples
polled every 10 s, so give any change several minutes.

**A rotating-address tag (Tile) loses its identity**
Bermuda binds these across rotations by matching the per-proxy RSSI pattern of
the old and new address — which needs **≥2 proxies** hearing both.

**An nRF52 tag can't be updated over the air**
In bootloader mode it advertises from a *different* MAC (commonly base+1), so
its `mac_allowlist` entry doesn't cover it. That is what `0xFE59` (Nordic DFU)
in `service_uuid_allowlist` is for.

---

## Network and stability

**Node reboots every ~5 minutes during a Home Assistant outage**
Expected on **Ethernet** nodes: they set `api.reboot_timeout: 300s` because they
have no WiFi timeout and no gateway watchdog, so it is their only automatic
recovery. Intentional — don't "fix" it. WiFi nodes are at `0s` and are unaffected.

**Gateway Watchdog Reboots Used is climbing**
The node has link but cannot reach its gateway. Check the VLAN, the DHCP-supplied
gateway and the uplink. The cap is 2 reboots per 1 h budget, so it will stop
rather than loop — a node sitting at 100% packet loss and staying up is the
designed outcome.

**Many nodes rebooting at once after an OTA**
This is the incident the `max_reboots: 2` cap exists for. Stop the rollout. See
[operations.md](operations.md#never-mass-flash).

**WiFi looks perfect in UniFi but the node is unreachable**
Classic BLE/WiFi contention on a single-radio board. Association is a link-layer
state the radio still manages fine; the data path is what's gone. Drop the scan
profile to **Balanced**, or move that node to Ethernet.

**Pings fine, but OTA and logs are slow or drop**
Same cause. Check the scan profile hasn't been left on **Aggressive** or
**Insane** on a WiFi board.

---

## Diagnostics worth reaching for

| Question | Look at |
|---|---|
| Is it hearing anything? | BLE Adverts Forwarded |
| Is the filter too tight? | BLE Advert Drop Rate, BLE RPAs Dropped |
| Did the pairing passthrough fire? | BLE Service UUID Allowed — sits at 0 unless it did |
| Why did it reboot? | Reset Reason (`debug-lite.yaml`) |
| Which firmware is it on? | Project Version — includes `+ble<version>` |
| Is the network the problem? | Gateway Packet Loss, Gateway Watchdog Reboots Used |
| Has it been up? | Uptime (adaptive: 10 s for the first 5 min after boot) |
