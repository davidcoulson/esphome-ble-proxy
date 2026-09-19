# Upgrading ESPHome

> **The rule: re-sync the fork on every ESPHome version bump.**
> `bluetooth_proxy` is a copy of a fast-moving core component. Skip this and
> devices build against stale proxy code that may no longer match the API
> protocol the rest of ESPHome is speaking.

## The runbook

### 1. Diff the fork against the new upstream

```bash
V=2026.10.0   # the new ESPHome version
for f in __init__.py bluetooth_proxy.cpp bluetooth_proxy.h; do
  curl -sL "https://raw.githubusercontent.com/esphome/esphome/$V/esphome/components/bluetooth_proxy/$f" \
    -o "upstream-$f"
done
```

Diff each against the fork's copy. The fork is upstream **plus** additions —
it modifies and deletes nothing — so a clean re-sync is: take the new upstream
file, re-apply the additions.

The additions, by file:

| File | Added |
|---|---|
| `__init__.py` | the filter config keys (`rssi_threshold`, `rssi_floor`, `rssi_mac_allowlist`, `rssi_irk`, `rssi_service_uuid`, `irks`, `mac_allowlist`, `mac_blocklist`, `allowlist_exclusive`, `manufacturer_blocklist`, `name_blocklist`, `drop_non_resolvable`, `allow_espressif`, `allow_homekit`, `allow_ibeacon`, `allow_findmy`, `service_uuid_allowlist`) in the schema and their `cg.add()` calls in **both** `to_code` arms |
| `bluetooth_proxy.h` | the members, setters and the counter getters |
| `bluetooth_proxy.cpp` | the filter chain in `on_raw_advertisement_()` plus its helpers (`address_is_rpa_`, `address_is_non_resolvable_`, `irk_matches_`, `set_irks`, `recompute_gate_`, `add_ibeacon_rule`, `ibeacon_match_`, `findmy_match_`, `uuid128_matches_`, `payload_has_allowed_service_uuid_`, `payload_blocked_`, `is_espressif_oui_`). After re-applying, run `tests/run.sh` in the fork — it checks all of these against the tested hook component. |

### 2. Decide whether the compiled code actually changed

This is the step that saves ~70 pointless OTAs.

- **C++ unchanged?** Tag the fork, bump `ref:` in `ble-proxy.yaml`, and **do
  not** bump `ble_proxy_version`. Between ESPHome 2026.8.2 and 2026.9.0 upstream
  moved exactly one Python type annotation on `validate_connections()` and
  changed no C++ at all, so fork v1.4.0 compiles to byte-identical output to
  v1.3.2 — bumping the version would have forced ~70 identical flashes.
- **C++ changed?** Tag it, bump `ref:`, **and** bump
  `substitutions.ble_proxy_version` in `ble-proxy.yaml`, so
  `sensor.*_project_version` tells you which nodes still need the new build.

### 3. Update the pin

```yaml
# common/ble-proxy.yaml
external_components:
  - source:
      type: git
      url: https://github.com/davidcoulson/esphome-bluetooth-proxy-filter
      ref: v1.7.0       # <- here
    components: [bluetooth_proxy]
    refresh: 1min
```

Never `ref: main`. That meant two devices flashed an hour apart could run
different filter code with no way to tell from the version string.

### 4. Check the other pins too

| Pin | Where | Watch for |
|---|---|---|
| ESP-IDF **6.1.0** | `djc-common.yaml` | The Bootloader Version sensor uses the IDF 6 signature of `esp_ota_get_bootloader_description(nullptr, &desc)` and will not build on 5.x. The "not the recommended version" warning is expected. |
| `min_version: 2026.9.0` | `djc-common.yaml` | Raise it when you adopt keys that need a newer ESPHome — it is what makes an old builder fail fast. |
| `gateway_watchdog@v1.0.0` | `gateway-watchdog.yaml` | Pinned deliberately; this component has already caused one fleet-wide reboot loop. |
| LibreTiny (non-proxy families) | `base-bk7231.yaml` | Not used by any proxy. |

### 5. Validate, then compile, then canary

```
validate  one device per affected family
compile   same — a validation pass does NOT compile lambdas, and half the
          interesting code here is lambdas
install   ONE canary device, watch it for ten minutes
```

Then roll by family, a handful at a time. **Never mass-flash** — see
[operations.md](operations.md#never-mass-flash).

### 6. Clear stale caches if a build misbehaves

Each build host keeps its own `.esphome/external_components/` clone, and a stale
one has served a build the wrong code before. `refresh:` does not always
suffice:

```bash
rm -rf /config/esphome/.esphome/external_components/<hash>
```

---

## Watch list for future ESPHome releases

**ESPHome 2026.10 is where the fork goes away.** Upstream merged an
`AdvertisementFilter` hook into stock `bluetooth_proxy`
([esphome/esphome#19220](https://github.com/esphome/esphome/pull/19220)), and
[`esphome-ble-advert-filter`](https://github.com/davidcoulson/esphome-ble-advert-filter)
is the same filter built on it — an ordinary external component that replaces
nothing, so there is nothing left to re-sync.

The two are kept interchangeable on purpose:

- Same tags (`v1.7.0` of one is `v1.7.0` of the other), same YAML keys, same
  lambda API (`set_irks()`, `get_adv_forwarded()`, …).
- The fork's CI runs `tools/check_parity.py`, which fails if any filter function,
  the chain itself or the public API differs textually from the hook component.
- The hook component's host tests compile the real source under ASan/UBSan. The
  fork cannot be compiled off-device, so parity is what carries those tests over.

The switch itself is three lines in `common/ble-proxy.yaml` — move the filter
keys from `bluetooth_proxy:` to `ble_advert_filter:`, give that the `ble_proxy`
id the lambdas already use, and point `external_components` at the new repo.
The hook repo's README has the exact diff under *Migrating from the
bluetooth_proxy fork*. All eight fleet families already validate that way on
2026.10.0-dev. It is a C++ change, so bump `ble_proxy_version` and roll it like
any other.

Until the fleet is on 2026.10, **any filter change lands in both repos** or the
parity check goes red.

Other things that would need work:

- **Any change to `on_raw_advertisement_()`'s signature or to
  `RawAdvertisement`.** The whole filter chain hangs off that one function.
- **Changes to the advertisement batching** (`BLUETOOTH_PROXY_ADVERTISEMENT_BATCH_SIZE`,
  the flush path). The counters live there.
- **`esp32_ble_tracker` scan setters.** The scan-profile lambda calls
  `set_scan_interval()` / `set_scan_window()` / `set_scan_active()` directly.
  Remember those take raw **0.625 ms controller units**, not milliseconds.
- **`TemplateNumber` restore semantics.** The RSSI threshold uses `on_value`
  rather than `set_action` specifically because `setup()` restores via
  `publish_state()` and never `control()`. If that changes, the restored value
  would silently stop being applied.

## Known non-proxy build failures

Carried from the 2026.9.0 fleet sweep, listed so they are not mistaken for
regressions introduced by a proxy change. None are built from `common/`:

| Device | Cause |
|---|---|
| `bk7321n-led` | Beken SDK header |
| `esphome-web-0e9154` | `stream_server` uses the removed `network::get_use_address` |
| `tigo-lilygo` | needs `esp_http_server` in `include_builtin_idf_components` |
| `litter-robot-4` | config error |

Everything built from `common/` validates and compiles.
