# DIY-44: espframe Freenove ESP32-S3 support — handoff

Continuing this on another machine (for board flashing access). This file is
temporary scratch for the handoff and can be deleted once the work merges.

## Where things live

- **Outer repo**: `compton-diy`, branch `diy-44-espframe-nx-submodule` (pushed
  to `origin`, tracks Jira card DIY-44). Contains the NX/pnpm plumbing that
  wires `apps/espframe` in as a git submodule.
- **Submodule**: `apps/espframe` -> fork `rcompton78/espframe`
  (`https://github.com/rcompton78/espframe.git`), branch
  `add-freenove-s3-device`, currently at commit `721fec8` (pushed to the
  fork's `origin`). Upstream is `jtenniswood/espframe` (`main` branch),
  wired as a second remote `upstream` inside the submodule (not tracked by
  either repo's history — see `scripts/espframe-sync-upstream.sh`).

## Getting set up on the new machine

```bash
git clone git@github.com:rcompton78/compton-diy.git
cd compton-diy
git checkout diy-44-espframe-nx-submodule
pnpm i
```

`pnpm i` triggers root `postinstall`
(`git submodule update --init --recursive && nx run-many -t install`), which:
1. Initializes/clones the `apps/espframe` submodule at the pinned commit.
2. Runs the NX `install` target on every project that defines one — today
   that's only `espframe` (`pnpm install --ignore-workspace` inside
   `apps/espframe`, isolated from the root pnpm workspace).

If you ever see `apps/espframe` checked out to an unexpectedly old commit,
it's almost always because `git submodule update --init` resets to whatever
SHA the **outer repo** has recorded — if you're mid-work inside the
submodule, run `git -C apps/espframe checkout add-freenove-s3-device` to get
back to the branch tip, then commit the outer repo's gitlink bump promptly
so this doesn't happen again.

## NX targets available (`apps/espframe/project.json`)

- `pnpm nx run espframe:install` — pnpm install inside the submodule only.
- `pnpm nx run espframe:generate` — runs `pnpm run generate` (regenerates
  `packages.yaml` from `product/contract/devices.json`; does NOT touch
  hand-authored files like screens/fonts/icons).
- `pnpm nx run espframe:build-freenove-s3` — Docker ESPHome compile
  (`ghcr.io/esphome/esphome:2026.6.4`, pinned to match
  `product/contract/project.json`).
- `pnpm nx run espframe:flash-freenove-s3` — Docker ESPHome upload, assumes
  the board enumerates at `/dev/ttyACM0`. **This is the target you'll
  actually run on the new machine** — adjust the device path in
  `project.json` if it enumerates differently there.
- `pnpm nx run espframe:sync-upstream` — merges `upstream/main` into
  `add-freenove-s3-device` and pushes to the fork, then bumps the outer
  repo's gitlink if it changed.

## What's done

- Submodule wiring, NX targets, pnpm workspace exclusion (`!apps/espframe`
  in `pnpm-workspace.yaml`) — all committed and pushed.
- `devices/freenove-s3/packages.yaml` generated via `npm run generate`
  (confirms which includes are required).
- `product/espframe.json` regenerated (freenove-s3 device entry present,
  `esphome_version` corrected to `2026.6.4`).
- Confirmed `product/budgets.json` needs **no per-device entry** — it's a
  single global budget checked against every device in CI's matrix, so
  nothing to add there.

## What's NOT done yet (the actual remaining work)

`devices/freenove-s3/packages.yaml` references several files that don't
exist yet. These are all **hand-authored per device** (confirmed by reading
`scripts/generate_assets.py` — only `packages.yaml` is generated):

- `devices/freenove-s3/device/screen_loading.yaml`
- `devices/freenove-s3/device/screen_wifi_setup.yaml`
- `devices/freenove-s3/device/screen_immich_setup.yaml`
- `devices/freenove-s3/device/screen_slideshow.yaml`
- `devices/freenove-s3/assets/fonts.yaml`
- `devices/freenove-s3/assets/icons.yaml`
- `devices/freenove-s3/dev.yaml` (local dev entry point)
- `devices/freenove-s3/.gitignore`
- `builds/freenove-s3.factory.yaml`
- `builds/freenove-s3.yaml`

Reference implementation for all of these:
`devices/guition-esp32-p4-jc8012p4a1/device/*.yaml` (the P4 board). **Do not
just resize** — the P4 is an 800x1280 portrait display and its slideshow
screen (`screen_slideshow.yaml`) uses a side-by-side dual-portrait-photo
layout; freenove-s3 is 240x320 landscape-capable but should use a single
full-frame photo layout instead. The other three screens (loading, wifi
setup, immich setup) are simpler and mostly need proportional resizing of
widths/fonts (P4 uses e.g. `width: 700`, `noto_400_46_font` — freenove-s3
will need much smaller values to fit 240px width) plus reuse of the same
scripts (`setup_screen_dim`, `finish_startup_screen`, etc. — these live in
shared `common/addon/*.yaml`, not per-device).

Already read in full during the previous session (for reference, no need to
re-read unless verifying): `screen_loading.yaml`, `screen_wifi_setup.yaml`,
`screen_immich_setup.yaml`, `screen_slideshow.yaml` from the P4 device.

### After screens/assets exist

1. `pnpm nx run espframe:build-freenove-s3` — fix compile errors.
2. `npm run check:pr` (or pnpm equivalent) inside `apps/espframe` — fix
   violations.
3. `pnpm nx run espframe:flash-freenove-s3` on the physical board — verify
   WiFi setup, Immich setup, slideshow, touch, backlight end-to-end. This is
   the step that needs the better flashing access on the new machine.
4. Verify actual runtime PSRAM/heap usage on-device.
5. Update the fork's README/docs to note the new supported board.
6. Push any new submodule commits to `origin` (the fork), then bump the
   outer repo's `apps/espframe` gitlink and push `diy-44-espframe-nx-submodule`.
7. Eventually: open a PR from `diy-44-espframe-nx-submodule` into
   `compton-diy` master, and mark Jira DIY-44 done.

## Jira

Card: **DIY-44** (project key `DIY` on `rcompton78.atlassian.net`). DIY-43
was consolidated into DIY-44 and closed — DIY-44 is the single source of
truth for this whole effort.
