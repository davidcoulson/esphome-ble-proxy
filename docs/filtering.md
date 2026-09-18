# Filtering

Core ESPHome's `bluetooth_proxy` forwards **every** advertisement it hears. It
has no RSSI, MAC or name filtering of any kind, and the `mac_address` /
`service_uuid` / `manufacturer_id` filters in `esp32_ble_tracker` do **not**
gate the proxy stream — they only apply to `on_ble_advertise` automation
triggers.

So this fleet runs a fork:
[`davidcoulson/esphome-bluetooth-proxy-filter`](https://github.com/davidcoulson/esphome-bluetooth-proxy-filter),
pinned by tag in `common/ble-proxy.yaml` (**v1.5.0**, synced to ESPHome
2026.9.0). It adds filtering inside `on_raw_advertisement_()` — the only point
at which a packet can be suppressed before it is queued for the API and crosses
the network.

With everything at its default the fork is **byte-for-byte equivalent to
upstream**: every option defaults to off or −127 ("forward everything").

> **`api: batch_delay` does nothing for this traffic.** The advertisement flush
> calls `send_message()` directly and bypasses the deferred batch queue that
> `batch_delay` governs. Flush cadence is hardcoded — a 16-advertisement batch,
> or the proxy's own ~200 ms WiFi / 100 ms Ethernet loop gate. `batch_delay`
> only affects sensor state.

---

## The filter chain, in execution order

This is the order the C++ actually runs. It matters: an earlier test wins, and
the cheap tests are deliberately first so expensive ones (AES for IRK
resolution, payload walks) run on as little traffic as possible.

| # | Stage | What it does |
|---|---|---|
| 1 | `mac_blocklist` | Dropped outright, ahead of every allow rule. "This proxy does not handle this device" must not be overridable by a broader allowlist. |
| 2 | **pre-gate** | A single computed RSSI floor — the loosest limit any rule could apply. Cheapest test there is; keeps the categoriser below from running on traffic no category would have kept. |
| 3 | **categorise** | First hit wins: `mac_allowlist` → IRK/RPA → iBeacon → service UUID. Everything else is `DEFAULT`. A match sets *protected*, which exempts it from stages 6 and 7. |
| 4 | **per-category RSSI** | Each category measured against its own limit (see below). |
| 5 | `allowlist_exclusive` | If on, the allowlist stops being a bypass and becomes the *only* way through. Off here. |
| 6 | `drop_non_resolvable` | Non-resolvable private addresses rotate and carry no identity — not even an IRK can resolve them. Skipped for protected advertisements. |
| 7 | `manufacturer_blocklist`, `name_blocklist` | Payload walk, last, because by here most traffic has already gone. Skipped for protected advertisements. |

Two consequences worth internalising:

**Anything that matched a category is protected from the payload filters.** Your
phone advertises Apple manufacturer data and `0x004C` is blocklisted — but the
IRK match at stage 3 marks it protected, so stage 7 never touches it. Same for
allowlisted tags, matched iBeacons and pairing devices.

**An unresolved RPA is dropped regardless of RSSI.** Proximity does not make an
unidentifiable device identifiable. This is where most of the 56–66% goes.

---

## Every option

Defaults are the fork's, not this fleet's. What this fleet sets is in the last
column — read `common/ble-proxy.yaml` for the reasoning, it is heavily commented.

### Distance

| Option | Default | Here | |
|---|---|---|---|
| `rssi_threshold` | −127 | −80 (compile) / **−75 (runtime)** | The `DEFAULT` category's limit. Compile-time value is a fallback only — the **BLE RSSI Threshold** number entity restores on boot and overwrites it. |
| `rssi_floor` | −127 | **−90** | Absolute reception limit applied to every category that did not bring its own. Below this an RSSI carries no usable distance information; forwarding it does not help a tracker triangulate, it misleads it. |
| `rssi_mac_allowlist` | −127 | unset → bounded only by the floor | Tracked tags are exempt from `rssi_threshold` — a weak reading is exactly what places a tag nearer another proxy. |
| `rssi_irk` | −127 | unset → inherits `rssi_threshold` | |
| `rssi_service_uuid` | −127 | **−90** | Bound by the floor rather than the threshold, so a Tile one room away is not dropped. |

Unset (−127) means **inherit**, and what it inherits from is chosen so that an
upgrade which sets nothing reproduces the old single-threshold behaviour exactly.

### Identity

| Option | Default | Here | |
|---|---|---|---|
| `irks` | `[]` | 8 keys | Identity Resolving Keys for your own phones/watches. An RPA resolving to none of them belongs to someone else. **This is the biggest single win.** |
| `allow_espressif` | `true` | `true` | Exempts Espressif-OUI addresses from the IRK test (not from RSSI). Near-inert in practice — ESPHome advertises on the public MAC and a public address is never an RPA. |
| `drop_non_resolvable` | `false` | **`true`** | ~25 of these here, mostly Tiles. Every rotation looks like a new device to Home Assistant. Your own Tiles are rescued by the `0xFEED`/`0xFEEC` service-UUID entries. |
| `mac_allowlist` | `[]` | `!secret ble_pet_tag_macs` | Bypasses every filter except the floor. |
| `mac_blocklist` | `[]` | unset | Stage 1 — wins over everything. |
| `allowlist_exclusive` | `false` | `false` | Turn on for a proxy that should *only* ever forward known devices. |

### Payload

| Option | Default | Here | |
|---|---|---|---|
| `manufacturer_blocklist` | `[]` | **`[0x004C]`** | Apple is ~69% of everything still reaching the network. AirPods, AirTags, HomePods, neighbours. |
| `name_blocklist` | `[]` | unset | Case-insensitive substring match on the local name. |
| `allow_homekit` | `true` | `true` | HomeKit accessories advertise under Apple's company id with subtype `0x06`, so the blocklist above would silently kill every one — and unlike phones they have no IRK to rescue them. |
| `allow_findmy` | `false` | **`{rssi: -85}`** | AirTags and licensed third-party tags: Apple company id, subtype `0x12`, random static address. There is **no per-accessory scoping** — the advertisement carries nothing a proxy could match on — so every FindMy tag in range comes through. −85 bounds that below the fleet threshold rather than the floor. |
| `allow_ibeacon` | `false` | **`[{major: 1, rssi: -95}]`** | iBeacon is Apple subtype `0x02`, so `0x004C` drops every one — and `allow_homekit` does not rescue them. Scoped to major 1 so only *our* beacons are exempt. |
| `service_uuid_allowlist` | `[]` | 5 entries | See below. |

### `service_uuid_allowlist` — the pairing escape hatch

Service UUIDs that bypass every filter including the address-type tests. Accepts
16-bit shorts and full 128-bit UUIDs (transmitted little-endian, reversed before
comparison; a SIG short advertised in Base-UUID long form still matches a 16-bit
entry).

```yaml
service_uuid_allowlist:
  - 0xFFF6                                   # Matter commissioning
  - "00467768-6228-2272-4663-277478268000"   # Improv Wi-Fi
  - 0xFE59                                   # Nordic DFU
  - 0xFEED                                   # Tile
  - 0xFEEC                                   # Tile, unclaimed
```

Each is there for a *different* reason:

- **Matter `0xFFF6`** — a device in commissioning mode advertises from a
  rotating private address, so `drop_non_resolvable` and the unresolved-RPA test
  would discard it. Its MAC is not knowable in advance, so `mac_allowlist`
  cannot help. The service UUID is the only handle that exists.
- **Improv** — advertises from its *public* Espressif address, so the
  address-type tests never touch it. Its only exposure is the RSSI threshold,
  which drops a far-away board. Note `allow_espressif` does **not** rescue it:
  that flag is consulted only in the unresolved-RPA test, never in the RSSI check.
- **Nordic DFU `0xFE59`** — an nRF52 tag in bootloader mode advertises from a
  *different* MAC (commonly base+1), so its `mac_allowlist` entry does not cover
  it. Without this the tag cannot be updated over the air.
- **Tile `0xFEED` / `0xFEEC`** — Tiles advertise from rotating non-resolvable
  addresses, so `drop_non_resolvable` takes them and no IRK or MAC can help.

**Resist adding high-volume public services** — `0xFD6F` exposure notification,
`0xFE2C` Fast Pair. Those devices are not being paired to Home Assistant, and
letting them bypass everything would undo the point of the file.

Verify it fired: the **BLE Service UUID Allowed** sensor counts advertisements
forwarded *only* because of this list. It sits at zero while nothing is pairing,
so any movement is direct evidence the passthrough did the work — which makes a
failed commissioning attempt diagnosable instead of guesswork.

---

## Getting your IRKs

An IRK is a 32-character hex string, one per device. Easiest sources, in order:

**1. Home Assistant already has them.** Settings → Devices & Services → **Private
BLE Device**. Any IRK you entered there is the same value the filter wants. Keep
the two lists mirrored.

**2. iOS / macOS.** Pair the phone to an ESP32 running
[`irk-capture`](https://github.com/DerekSeaman/irk-capture); it publishes the
IRK as an entity. `config/common/irk-capture-base.yaml` in this repo is a
ready-made package for that — flash it to a **spare ESP32 / S2 / S3**. Not a
C3/C6/H2: that component needs the Bluedroid stack, and those chips are
NimBLE-only.

**3. Android.** Root, then read `/data/misc/bluedroid/bt_config.conf`.

Then in `secrets.yaml`:

```yaml
ble_irk_phone_1: "00112233445566778899aabbccddeeff"
```

Missing one is harmless — an empty `irks:` list simply disables IRK filtering.
A *wrong* one is also harmless; it just never matches.

---

## Measuring it

Five diagnostic sensors per proxy, all 60-second deltas of free-running `uint32`
counters (unsigned subtraction stays correct across the rollover, so nothing
needs resetting):

| Sensor | Reads |
|---|---|
| **BLE Adverts Forwarded** | adv/min that crossed the network |
| **BLE Adverts Dropped** | adv/min suppressed on-device |
| **BLE RPAs Dropped** | subset of Dropped: other people's phones and watches |
| **BLE Service UUID Allowed** | forwarded *only* because of the UUID allowlist |
| **BLE Advert Drop Rate** | % suppressed — **this is the tuning number** |

Drop Rate returns `NAN` (unknown), not `0`, when the proxy heard nothing at all,
so an idle proxy is not misreported as a 0% drop rate.

A healthy proxy in a normal house sits around **50–70%**. Much lower usually
means the IRKs are missing. Much higher, with devices going missing, means the
threshold is too tight — see [tuning.md](tuning.md).
