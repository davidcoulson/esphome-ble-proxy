# Operations

## Where things live

| | |
|---|---|
| Host | Home Assistant OS, `ssh hassio@10.2.3.6` (key auth) |
| Config | `/config/esphome` (also `/homeassistant/esphome`) |
| Shared packages | `/config/esphome/common/` |
| Secrets | `/config/esphome/secrets.yaml` — never print, copy or commit |
| ESPHome | 2026.9.0, stock **ESPHome Device Builder** add-on, port 6052 |
| Git | `/config/esphome` is a repo, remote `davidcoulson/esphome-config` (private) |

Practical limits of the `hassio` SSH user:

- `esphome` is **not** on PATH, and `docker` / `ha` are unusable (no socket, no
  token). Validate, compile, flash and read logs through the **ESPHome MCP
  server** or the Device Builder web UI.
- Git refuses the repo as "dubious ownership". Use `git -c safe.directory='*'`
  for read-only work; commits need `sudo`.
- `sudo -n` works for reading root-owned files.
- Device Builder auto-commits edits made through **its UI** ("Edit x.yaml").
  Edits made over SSH to `common/` are **not** committed — commit them as root,
  or you will lose them to a later UI-driven commit.

## Adding a proxy

1. Write the device file — four substitutions and one include, see
   [QUICKSTART.md](../QUICKSTART.md) or `config/devices/`.
2. Validate. One device per affected family is enough for a `common/` change.
3. **Compile** before flashing anything shared. A validation pass does not
   compile lambdas, and half the interesting code here is in lambdas.
4. Install to **one canary device** first.
5. Watch it for ten minutes: uptime climbing, **BLE Adverts Forwarded** above
   zero, **Gateway Watchdog Reboots Used** at 0.
6. Then roll the rest of the family.

## Flashing

**First flash is over USB.** Every one after that is over the air.

```bash
esphome run /config/esphome/ble-esp32s3-a1b2c3.yaml
```

Or via the Device Builder UI: **Install → Plug into this computer** / **Wirelessly**.

### Never mass-flash

A fleet-wide OTA once put **~40 nodes into a rolling reboot loop** via the
gateway watchdog. A deliberate 55-device reproduction later produced zero packet
loss on 35 control nodes, so congestion is not the cause and the mechanism is
still unknown.

**Roll out by family, a handful at a time, and watch
`Gateway Watchdog Reboots Used` between batches.** The watchdog's `max_reboots: 2`
cap is what stops a repeat becoming unbounded, but it is a backstop, not a
licence.

### A node will not take an OTA

Almost always a C3 / C6 / ESP8685 whose radio is saturated by BLE scanning.
WiFi and BLE share one antenna there, and the OTA connection cannot even be
established.

**The `ota: on_begin: stop_scan` hook cannot rescue this.** It only fires once
the OTA has *already* begun — and the problem is that it cannot begin.

The fix is **safe mode**, which boots without BLE:

1. Press the **Safe Mode** button in Home Assistant (every proxy has one), or
   power-cycle the device rapidly several times.
2. Flash it while it is in safe mode.
3. It reboots into the new firmware normally.

## Reading logs

Logs reach the API and the device's own web UI. They do **not** reach the serial
port: `logger.baud_rate: 0` on every proxy, which disables the UART outright to
free CPU and memory.

`logger.level: WARN` is the default. ESPHome strips log calls above the
configured level **at compile time**, so raising it to DEBUG to chase something
means a rebuild and a reflash. For the filter specifically, the per-packet drop
reasons are at `VERY_VERBOSE`.

## Offloaded builds

Builds can be handed to paired remote builders (`.offloader_pairings.json`).
This is the reason every external component is referenced by **git URL, never
`type: local`** — a local path only exists on the Home Assistant box.

It is also why `refresh:` is short despite the pinned tags: each build host
keeps its own `.esphome/external_components/` cache, and a stale one has already
served a build the wrong code. If a build refuses to pick up a new tag, delete
`.esphome/external_components/<hash>` — `refresh:` alone does not always
suffice.

## Backups before bulk edits

Copy to `/config/esphome/.bak-<topic>-<date>/` first. `.gitignore` covers
`*.bak-*` anywhere, so stray backups stay out of git.

Note that `secrets.yaml` alone is **not** a sufficient gitignore entry — the
timestamped copies that accumulate next to it (`secrets.yaml.bak-20260911-144828`
and friends) hold the real WiFi password, every API/OTA key and every IRK in
plain text. `secrets.yaml*` is the entry that covers them.

## Routine checks

| Signal | Healthy | Investigate when |
|---|---|---|
| **BLE Adverts Forwarded** | > 0, roughly comparable across proxies | zero, or a lone outlier — usually placement |
| **BLE Advert Drop Rate** | ~80% median with the Apple blocklist, ~55–65% without | far below its siblings (IRKs missing?), or a sudden jump with devices vanishing |
| **Gateway Watchdog Reboots Used** | 0 | anything above 0, especially right after an OTA |
| **Uptime** | days | repeated resets — check Reset Reason |
| **Project Version** | same across a family | a straggler means a failed OTA |
