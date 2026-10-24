# Agent Instructions for zmk-for-crosses36

Single, tool-agnostic instructions file (Copilot CLI, Codex, Gemini CLI and
Claude Code all read `AGENTS.md`). Don't create `CLAUDE.md`/`GEMINI.md`.

## Context

ZMK firmware config for @vergilcw's **secondary** keyboard: a wireless
**Crosses 36 — 3x6+3, single trackball on the right half** (MIAOMIAO /
HeeTuic modified PCB, two nice!nano v2, one PMW3610 sensor).

- Fork of [HeeTuic/zmk-for-crosses](https://github.com/HeeTuic/zmk-for-crosses),
  **branch `36`** (that repo uses one branch per variant: `35`, `36`, `46`, `46_RGB`;
  the default branch upstream is `35` — do not work on it).
- Sister repo: [vergilcw/zmk-for-crosses35-dual](https://github.com/vergilcw/zmk-for-crosses35-dual)
  is the user's **daily driver** (3x5 dual trackball). Its `config/crosses.keymap` is the
  source of the inner 3x5+3 keymap here. Read it, don't modify it from this session.
- The current task and its full spec live in [`docs/plan.md`](docs/plan.md). Read it first.

## How this repo differs from the daily driver's repo (important)

| | This repo (3x6 single) | Daily driver (3x5 dual) |
| --- | --- | --- |
| ZMK | `v0.3` branch (Zephyr 3.5) | pinned commit on `main` (Zephyr 4.1) |
| Shield definition | **in-repo** `config/boards/shields/crosses/` (edit directly) | external module `HeeTuic/cb-zmk-keebs` |
| Board name in `build.yaml` | `nice_nano_v2` | `nice_nano` |
| Trackball driver | **vendored** `modules/zmk-pmw3610-driver` (from `DoctorWangWang/zmk-pmw3610-driver@1d9c2c6`, loaded via root `zephyr/module.yml`, patched with `CONFIG_PMW3610_ROTATION_DEG`), compatible `pixart,pmw3610`, Kconfig-tuned (`CONFIG_PMW3610_*`), DT props `scroll-layers`, `snipe-layers`, `automouse-layer` | `efogdev` driver |
| Physical layout | `crosses_layout` (42 keys) set in `crosses.dtsi` `chosen` — the keymap must **not** set `zmk,physical-layout` | keymap chooses `gggw_crosses_36_layout` |
| keymap-drawer layout | `config/crosses.json` (auto-picked by the draw workflow), output to `img/` | `config/info.json`, output to `keymap-drawer/` |
| Keyboard name | `Crosses 36` (`config/crosses.conf`) | `Crosses BT` |

Key positions: 12 per row × 3 rows = 0–35, thumbs 36–41. Row r, column c (0–11) → `12*r + c`.
Left half = columns 0–5 + thumbs 36–38; right half = columns 6–11 + thumbs 39–41.
Layer indices: Base 0, Lower 1, Raise 2, Mouse 3 (keep these; the overlay refers to them by number).

## Approval boundaries (hard rules)

Ask for an explicit in-conversation go-ahead before:

- **Flashing** either half, or anything that enters the bootloader / copies a `.uf2`.
- **Writing to the keyboard over ZMK Studio RPC** (set/save/discard/reset). Read-only
  dumps are fine, but the user must press `&studio_unlock` themselves — ask them to.
- Pushing to any remote, opening PRs, changing repo settings.
- Destructive git (force-push, history rewrite, `reset --hard` on unpushed work).
- Touching the daily-driver repo or the daily-driver keyboard (`Crosses BT`) in any way.
- Changing the user's system (NixOS) config beyond proposing a diff.

Safe without asking: editing files in this repo, local keymap-drawer renders, `gh`
read-only queries, reading the sister repo.

## Working conventions

- **Paths:** absolute paths inside the repo checkout or `/tmp`. Never use `..` in paths
  (the user's permission policy rejects it). Keep caches in the repo's gitignored `.cache/`.
- **Shell:** the user's shell is fish. Commands you suggest for them must be fish-compatible
  (`set -x VAR value`; use `env VAR=value cmd`, never `VAR=value cmd`).
- **Tooling (NixOS):** `uv`, `python3`, `keymap-drawer`, `jq` may be installed system-wide; if not,
  use `nix shell nixpkgs#<pkg> -c …`. Don't wrap `nix` itself in a custom `XDG_CACHE_HOME`.
  uv-managed Pythons don't run on NixOS: use `env UV_PYTHON_DOWNLOADS=never uv run …`.
- **Network quirk:** `raw.githubusercontent.com` is unreachable from agent sessions on the dev
  machine; `gh api` works (use it to fetch files, or to seed keymap-drawer glyphs — see plan).
- **Builds:** GitHub Actions is the build path (`build.yml`, ZMK `build-user-config.yml@v0.3.0`).
  Local docker builds are possible but the root filesystem is tight on space.
- **Keymap edits:** keep the 12-column grid aligned, use the `BASE/LOWER/RAISE/MOUSE` defines,
  re-render with keymap-drawer after changes.
- **ZMK docs:** target the v0.3 era. Input processors, input-listener layer overrides,
  `zip_x_scaler`/`zip_y_scaler`/`zip_scroll_scaler`, `zip_xy_to_scroll_mapper` and
  `zip_scroll_transform` all exist at v0.3 (verified in ZMK `v0.3` branch source).
- **Studio caveat:** keymap edits saved via ZMK Studio live in flash and override the compiled
  keymap even after re-flashing. After flashing, run *Restore Stock Settings* in Studio.
- **Privacy:** the repo is public. Don't commit hardware serial numbers, hostnames, home paths,
  or personal emails. Studio dumps must have the serial removed.
- **Commits:** small, descriptive; include the Copilot co-author trailer.

## Documentation stays in sync

- Check items off in `docs/plan.md` as soon as they're done; put newly discovered work in its
  Inbox section.
- Record verified findings (with date + evidence) and decisions in `docs/knowledge-base.md`
  (create it on first use). Mark guesses *unverified*.
- Update `README.md` when layout or features change.
