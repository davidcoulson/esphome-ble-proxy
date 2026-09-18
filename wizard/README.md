# The wizard

A single static `index.html` — no build step, no framework. Two modes:

**Quick flash** — pick a board, install prebuilt firmware over Web Serial, hand
it your Wi-Fi over Improv. Nothing you type goes anywhere except down the USB
cable. Loads [ESP Web Tools](https://github.com/esphome/esp-web-tools) (pinned,
from unpkg) for the install button; everything else is inline.

**Advanced** — the config generator. Two outputs:

- *Standalone*: one self-contained device YAML with the proxy, the filter
  config, the scan profiles and all the diagnostic entities inline. Needs
  nothing else from this repo.
- *Uses `common/` packages*: a short device file including the shared packages,
  with per-device overrides only where your answers differ from the defaults.

Plus a matching `secrets.yaml` stub and a next-steps checklist that warns on the
combinations that bite — no IRKs, an Aggressive profile on a shared radio, a
threshold tight enough to blind the tracker.

**Live:** https://davidcoulson.github.io/esphome-ble-proxy/
Deep links: `#quick`, `#advanced`.

## Running it locally

Open `index.html` in a browser and the Advanced mode works immediately.

Quick flash will not: the install button needs
`firmware/<board>/<variant>/manifest.json` next to the page, and those binaries
are produced by CI, not committed. To try it locally, build one and drop it in
place:

```bash
firmware/compose.sh esp32s3 standard > firmware/_build.yaml && esphome compile firmware/_build.yaml
```

Web Serial also needs a secure context — `localhost` counts, `file://` does not,
so serve the directory rather than opening the file.

## How the firmware gets there

`.github/workflows/pages.yml` runs a board x variant matrix (7 x 4 = 28),
composing each config with `firmware/compose.sh`, compiling it with
`esphome/build-action`, then assembling the Pages site as `wizard/` plus
`firmware/<board>/<variant>/`. The manifest and its `.bin` must stay in the same
directory — the manifest references binaries by bare filename.

Every board is rebuilt on every deploy. The deploy publishes the whole site at
once, so a partial build would silently drop firmware that is already live.

## Adding a board

1. `config/adopt/<slug>.yaml` — the real configuration, including
   `!include proxy-core.yaml`. Keep it **secret-free**: it doubles as the
   `dashboard_import` target, and someone adopting the device will not have your
   `!secret` keys.
2. `firmware/<slug>.yaml` — the adoption package plus `dashboard_import`.
   Device name must be ≤17 characters (24-char hostname cap, minus the 7-char
   MAC suffix `name_add_mac_suffix` appends).
3. Add the slug to `matrix.slug` in `.github/workflows/pages.yml`. It is built
   against every variant automatically.
4. Add an entry to `BOARDS` in `index.html` with `fw`, `short` and `sub`, and
   list its key in `QUICK_ORDER`.

`BOARDS` drives both modes: variant, flash size, family package, sdkconfig
flags, default scan profile and hint text all come from that one object.

## A note on the generated YAML

It keeps the rationale comments from `config/common/`. That is deliberate —
someone who generates a config and never opens the docs should still find out
why scanning is passive and why `batch_delay` will not help them.
