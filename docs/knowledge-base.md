# Knowledge base

Dated, evidence-backed findings. Mark anything not directly verified as *unverified*.
Newest entries go at the top of each section.

## Repo & fork

- **2026-10-22** — The fork was first created default-branch-only as `vergilcw/zmk-for-crosses` (branch `35`
  only). It was renamed to `vergilcw/zmk-for-crosses36`, upstream `36`, `46`, `46_RGB` were pushed, and the
  default branch set to `36`. Because both `origin` and `upstream` have a `36`, `git switch 36` is ambiguous on a
  fresh clone with both remotes; use `git switch --track origin/36`.
- **2026-10-22** — Upstream `HeeTuic/zmk-for-crosses` keeps one branch per variant (`35`, `36`, `46`, `46_RGB`).
  Branch `36` README: "This branch is for the 3x6 layout single-trackball variant". Other HeeTuic repos
  (`zmk-for-crosses36-dual`, `-35-dual`, `-46-dual`) are dual-trackball variants, not this keyboard.

## Device

- **2026-10-23** — Right half enumerates as `/dev/serial/by-id/usb-ZMK_Project_Crosses_36_<serial>-if00` →
  `/dev/ttyACM0`, matching `CONFIG_ZMK_KEYBOARD_NAME="Crosses 36"` in `config/crosses.conf`. So the board
  runs branch-`36`-derived firmware.
- **2026-10-23** — User confirmed the board is factory-stock with no ZMK Studio edits; no stock dump was taken.
- Right half is the BLE central (`ZMK_SPLIT_BLE_ROLE_CENTRAL` in `Kconfig.defconfig` under
  `SHIELD_CROSSES_RIGHT`), has the trackball and Studio. No displays on either half.

## Firmware stack (branch `36`)

- ZMK `v0.3` (Zephyr 3.5) via `config/west.yml`; CI uses `build-user-config.yml@v0.3.0`.
- Trackball driver `DoctorWangWang/zmk-pmw3610-driver@main` (commit `1d9c2c6`, 2024-04-13), compatible
  `pixart,pmw3610`, configured via `CONFIG_PMW3610_*` Kconfig plus DT `scroll-layers` / `snipe-layers` /
  `automouse-layer`.
- Shield lives in-repo at `config/boards/shields/crosses/`; physical layout `crosses_layout` (42 keys) is chosen
  in `crosses.dtsi`, so the keymap must not set `zmk,physical-layout`.
- `build.yaml` board is `nice_nano_v2`; `artifact-name` (supported by `build-user-config.yml@v0.3.0`) names the
  outputs `crosses_36_left` / `crosses_36_right`, distinct from the daily driver's `crosses_35_*`.

## Trackball scrolling via ZMK overrides (2026-10-23; superseded by driver layers, see below)

- The driver's `scroll-layers` mode scrolls **both** axes (whichever accumulated delta exceeds
  `CONFIG_PMW3610_SCROLL_TICK` first) and can't lock an axis (`src/pmw3610.c` `pmw3610_report_data`).
  The driver picks its mode from `zmk_keymap_highest_layer_active()`.
- ZMK `v0.3` input-listener child nodes (`layers`, `input-processors`, optional `process-next`) replace the base
  processor chain while any listed layer is active. The first matching child wins
  (`app/src/pointing/input_listener.c` `filter_with_input_config`;
  `app/dts/bindings/zmk,input-listener.yaml`).
- `v0.3` ships `zip_x_scaler`, `zip_y_scaler`, `zip_scroll_scaler` (all `track-remainders`),
  `zip_xy_to_scroll_mapper` (Y→WHEEL, X→HWHEEL, sign kept) and `zip_scroll_transform`, via
  `#include <input/processors.dtsi>`. `INPUT_TRANSFORM_{X,Y}_INVERT` come from
  `<dt-bindings/zmk/input_transform.h>`. A scaler with multiplier `0` sets the value to 0
  (`input_processor_scaler.c: scale_val`).
- Driver sign conventions: in scroll mode, Y delta > 0 emits WHEEL −1 (no `INVERT_SCROLL_Y`). X delta (already
  flipped by `INVERT_X`) > 0 emits HWHEEL +1, because `INVERT_SCROLL_X` defaults to `INVERT_X`. So with ZMK's
  mapper, invert Y and leave X as is to match the stock direction.
- Driver scroll mode reads with divisor 1 and tick 24; pointer mode divides by `CPI_DIVIDOR` (4). In pointer
  mode one wheel step ≈ 24/4 = 6 counts → `zip_scroll_scaler 1 6`.
- Implemented 2026-10-23 in `crosses_right.overlay` (`lower_vscroll` on layer 1, `raise_hscroll` on layer 2).
  Compiles: CI run `39998744897` (2026-10-23) is green, and the right half's final devicetree (printed in the
  build log) contains both override nodes with the expected `input-processors`. *Not yet tested on hardware.*

## Trackball rotation & driver layers (2026-10-23)

- The driver picks scroll/snipe/move from `zmk_keymap_highest_layer_active()` against DT `scroll-layers` /
  `snipe-layers`; snipe changes the sensor CPI (`SNIPE_CPI` 400, dividor 4 → ¼ of normal 1600/4).
- Lower+Raise can't be held together on this keymap: on Lower the Raise thumb is `&mkp RCLK`, on Raise the Lower
  thumb is `&kp DEL` (whichever layer is active when the second thumb is pressed decides its binding).
- ZMK v0.3 input processors/overrides depend on layers only, not on modifiers.
- Rotation candidates rejected: `hsgw/zmk-feature-sensor_rotation@e1a0186` rotates each axis event using the
  other axis's value from the *previous* report (stateful skew on diagonals) and truncates per event;
  `efogdev/zmk-input-processor-rotate-plane` needs `zmk_runtime_config` (efogdev) and targets newer ZMK.
- Rotation implemented in the vendored driver (`modules/zmk-pmw3610-driver`, `CONFIG_PMW3610_ROTATION_DEG`):
  after orientation/inversion, before the dividor, Q14 integer math with compile-time Taylor-series cos/sin and
  per-axis remainders (reset on mode change). With the option at 0 the code path is the original.
  Host-side test (gcc, the patched lines extracted verbatim): −45 → cos = sin = 11585 (exact 11585.2); a roll
  toward 10:30 gives (0, −1060) over 1000 reports vs ideal (0, −1061); slow (1,0)/4 motion is no longer lost.
- `build-user-config.yml@v0.3.0` passes `-DZMK_EXTRA_MODULES=$GITHUB_WORKSPACE` when `zephyr/module.yml` exists at
  the repo root; ours points `cmake`/`kconfig`/`dts_root` at `modules/zmk-pmw3610-driver`.

## Decisions

See `docs/plan.md` § Decisions (D1–D8). Summary of user-confirmed ones:

- **D2 (2026-10-22)** — Left Alt on both bottom-pinky outer keys.
- **D5 (2026-10-23)** — Q+W → Esc combo dropped.
- **D8 (2026-10-23)** — `;` on the right home row, `'` on the right outer column (a deliberate deviation from the
  daily driver).
- **D9/D10 (2026-10-23)** — Lower = 2-axis scroll, Raise = snipe, Mouse = normal; all movement rotated −45°
  (negative = counterclockwise, user's convention).

## Tooling

- **2026-10-23** — keymap-drawer renders work offline by copying the glyph cache from the sister repo's
  `.cache/keymap-drawer/glyphs/` into this repo's `.cache/` and running `keymap` with
  `XDG_CACHE_HOME=<repo>/.cache`. All 32 `mdi:` glyphs end up embedded in `img/crosses.svg`.
  `rsvg-convert` doesn't draw those glyphs when rasterizing; view the SVG in a browser instead.
