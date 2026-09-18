# Tuning

Two dials, both live in Home Assistant, neither needs a reflash:
**BLE RSSI Threshold** and **BLE Scan Profile**.

---

## The constraint everything else follows from

**BLE and WiFi share one 2.4 GHz radio** on the single-core C3 / C6 / ESP8685
boards. The BLE scan duty cycle is airtime taken directly away from WiFi.

At a 75% duty cycle it starved WiFi badly enough to **break ping and OTA** —
while UniFi still reported a perfect link. That is the trap: the access point
sees an associated, healthy client, because association is a link-layer state
and the radio is still doing that part fine. It is the data path that is gone.

Hence 37.5% as the fleet default for WiFi proxies, and hence the hardware
preference order:

| | Radio contention | Profile it can run |
|---|---|---|
| **Ethernet (S3 + W5500)** | none | Aggressive (94%) |
| **ESP32-C5 on 5 GHz** | none — WiFi is off BLE's band | Aggressive |
| ESP32-S3, classic ESP32 | dual-core, some | Balanced, Aggressive with care |
| C3 / C6 / ESP8685 | **single core, one radio** | Balanced (37.5%) |

## Scan profiles

Set per-proxy from the **BLE Scan Profile** select. Restores across reboot.

| Profile | Interval | Window | Duty | Use on |
|---|---|---|---|---|
| **Insane** | 1100 ms | 1100 ms | 100% | Diagnostics only. Nothing else gets the radio. |
| **Aggressive** | 320 ms | 300 ms | 94% | Ethernet nodes, C5 on 5 GHz |
| **Balanced** | 320 ms | 120 ms | 37.5% | **Every WiFi proxy.** The default. |
| **Disabled** | — | — | — | Actually stops the radio |

37.5% is ample for Bermuda: it polls every 10 s with 20-sample smoothing, and
tracked tags advertise several times a second. Going faster buys resolution you
cannot use and costs WiFi you need.

> **"Disabled" really disables.** An earlier version only set passive mode and a
> 30 ms window, which still scanned and still forwarded. It now calls
> `stop_scan` properly.

> **The lambda is in 0.625 ms controller units, not milliseconds.** The
> `scan_parameters:` block in YAML is in ms and gets converted at codegen; the
> C++ setters in the select's lambda do **not**. `ms / 0.625 = units`. All three
> profiles were originally written as if they were ms and have been corrected —
> keep the inline `// ms` comments if you edit them.

## Scanning is passive, on purpose

`scan_parameters: active: false`.

Active scanning transmits a scan *request* per advertiser to pull a scan
*response*. That costs TX airtime on the shared radio **and** inflates every
forwarded packet, because the hub merges the scan response into the advertisement
payload. Bermuda only needs RSSI, so the scan response buys nothing.

**The trade-off:** local names and anything carried *only* in a scan response are
lost. Many BLE devices put their name only there, and Home Assistant discovers
new BLE devices by name — so a brand-new device may never be discoverable at all.
Already-known devices keep working.

That is what the **BLE Active Scan (onboarding)** switch is for. Flip it on, add
the device in Home Assistant, flip it off. It is deliberately *not*
`restore_value` — a reboot returns to passive, so leaving it on by accident
cannot quietly cost airtime for weeks.

## The RSSI threshold

**BLE RSSI Threshold** is a number entity, −90 to −40 dBm, default −75. It calls
`set_rssi_threshold()` live.

With Bermuda's `ref_power: -55` and `attenuation: 3`, the mapping is
`distance = 10 ^ ((-55 - rssi) / 30)`:

| Threshold | ≈ distance |
|---|---|
| −75 dBm | 4.6 m |
| −80 dBm | 6.8 m |
| −85 dBm | 10 m |
| −90 dBm | 14 m |
| −94 dBm | 20 m |

**Which way to move it:**

- Devices going missing, or Bermuda's area assignments flapping? **Lower** it
  (−75 → −85). You are cutting off readings Bermuda needs.
- Drop rate under ~40% and network traffic still heavy? **Raise** it. But check
  the IRKs first — a missing IRK list is almost always the real cause, and it
  costs nothing in range to fix.

The minimum is pinned to `rssi_floor` (−90) deliberately. The floor discards
everything below −90 anyway, so a setting under it would be **dead travel on the
dial** — it would appear to loosen the filter and change nothing. If you raise
`rssi_floor`, raise this `min_value` to match or the dial silently stops working
below the new floor.

−127 ("forward everything", the old filter-off setting) is deliberately not
reachable. To disable filtering for a comparison, raise `rssi_floor` and lower
`min_value` together and reflash, or use the **Disabled** scan profile to stop
the radio entirely.

> **Why `on_value`, not `set_action`:** `TemplateNumber::setup()` restores a
> saved value by calling `publish_state()` and never `control()`, so
> `set_action` would not fire on boot — the restored value would show in the UI
> while the proxy silently kept the YAML default. `on_value` fires on both
> boot-restore and user changes. Ordering is safe: ESPHome emits all `cg.add()`
> configuration into `main.cpp` before `App.setup()`, so the YAML default is
> applied first and the restored number then overwrites it.

## Tuning loop

1. Leave the fleet a day at defaults.
2. Compare **BLE Advert Drop Rate** across proxies. An outlier is usually
   placement (a proxy in a cupboard hears nothing) or a dead radio.
3. Watch Bermuda's `sensor.*_area` entities for the devices you actually care
   about. Flapping between areas = threshold too tight, or too few proxies.
4. Change one thing. Wait an hour. The sensors are 60-second deltas and Bermuda
   smooths over 20 samples; nothing here responds instantly.

**Do not** tune `api: batch_delay` expecting it to help. It does nothing for
advertisement traffic — see [filtering.md](filtering.md).
