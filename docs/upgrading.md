# Upgrading ESPHome

> **The rule: read the release notes for `bluetooth_proxy` and
> `esp32_ble_tracker` changes on every ESPHome version bump.** The filter is
> [`esphome-ble-advert-filter`](https://github.com/davidcoulson/esphome-ble-advert-filter),
> an ordinary external component plugged into stock `bluetooth_proxy`'s filter
> hook (ESPHome 2026.10+). It replaces no core code, so there is nothing to
> re-sync, but it does depend on the hook and on the scan setters listed in the
> watch list below.

## The runbook

### 1. Check the filter still builds

Compile one proxy per family against the new ESPHome. A hook change shows up as
a build error in `ble_advert_filter`.

### 2. Decide whether the compiled code actually changed

This is the step that saves ~70 pointless OTAs.

- **Only the ESPHome version changed?** Bump `ESPHOME_VERSION` in
  `.github/workflows/pages.yml` and leave `ble_proxy_version` alone: the
  project version only tracks this package, and the ESPHome Version sensor
  already shows which nodes have the new release.
- **Filter or package changed?** Bump the filter `ref:` (if it moved) **and**
  `substitutions.ble_proxy_version` in `ble-proxy.yaml`, so
  `sensor.*_project_version` tells you which nodes still need the new build.

### 3. Update the pin

```yaml
# common/ble-proxy.yaml
external_components:
  - source:
      type: git
      url: https://github.com/davidcoulson/esphome-ble-advert-filter
      ref: v2.0.3       # <- here
    components: [ble_advert_filter]
    refresh: 1min
```

The same tag appears in `config/adopt/proxy-core.yaml` and as `FILTER_TAG` in
`wizard/index.html`. Never `ref: main`. That meant two devices flashed an hour
apart could run different filter code with no way to tell from the version
string.

### 4. Check the other pins too

| Pin | Where | Watch for |
|---|---|---|
| ESP-IDF **6.1.0** | `djc-common.yaml` | The Bootloader Version sensor uses the IDF 6 signature of `esp_ota_get_bootloader_description(nullptr, &desc)` and will not build on 5.x. The "not the recommended version" warning is expected. |
| `min_version: 2026.10.0` | `djc-common.yaml`, `adopt/proxy-core.yaml` | Raise it when you adopt keys that need a newer ESPHome — it is what makes an old builder fail fast. |
| `gateway_watchdog@v1.1.0` | `gateway-watchdog.yaml` | Pinned deliberately; this component has already caused one fleet-wide reboot loop. |
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

**The fleet left the fork at ESPHome 2026.10.** Upstream merged an
`AdvertisementFilter` hook into stock `bluetooth_proxy`
([esphome/esphome#19220](https://github.com/esphome/esphome/pull/19220)), and
`esphome-ble-advert-filter` is the same filter built on it. The filter options
moved from `bluetooth_proxy:` to `ble_advert_filter:`, which took the
`ble_proxy` id the lambdas use; stock `bluetooth_proxy` is now `ble_proxy_core`.

The old fork,
[`esphome-bluetooth-proxy-filter`](https://github.com/davidcoulson/esphome-bluetooth-proxy-filter),
stays available for ESPHome 2026.9 and earlier and gets filtering fixes only.
Its version numbers do **not** match this one's: fork v1.8.1 carries the same
filter as `ble_advert_filter` v2.0.3. Its CI runs `tools/check_parity.py`
against a pinned `ble_advert_filter` tag, so a filter fix lands in this
component first and is then ported to the fork.

Other things that would need work:

- **Any change to the `AdvertisementFilter` hook or to
  `RawAdvertisement`.** The whole filter chain hangs off that one callback.
- **Changes to the advertisement batching** (`BLUETOOTH_PROXY_ADVERTISEMENT_BATCH_SIZE`,
  the flush path). The counters live there.
- **`esp32_ble_tracker` scan setters.** The scan-profile lambda calls
  `set_scan_interval()` / `set_scan_window()` / `set_scan_active()` directly.
  Remember those take raw **0.625 ms controller units**, not milliseconds.
- **`TemplateNumber` restore semantics.** The RSSI threshold uses `on_value`
  rather than `set_action` specifically because `setup()` restores via
  `publish_state()` and never `control()`. If that changes, the restored value
  would silently stop being applied. The scan-profile select relies on the same
  thing, and on `on_value` seeing the new option (`set_action` sees the old one).

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
