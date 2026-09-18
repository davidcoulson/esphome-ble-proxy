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
./compose.sh esp32s3 nophones > _build.yaml
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
| `nophones` | **yes** | yes |
| `nophones-adv` | **yes** | no |

**Why "nophones" is a separate build and not a default.** Blocking `0x004C`
removes ~69% of what a proxy hears. On the fleet that is safe because the fleet
lists its own phones' IRKs — an advertisement whose resolvable private address
matches one is categorised as `CAT_IRK`, marked *protected*, and skips the
payload filters where the manufacturer blocklist lives. With an empty `irks:`
list nothing is protected, so the blocklist takes your own iPhones too.

A prebuilt binary cannot contain a per-person IRK, so the honest options are
"block Apple and lose phones" or "keep phones and keep the noise". Both are
legitimate: a proxy that exists to reach BTHome/Xiaomi/SwitchBot sensors and
Tiles has no reason to hear a single Apple packet.

To get both, adopt the device (`dashboard_import` makes it adoptable) and add
`irks:` in your own dashboard.

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
