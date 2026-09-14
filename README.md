# Dual-OS Glove80 keymap (macOS / Linux)

A [ZMK](https://zmk.dev) keymap for the [MoErgo Glove80](https://www.moergo.com/) split
keyboard with a **runtime** macOS/Linux mode switch, bilateral-combination home row
mods, and OS-aware RGB underglow. Layout ideas, the bilateral-mods technique, and the
OS-overlay-layer pattern are adapted from [sunaku/glove80-keymaps][sunaku], trimmed
down and rebuilt as a plain, from-scratch, CI-buildable ZMK config (sunaku's own repo
is a set of Ruby templates pasted into MoErgo's web editor, not a buildable config).

[sunaku]: https://github.com/sunaku/glove80-keymaps

A full visual reference of every layer is in [`docs/keymap.png`](docs/keymap.png)
(`docs/keymap-compact.png` is the same thing with the three near-empty macOS
sub-overlay layers omitted). Regenerate it any time with:

```sh
pip install --user keymap-drawer
keymap parse -z config/glove80.keymap -c 6 -o docs/keymap.yaml
# docs/keymap.yaml's `layout:` line points at config/info.json (Glove80's real
# physical key positions) -- keep that line as-is when re-parsing.
keymap draw docs/keymap.yaml -o docs/keymap.svg
# any SVG->PNG rasterizer with real CSS support works; resvg-cli renders it
# correctly, cairosvg does not (it drops the class-based key styling):
npx --yes resvg-cli --fit-width 2000 --background white docs/keymap.svg docs/keymap.png
```

## Building / flashing

This repo follows MoErgo's official [`glove80-zmk-config`][template] template layout, so
it builds the same way:

1. Push this repo to your own GitHub repo (or fork MoErgo's template and copy these
   files in).
2. GitHub Actions (`.github/workflows/build.yml`) builds against `moergo-sc/zmk@main`
   and uploads a `glove80.uf2` artifact on every push.
3. Download the artifact, put both keyboard halves in bootloader mode, and copy
   `glove80.uf2` onto the `GLV80LHBOOT` / `GLV80RHBOOT` USB drive that appears (MoErgo's
   [flashing guide][flashing] has the details).

`build.sh` / `Dockerfile` / `build.bat` mirror the same Nix build locally if you'd
rather not wait on CI.

[template]: https://github.com/moergo-sc/glove80-zmk-config
[flashing]: https://docs.moergo.com/glove80-user-guide/

## Layers

| # | Layer | Access | Contents |
|---|-------|--------|----------|
| 0 | `default` | base | QWERTY + bilateral home row mods |
| 1 | `cursor` | hold left thumb (`&mo CURSOR`) | arrows, page up/down, word-left/right, insert |
| 2 | `number` | hold left thumb (`&mo NUMBER`) | numpad on the right hand |
| 3 | `function` | hold left thumb (`&mo FUNCTION`) | F11-F20 |
| 4 | `symbol` | tap/hold `&symbol_td` | `!@#$%^&*()[]{}` etc. |
| 5 | `mouse` | hold right thumb (`&mo MOUSE`) | pointer move/click/scroll |
| 6 | `system` | hold right thumb (`&mo SYSTEM`) | media keys, BT profile select, output select |
| 7 | `lower` | tap/hold `&lower_td` | secondary numpad/nav/media, mirrors stock MoErgo Lower |
| 8 | `magic` | hold outer-corner key (`&magic`) | BT/RGB controls, **OS mode select**, bootloader/reset, Factory escape hatch |
| 9 | `factory` | `&to FACTORY` from Magic | plain QWERTY, no mods, no custom layers -- safety net |
| 10 | `macos` | `&to MACOS` / `&to DEFAULT` (Magic layer) | OS overlay, see below |
| 11 | `macos_left` | automatic (macOS + Cursor/Number/Function) | reserved for future Mac-specific left-hand tweaks |
| 12 | `macos_right` | automatic (macOS + Symbol/Mouse/System) | reserved for future Mac-specific right-hand tweaks |
| 13 | `macos_lower` | automatic (macOS + Lower) | reserved for future Mac-specific Lower tweaks |

No gaming layer, per your request.

## OS switching

ZMK has no way to detect which OS the connected host is running -- there's no such
signal over USB or BLE. So switching "OS mode" here means: two keys on the Magic layer,
top-left area, marked **`os_to_linux`** and **`os_to_macos`**. Each one unconditionally:

1. Sets the persistent `macos` overlay layer on or off (`&to MACOS` / `&to DEFAULT`), and
2. Recolors the underglow (`&rgb_ug RGB_COLOR_HSB(...)`) -- cyan/teal for Linux, warm
   red for macOS -- so the lighting always matches the current mode.

They're plain "set to X" actions rather than a toggle, so the keyboard's mode can't get
out of sync with what you last pressed.

While `macos` is active, the overlay:

- **Swaps the pinky/middle home-row mods** on both hands: pinky (`A` / `;`) becomes
  Ctrl instead of Gui, middle (`D` / `K`) becomes Gui instead of Ctrl. This mirrors
  sunaku's `PINKY_FINGER_MOD`/`MIDDY_FINGER_MOD` swap for macOS. Because of the
  bilateral-mods rule below, you naturally reach for a modifier with the hand
  *opposite* the letter you're pressing -- so e.g. Cmd+C on macOS is: hold the right
  hand's `K` (now Gui) and tap `C` with the left hand. No separate Copy/Paste/Undo
  macros are needed; the same finger combo just produces the OS-correct modifier.
- **Remaps Home/End to Cmd+Left/Right** (`LG(LEFT)`/`LG(RIGHT)`), matching macOS's
  line-start/line-end convention instead of the Linux `HOME`/`END` keycodes.
- The `macos_left`/`macos_right`/`macos_lower` layers exist (via `conditional-layers`,
  see `config/glove80.keymap`) so that if you later want Mac-specific bindings on
  Cursor/Number/Function/Symbol/Mouse/System/Lower too, there's already a layer to put
  them on -- they're just fully transparent placeholders for now.

## Bilateral combinations (home row mods)

Home row mods (hold `A` for Gui, `S` for Alt, `D` for Ctrl, `F` for Shift, mirrored on
the right hand) are the classic way to get every modifier under your fingers without
dedicated keys -- but they can misfire during fast same-hand typing (e.g. rolling
`"as"` triggers a phantom Alt-hold on `A`). This keymap uses **bilateral
combinations** (a term [sunaku's docs][bilateral] use for this exact ZMK technique) to
prevent that:

```c
hold-trigger-key-positions = <RIGHT_HAND_KEYS>;  // (or LEFT_HAND_KEYS, mirrored)
hold-trigger-on-release;
```

Each home-row-mod key only resolves as a **hold** if the very next key pressed is on
the *opposite* hand. Same-hand rolls always resolve as a plain tap. In practice this
means: reach across with the other hand's home-row mod for any shortcut whose letter is
on the same side (see the Cmd+C example above).

`LEFT_HAND_KEYS` / `RIGHT_HAND_KEYS` in `config/glove80.keymap` list Glove80's 80 key
positions (0-79, physical row-major order, thumb clusters included) split by hand --
see the comment block above them for how the numbering works.

[bilateral]: https://sunaku.github.io/home-row-mods.html

## RGB lighting

OS-based lighting is implemented with mainline, stock-firmware-safe
`&rgb_ug RGB_COLOR_HSB(h,s,b)` commands (see `os_to_macos`/`os_to_linux` macros in
`config/glove80.keymap`), triggered by the OS-select keys on the Magic layer. This
works on Glove80's regular firmware, no special build required.

I initially looked into per-key/per-layer RGB indicators (the richer effect you'd get
from lighting just a couple of accent keys differently per layer, the way sunaku's own
config lights two corner keys red specifically in Mac mode) via the community
`darknao/zmk` fork's `zmk,underglow-layer` devicetree node. I didn't include it here:
that feature needs a `pixel-lookup` table mapping each of Glove80's 80 key positions to
a physical LED index, which isn't documented anywhere I could verify against real
hardware, and shipping a guessed table risked a broken build or wrong colors. The
whole-board color swap is the reliable version of "OS-based lighting" -- if you want to
chase the per-key version later, `darknao/zmk`'s `rgb-layer-25.08` branch (and its
`app/dts/bindings/zmk,underglow-layer.yaml`) is the place to start, ideally tested
incrementally on real hardware.

## Mouse layer

Pointer emulation (`&mmv`/`&mkp`/`&msc`) needs `CONFIG_ZMK_POINTING=y` (already set in
`config/glove80.conf`) and, the **first time** you use it over Bluetooth, a HID
descriptor refresh on the host (unpair/re-pair, or your OS's Bluetooth device removal +
re-add) -- this is a one-time ZMK/BLE requirement, not specific to this keymap.
