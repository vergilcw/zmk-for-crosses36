# Plan — Crosses 36 (3x6 single trackball) firmware

Handoff plan written 2026-10-22 from the daily-driver session
(`vergilcw/zmk-for-crosses35-dual`). Status legend: `[x]` done, `[ ]` todo, `[~]` in progress, `[-]` skipped.

## Goal

Make the secondary keyboard (Crosses 3x6+3, single trackball on the right half) run:

1. **The exact same keymap as the daily driver** (3x5+3 dual trackball), placed on the inner
   3x5+3 block, **plus** six new outer-column keys on Base:

   | Position | Key | Binding |
   | --- | --- | --- |
   | Left pinky top (pos 0) | backtick / tilde (shift) | `&kp GRAVE` |
   | Left pinky middle (pos 12) | Escape | `&kp ESC` |
   | Left pinky bottom (pos 24) | Alt | `&kp LALT` |
   | Right pinky top (pos 11) | underscore | `&kp UNDER` |
   | Right pinky middle (pos 23) | quote | `&kp SQT` (swapped with `;`, see D8) |
   | Right pinky bottom (pos 35) | Alt | `&kp LALT` (see decision D2) |

2. **Trackball behaviour by layer** (revised 2026-10-23, see D9/D10): normal pointer on Base and
   Mouse, **2-axis scroll while Lower is held**, **snipe (slow pointer) while Raise is held**; all
   movement rotated 45° counterclockwise (`CONFIG_PMW3610_ROTATION_DEG=-45`).

Out of scope for this task: mod-tap/hold-tap tuning, connectivity work, anything on the daily driver.

## Research summary (verified 2026-10-22)

- Correct firmware: **`HeeTuic/zmk-for-crosses`, branch `36`** — its README says "This branch is for
  the 3x6 layout single-trackball variant"; `crosses_right.overlay` defines one PMW3610 on SPI0, the
  left overlay has none. Upstream CI on branch `36` is green (last runs 2026-05-28). Other HeeTuic repos
  (`zmk-for-crosses36-dual`, `-35-dual`, `-46-dual`) are dual-trackball variants — not this keyboard.
- Branch `36` stack: ZMK `v0.3` branch (Zephyr 3.5), `DoctorWangWang/zmk-pmw3610-driver@main`
  (commit `1d9c2c6`, 2024-04-13), in-repo shield `config/boards/shields/crosses/`, 42-key physical
  layout `crosses_layout`, matrix 4 rows × 12 cols, `build.yaml` board `nice_nano_v2`, right half has
  Studio (`studio-rpc-usb-uart` snippet, `CONFIG_ZMK_STUDIO=y`, locking on).
- Stock trackball config (`crosses_right.conf`): `CPI=1600`, `CPI_DIVIDOR=4`, `ORIENTATION_270`,
  `INVERT_X`, snipe `CPI=400`/`DIVIDOR=4`, `SCROLL_TICK=24`, `POLLING_RATE_125_SW`, smart algorithm;
  overlay: `snipe-layers = <3>`, `scroll-layers` commented out, plain `trackball_listener`.
- The driver's own `scroll-layers` scroll **both axes** (whichever exceeds the tick) — it cannot lock
  an axis. ZMK v0.3 itself can: input-listener child nodes with `layers = <N>` override the processor
  chain while that layer is active (`app/src/pointing/input_listener.c`, `filter_with_input_config`),
  and v0.3 ships `zip_x_scaler`, `zip_y_scaler`, `zip_scroll_scaler`, `zip_xy_to_scroll_mapper`,
  `zip_scroll_transform` (`app/dts/input/processors/*.dtsi`). The scaler with multiplier `0` sets the
  event value to 0 (`input_processor_scaler.c: scale_val`), which is how one axis is suppressed.
- Driver sign conventions (for matching the stock scroll direction): in its scroll mode, Y movement > 0
  emits WHEEL −1 (no `INVERT_SCROLL_Y`), X movement (already inverted by `INVERT_X`) > 0 emits HWHEEL +1
  (`INVERT_SCROLL_X` defaults to `INVERT_X`). With ZMK's mapper (Y→WHEEL, X→HWHEEL, same sign) we must
  **invert Y** to match; X needs no inversion. Driver scroll mode uses divisor 1 with tick 24, while
  pointer mode divides by 4 — so `zip_scroll_scaler 1 6` ≈ the stock scroll speed.
- `CONFIG_ZMK_MOUSE=y` (used upstream) implies `CONFIG_ZMK_POINTING` at v0.3 — no change needed.

## Decisions (confirm with the user if they object)

- **D1 — Outer columns on non-Base layers are `&trans`.** Esc/Alt/`/_/; stay reachable on every layer.
- **D2 — Both bottom-pinky Alts are `LALT`** (confirmed by user 2026-10-22). `RALT` acts as AltGr in many layouts; Base already has a
  `RALT` mod-tap on `/` if a right Alt is ever needed. Ask the user if they want `RALT` instead.
- **D3 — Drop upstream's global `&lt` override** (`tapping-term 200, balanced, quick-tap 150`) so
  layer-taps feel identical to the daily driver (which uses stock `&lt`). Typing tuning is a later,
  shared task for both keyboards.
- **D4 — ~~Keep driver sniper on the Mouse layer~~ Superseded by D9** (user, 2026-10-23): Mouse keeps a normal pointer.
- **D5 — ~~Keep the Q+W → Esc combo~~ Dropped** (user, 2026-10-23): Esc has a dedicated key now.
- **D6 — Drop the unused `&as` (auto_shift) hold-tap** from the daily-driver keymap; nothing binds it.
- **D7 — ~~Scroll implemented with ZMK input-listener layer overrides~~ Superseded by D9**, not the driver's
  `scroll-layers` (which can't lock an axis). Fallback if overrides misbehave: driver `scroll-layers`
  for free 2-axis scroll on Lower/Raise.
- **D8 — `;` on the home row (pos 22), `'` on the outer column (pos 23)** (user, 2026-10-23).
  Deliberate deviation from the daily driver, which has `'` at that home-row position.
- **D9 — Trackball layers via the driver** (user, 2026-10-23): `scroll-layers = <1>` (Lower, both axes;
  the user dropped axis-locked scroll), `snipe-layers = <2>` (Raise), Mouse normal. The ZMK input-listener
  overrides (Appendix B) were removed. Alternatives considered and rejected: Shift+Lower for horizontal
  scroll (v0.3 processors can't see modifiers; Shift+HWHEEL is host-dependent); Lower+Raise (the other
  thumb is `RCLK`/`DEL` on those layers, so the combination can't be reached).
- **D10 — Rotate all trackball movement 45° CCW in the vendored driver** (user, 2026-10-23): ZMK v0.3 has
  no rotation processor and per-axis events can't be rotated correctly by a stateless processor
  (`hsgw/zmk-feature-sensor_rotation` pairs each axis with the *previous* report's other axis;
  `efogdev/zmk-input-processor-rotate-plane` needs efogdev's runtime-config module and newer ZMK). The driver
  is vendored in `modules/zmk-pmw3610-driver` with `CONFIG_PMW3610_ROTATION_DEG` (negative = CCW, the user's
  preferred convention), applied to raw counts before the dividor with remainder tracking.

## Inbox

- [ ] (empty)

## Phase 0 — Repo setup (user does this before starting the agent)

```fish
gh repo fork HeeTuic/zmk-for-crosses --clone=false --fork-name zmk-for-crosses36 --default-branch-only=false
gh repo edit vergilcw/zmk-for-crosses36 --default-branch 36
gh repo clone vergilcw/zmk-for-crosses36 ~/zmk-for-crosses36 -- -b 36
cd ~/zmk-for-crosses36
mkdir -p docs
# copy this file to docs/plan.md and the companion AGENTS.md to the repo root, then:
git add AGENTS.md docs/plan.md
git commit -m "Add agent instructions and plan"
```

- [x] Fork exists with branch `36` as default; working copy on `36`.
      (2026-10-22: fork was created default-branch-only as `vergilcw/zmk-for-crosses`; renamed to
      `zmk-for-crosses36`, pushed upstream `36`, `46`, `46_RGB`, set default to `36`.)
- [x] `AGENTS.md` + `docs/plan.md` committed.

## Phase 1 — Confirm the device runs this firmware & back it up (read-only)

- [x] Add a `.gitignore` (`.cache/`, `__pycache__/`, `.venv/`, `*.uf2`).
- [ ] *Deferred to Phase 6 (only needed for the post-flash re-dump).* Copy the Studio tools from the sister repo (local checkout, absolute path):
      `cp /home/vergilcw/zmk-for-crosses35-dual/scripts/studio_dump.py scripts/`
      (also `studio_to_keymap.py` if wanted — note it currently assumes 36 keys/rows of 10; it would need
      its `rows`/grid logic generalised to 12-column rows + 6 thumbs to print this board's layers).
- [x] Ask the user to plug in the **right** half over USB. Expect
      `/dev/serial/by-id/usb-ZMK_Project_Crosses_36_*` (name from `config/crosses.conf`). If the name
      differs, stop and report — the board may be running different firmware.
      (2026-10-23: `usb-ZMK_Project_Crosses_36_*` → `/dev/ttyACM0`, the only ZMK device attached.)
- [-] *Skipped (2026-10-23): user confirmed the board is factory-stock with no Studio edits; no backup wanted.* Ask the user to press Studio unlock on the **stock** keymap: hold `Z` or `/` (Mouse layer) and
      press the `N` key position (pos 30).
- [-] *Skipped (see above).* Dump: `env UV_PYTHON_DOWNLOADS=never uv run scripts/studio_dump.py /dev/ttyACM0 > docs/studio-snapshots/2026-MM-DD-stock.json`
      (the script omits the serial). Confirm: 4 layers, ≥42 bindings per layer, physical layout
      `crosses`, and bindings matching upstream `config/crosses.keymap`. If they don't match, the user
      has Studio edits on the board — record them in the knowledge base before continuing.
- [x] Check which `ttyACM` is which: the daily driver (`Crosses BT`) may also be plugged in. Never talk
      to `Crosses BT` from this repo.

## Phase 2 — Keymap

- [x] Replace `config/crosses.keymap` with **Appendix A** verbatim (already validated with keymap-drawer:
      42 bindings on each of the 4 layers, layout renders correctly against `config/crosses.json`).
- [x] Diff the inner block against the sister repo's `config/crosses.keymap` to confirm parity
      (inner pos `12*r + c + 1` here == pos `10*r + c` there; thumbs 36–41 == 30–35).
      (2026-10-23: all 4 layers match binding-for-binding; keymap-drawer parses it. Later the same day
      the user swapped `;`/`'` (D8) and dropped the combo (D5) — the only intended differences.)

## Phase 3 — Trackball scroll layers (superseded by Phase 3b)

- [x] Edit `config/boards/shields/crosses/crosses_right.overlay` per **Appendix B**: add the two includes
      and the `lower_vscroll` / `raise_hscroll` child nodes inside the existing `trackball_listener`.
      Leave the driver's `scroll-layers` commented out (it would pre-empt our overrides by emitting
      WHEEL/HWHEEL itself on those layers).
      (2026-10-23: re-verified against ZMK `v0.3` source — `input/processors.dtsi`, processor labels,
      `input_transform.h`, listener child-binding `layers`/`process-next`/`input-processors` — and the
      driver @`1d9c2c6` sign/divisor logic. Not compiled yet; CI is the first build.)
- [x] Don't change `crosses_right.conf` yet; tune only after hardware testing (Appendix D).

## Phase 3b — Rotation + driver layers (2026-10-23, after first hardware test)

- [x] Vendor `DoctorWangWang/zmk-pmw3610-driver@1d9c2c6` into `modules/zmk-pmw3610-driver` (pristine commit),
      load it via root `zephyr/module.yml`, drop it from `config/west.yml`.
- [x] Add `CONFIG_PMW3610_ROTATION_DEG` to the driver; host-side unit test of the exact patched lines:
      −45 maps a roll toward 10:30 to straight up; Q14 constants match exact cos/sin; slow motion kept.
- [x] `crosses_right.overlay`: `scroll-layers = <1>`, `snipe-layers = <2>`, listener overrides removed.
      `crosses_right.conf`: `CONFIG_PMW3610_ROTATION_DEG=-45`.
- [ ] Push (ask), CI green, flash right half only (ask), test with the user (Appendix C).

## Phase 4 — Docs, images, build matrix

- [x] Port the improved `keymap_drawer.config.yaml` from the sister repo (copy the file; it adds
      shifted-symbol legends, media/brightness glyphs, text `⇧` modifier). Ensure `UNDER: _` and
      `GRAVE: '`'` mappings exist (they do in that file).
- [x] Optional: add `artifact-name: crosses_36_left` / `crosses_36_right` to `build.yaml` entries.
- [x] Local render (optional; CI also renders to `img/`):
      `keymap -c keymap_drawer.config.yaml parse -z config/crosses.keymap > img/crosses.yaml` then
      `keymap -c keymap_drawer.config.yaml draw img/crosses.yaml -j config/crosses.json > img/crosses.svg`.
      Glyphs: `raw.githubusercontent.com` is blocked; seed
      `<repo>/.cache/keymap-drawer/glyphs/mdi:<name>.svg` via
      `gh api repos/Templarian/MaterialDesign-SVG/contents/svg/<name>.svg --jq .content | base64 -d`
      and run only the `keymap` binary with `XDG_CACHE_HOME=<repo>/.cache`.
- [x] README: replace upstream text with a short description of this keyboard, the keymap image,
      the outer-column additions, the scroll layers, flashing steps, and credits (HeeTuic, GGGW,
      DoctorWangWang, caksoylar, ZMK). Point badges at `vergilcw/zmk-for-crosses36` branch `36`.
- [x] Create `docs/knowledge-base.md` with the verified findings from this plan + anything new.
      (2026-10-23: also removed upstream `README_ZH.md`, which described HeeTuic's stock firmware;
      glyphs seeded by copying the sister repo's `.cache/keymap-drawer/glyphs/`.)

## Phase 5 — Build (needs push approval)

- [x] Ask the user, then push to `origin 36`. Watch `gh run list -R vergilcw/zmk-for-crosses36 -b 36`.
- [x] Build must be green for left, right (Studio) and settings_reset. If DT errors mention the
      processors, check the includes in Appendix B and that node/property names match v0.3
      (`input-processors`, `layers`, `process-next`).
      (2026-10-23: run `39998744897` green on all three; the right half's devicetree in the build log
      contains `lower_vscroll` / `raise_hscroll` with the expected processor chains.)
- [~] Draw workflow commits a fresh `img/crosses.svg` (done: `b756923`); user to eyeball it in a browser.

## Phase 6 — Flash (needs explicit approval; this is the secondary keyboard)

- [x] `gh run download -R vergilcw/zmk-for-crosses36 -D /tmp/crosses36-fw`
- [x] Right half: plug in, double-tap reset, copy the right `.uf2` to `/run/media/$USER/NICENANO/`.
      Then the left half. (Both halves: the keymap lives on the right, but flashing both keeps
      them on the same build.)
      (2026-10-23: both halves flashed from run `39998744897`. Right half re-enumerated as
      `ZMK_Project_Crosses_36`; left half (peripheral) has no USB serial, as expected.)
- [-] *Not needed: board was factory-stock.* If Phase 1 found Studio edits, run *Restore Stock Settings* in ZMK Studio.
      Do **not** flash `settings_reset` unless pairing breaks (it wipes BLE bonds).
- [-] *Skipped (no Studio tools in this repo); verify by testing instead (Phase 7).* Re-dump with `studio_dump.py` and confirm the device keymap == Appendix A.

## Phase 7 — Test & tune with the user (Appendix C checklist, Appendix D knobs)

- [ ] Walk the user through the checklist; adjust scroll speed/direction; one change per build.
- [ ] Record final values + rationale in the knowledge base; check off this plan.

---

## Appendix A — `config/crosses.keymap`

```dts
/*
 * Crosses 36 (3x6, single right trackball) keymap.
 *
 * Inner 3x5+3 block mirrors vergilcw/zmk-for-crosses35-dual config/crosses.keymap
 * (exported from ZMK Studio 2026-10-10), except: ; is on the home row and ' moves
 * to the outer column, and the Q+W -> Esc combo is dropped. Outer pinky columns are new:
 *   Base L: ` ~ / Esc / Alt    Base R: _ / ' / Alt    other layers: &trans
 * Trackball: vertical-only scroll on Lower, horizontal-only scroll on Raise
 * (input-listener layer overrides in boards/shields/crosses/crosses_right.overlay).
 */

#include <behaviors.dtsi>
#include <dt-bindings/zmk/bt.h>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/outputs.h>
#include <dt-bindings/zmk/pointing.h>

#define BASE  0
#define LOWER 1
#define RAISE 2
#define MOUSE 3

/ {
    keymap {
        compatible = "zmk,keymap";

        default_layer {
            display-name = "Base";
            bindings = <
&kp GRAVE  &kp Q       &kp W        &kp E  &kp R  &kp T    &kp Y  &kp U        &kp I      &kp O          &kp P          &kp UNDER
&kp ESC    &kp A       &kp S        &kp D  &kp F  &kp G    &kp H  &kp J        &kp K      &kp L          &kp SEMI       &kp SQT
&kp LALT   &mt LALT Z  &mt LSHFT X  &kp C  &kp V  &kp B    &kp N  &lt LOWER M  &kp COMMA  &mt LSHFT DOT  &mt RALT FSLH  &kp LALT
                         &lt MOUSE TAB  &mt LSHFT BSPC  &lt LOWER ESC    &mt LCTRL RET  &lt RAISE SPACE  &kp LGUI
            >;
        };

        lower_layer {
            display-name = "Lower";
            bindings = <
&trans  &kp LS(N1)    &kp HOME      &kp UP      &kp END       &kp LS(N5)      &none         &kp F1  &kp F2  &kp F3  &kp F4   &trans
&trans  &kp N1        &kp LEFT      &kp DOWN    &kp RIGHT     &kp N5          &none         &kp F4  &kp F5  &kp F7  &kp F11  &trans
&trans  &kp C_BRI_DN  &kp C_VOL_DN  &kp C_MUTE  &kp C_VOL_UP  &kp C_BRI_UP    &out OUT_TOG  &kp F1  &kp F2  &kp F3  &kp F12  &trans
                                    &trans      &trans        &trans          &mkp LCLK     &mkp RCLK  &mkp MCLK
            >;
        };

        raise_layer {
            display-name = "Raise";
            bindings = <
&trans  &kp LS(N1)  &kp LS(N2)     &kp LS(N3)    &kp LS(N4)      &kp LS(N5)    &kp LS(KP_PLUS)  &kp N7  &kp N8  &kp N9  &kp PSCRN  &trans
&trans  &kp GRAVE   &kp LS(GRAVE)  &kp LS(BSLH)  &kp LS(N7)      &kp LS(N6)    &kp LS(N8)       &kp N4  &kp N5  &kp N6  &kp MINUS  &trans
&trans  &kp BSLH    &kp C_PREV     &kp LS(C_PP)  &kp LS(C_NEXT)  &bt BT_CLR    &kp N0           &kp N1  &kp N2  &kp N3  &kp DOT    &trans
                                   &kp LS(TAB)   &kp BSPC        &kp DEL       &trans           &trans  &trans
            >;
        };

        mouse_layer {
            display-name = "Mouse";
            bindings = <
&trans  &bt BT_SEL 3  &kp HOME   &kp UP     &kp END    &bt BT_SEL 2    &kp PG_UP       &kp LBKT      &kp RBKT      &none     &kp EQUAL      &trans
&trans  &bt BT_SEL 4  &kp LEFT   &kp DOWN   &kp RIGHT  &bt BT_SEL 1    &kp PG_DN       &kp LS(N9)    &kp LS(N0)    &kp SEMI  &kp LS(SEMI)   &trans
&trans  &kp LALT      &mkp MCLK  &mkp RCLK  &mkp LCLK  &bt BT_SEL 0    &studio_unlock  &kp LS(LBKT)  &kp LS(RBKT)  &none     &kp LS(MINUS)  &trans
                                 &none      &kp LSHFT  &none           &kp RET         &kp SPACE     &trans
            >;
        };
    };
};
```

Notes:
- Lower is held with the left-inner thumb (`Esc`/Lower, pos 38) — the natural way to scroll with the
  right hand on the ball. `&lt LOWER M` also works but occupies a right-hand finger.
- Raise is the right-middle thumb (`Space`/Raise, pos 40); holding it gives the slow snipe pointer.
- Oddities inherited from the daily driver (duplicate F-keys on Lower, `LS(KP_PLUS)`, shifted media
  keys, `&bt BT_CLR` on Raise) are intentional parity — don't "fix" them here.

## Appendix B — `crosses_right.overlay` change (historical; removed in Phase 3b)

Replace the existing `/ { trackball_listener { … }; };` block at the bottom with:

```dts
#include <input/processors.dtsi>
#include <dt-bindings/zmk/input_transform.h>

/ {
    trackball_listener {
        compatible = "zmk,input-listener";
        device = <&trackball>;

        /* Lower (layer 1): vertical-only scroll. Zero X, map Y->WHEEL, invert to match the
         * driver's native scroll direction, then ~ driver speed (tick 24 / CPI divisor 4). */
        lower_vscroll {
            layers = <1>;
            input-processors = <&zip_x_scaler 0 1>,
                               <&zip_xy_to_scroll_mapper>,
                               <&zip_scroll_transform INPUT_TRANSFORM_Y_INVERT>,
                               <&zip_scroll_scaler 1 6>;
        };

        /* Raise (layer 2): horizontal-only scroll. Zero Y, map X->HWHEEL. */
        raise_hscroll {
            layers = <2>;
            input-processors = <&zip_y_scaler 0 1>,
                               <&zip_xy_to_scroll_mapper>,
                               <&zip_scroll_scaler 1 6>;
        };
    };
};
```

Notes:
- Without `process-next`, a matching override's chain replaces the base chain (base is empty anyway).
- Zeroed axes still emit a `0` event (WHEEL or HWHEEL = 0) — harmless.
- The driver keeps applying `INVERT_X` / orientation before ZMK sees the events, so pointer
  orientation on Base is unchanged.

## Appendix C — Test checklist (with the user)

- [ ] Both halves connect; ZMK Studio connects to the right half; Studio unlock works (Mouse layer, N).
- [ ] Base outer keys: `` ` `` / `~` (with Shift), Esc, left Alt, `_`, `'` / `"`, right Alt (Alt-Tab works).
- [ ] Inner block behaves exactly like the daily driver (spot-check each layer against the image).
- [ ] Base: pointer moves at the same speed as before; rolling toward 10:30 moves it straight up
      (rotation feels natural, else adjust `CONFIG_PMW3610_ROTATION_DEG`).
- [ ] Hold Lower: ball scrolls on both axes, direction and speed comfortable.
- [ ] Hold Raise: slow snipe pointer.
- [ ] Hold Mouse: normal pointer speed; mouse buttons work.
- [ ] `;` is on the right home row (pinky), `'` / `"` on the outer column next to it.

## Appendix D — Tuning knobs

| Want | Change |
| --- | --- |
| Rotation | `CONFIG_PMW3610_ROTATION_DEG` in `crosses_right.conf` (−90…90, negative = counterclockwise) |
| Slower / faster scroll | Raise / lower `CONFIG_PMW3610_SCROLL_TICK` (stock 24) |
| Flip scroll direction | `CONFIG_PMW3610_INVERT_SCROLL_Y=y` and/or `CONFIG_PMW3610_INVERT_SCROLL_X=n` (X defaults to `INVERT_X`, which is `y`) |
| Pointer speed | `CONFIG_PMW3610_CPI` / `CONFIG_PMW3610_CPI_DIVIDOR` in `crosses_right.conf` (or host-side pointer speed) |
| Sniper speed | `CONFIG_PMW3610_SNIPE_CPI` / `CONFIG_PMW3610_SNIPE_CPI_DIVIDOR` |
| Choppy pointer | Consider removing `CONFIG_PMW3610_POLLING_RATE_125_SW` (software 125 Hz) — test battery impact |
