# Prebuilt firmware

What CI compiles and the [browser flasher](https://davidcoulson.github.io/esphome-ble-proxy/)
installs. Nothing here is committed as a binary — the images are built on every
deploy and published to GitHub Pages.

## Shape

```
<board>.yaml        the adoption package + dashboard_import. One per board.
variants/*.yaml     a bluetooth_proxy: override block. One per build variant.
compose.sh          board x variant -> a buildable config
```

`compose.sh` exists so a board is described once. Main-config keys beat package
keys in ESPHome, so the variant's `bluetooth_proxy:` block overrides whatever
`config/adopt/<board>.yaml` set:

```bash
./compose.sh esp32s3 noapple > _build.yaml
esphome config _build.yaml
```

## The variants

Both axes are compile-time. That is the entire reason they are variants rather
than settings — everything tunable at runtime (RSSI threshold, RSSI floor, scan
profile) is a Home Assistant entity instead.

| | Apple blocked | Relays connections |
|---|---|---|
| `standard` | no | yes |
| `standard-adv` | no | no |
| `noapple` | **yes** | yes |
| `noapple-adv` | **yes** | no |

**The Apple blocklist is only safe with IRKs.** Blocking `0x004C` removes ~69%
of what a proxy hears, but your own iPhones advertise Apple data too. They get
through only when their resolvable private address matches an IRK: that match
categorises the advertisement as `CAT_IRK` and marks it *protected*, and
protected advertisements skip the payload filters where the blocklist lives.

A prebuilt binary starts with no IRKs, so `noapple` drops your phones until you
supply them. That happens from Home Assistant, at runtime, with no reflash (see
[docs/irks-from-ha.md](../docs/irks-from-ha.md)). A proxy that exists only for
sensors and tags can skip that entirely.

## Device names

`esphome.name` must be ≤17 characters: the hostname cap is 24 and
`name_add_mac_suffix: true` appends 7 (`-aabbcc`). The name is deliberately the
same across a board's four variants — it is the same product, and the variants
are published to different paths rather than named differently.

## Adding a board

1. `config/adopt/<slug>.yaml` — the real configuration. Keep it **secret-free**:
   it doubles as the `dashboard_import` target, and whoever adopts the device
   will not have your `!secret` keys.
2. `firmware/<slug>.yaml` — that package plus `dashboard_import`.
3. Add the slug to `matrix.slug` in `.github/workflows/pages.yml`.
4. Add a `BOARDS` entry in `wizard/index.html` with `fw`/`short`/`sub`, and list
   its key in `QUICK_ORDER`.
