# The config wizard

A single static `index.html` — no build step, no dependencies, no network calls.
Nothing you type in it leaves the browser.

**Live:** https://davidcoulson.github.io/esphome-ble-proxy/

**Locally:** open `index.html` in a browser. That's it.

Deployed by `.github/workflows/pages.yml`, which uploads this directory as the
Pages site root on any push to `main` that touches it.

## What it generates

Two output modes:

- **Standalone** — one self-contained device YAML with the proxy, the filter
  config, the scan profiles and all the diagnostic entities inline. Needs
  nothing else from this repo. This is the one to use when setting up a proxy
  from scratch.
- **Uses `common/` packages** — a short device file that includes the shared
  packages from `config/common/`, plus per-device overrides only where the
  answers differ from the package defaults. For adding to an existing fleet.

Plus a matching `secrets.yaml` stub and a next-steps checklist.

## Editing it

Board definitions are the `BOARDS` object near the top of the `<script>`. Each
entry drives the variant, flash size, family package, sdkconfig flags, default
scan profile and the hint text. Adding a board is one object entry.

The generated YAML keeps the rationale comments from `config/common/` — that is
deliberate. Someone who generates a config and never reads the docs should still
learn why passive scanning is passive and why `batch_delay` won't help them.
