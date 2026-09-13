# Scroll inertia (Mac-trackpad-style momentum scrolling)

## Goal

Make two-finger scroll on the Azoteq trackpad coast/decelerate after you lift
your finger, like macOS's native trackpad momentum scrolling, instead of
stopping dead. This is done in firmware (not via a macOS app like LinearMouse)
so it feels the same on every host OS and every app — see the chat history
for why the macOS-app route (LinearMouse, Mos, etc.) can't really replicate
this for a third-party HID device.

## What's wired in

- **[amgskobo/zmk-input-inertia](https://github.com/amgskobo/zmk-input-inertia)**
  — a ZMK input processor that adds velocity-decay momentum after relative
  input stops. Only decays with configurable start/stop thresholds; doesn't
  touch movement below the "start" threshold, so small deliberate scrolls
  don't randomly coast.
- Added to the **end** of *two* input-processor pipelines in
  [boards/shields/toucan/toucan.dtsi](../boards/shields/toucan/toucan.dtsi)
  — see "Second bug hit" below for why it has to be both, not just one
  (must be last in each — it emits HID reports directly and bypasses
  anything after it):
  ```dts
  trackpad_listener: trackpad_listener {
      ...
      // native two-finger scroll, any layer other than 1/2
      input-processors = <&zip_xy_scaler 100 100>, <&zip_scroll_scaler 1 20>,
                          <&zip_zoom_mapper>, <&is_touching_processor>, <&zip_inertia>;
      scroller {
          // drag-to-scroll while holding layer 1 (SYM) or 2 (NUM)
          layers = <1 2>;
          input-processors = <
              &zip_xy_to_scroll_mapper
              &zip_scroll_scaler 1 20
              &zip_scroll_transform INPUT_TRANSFORM_X_INVERT
              &zip_inertia
          >;
      };
  };
  ```
- Tuning block, also in `toucan.dtsi` (as a top-level `&zip_inertia { ... };`
  fragment *after* the root `/ { ... };` closes — see "gotcha" below):

  | Property | Current value | Meaning |
  |---|---|---|
  | `trigger-ms` | 40 | Delay after input stops before inertia kicks in. Keep ≥ 2x the sensor's report interval or a normal flick gets misread as "stopped". |
  | `move-threshold-start` | 32000 | Effectively disables *cursor* inertia (see "Second bug hit" below for why this matters now). |
  | `scroll-decay-factor-int` | 85 | % velocity retained per report. Higher = coasts further/longer. Keep < 100 (100 never stops, >100 runs away). |
  | `scroll-report-interval-ms` | 65 | How chunky vs. smooth the coast feels. |
  | `scroll-threshold-start` | 1 | Minimum velocity to trigger coasting at all. **Not 2 (the module's default) — see "first bug hit" below.** |
  | `scroll-threshold-stop` | 0 | Velocity floor where coasting ends. |
  | `cancel-scroll-inertia-on-ctrl` | on | Stops/suppresses inertia while Ctrl is held, so leftover momentum can't trigger an accidental Ctrl+wheel zoom. |

  If it coasts too long, lower `scroll-decay-factor-int` (e.g. 80); if it
  stops too abruptly, raise it.
- Real trackpads don't let the cursor glide after you lift your finger —
  only two-finger scroll gets momentum — so cursor inertia is intentionally
  neutralized via `move-threshold-start = 32000` rather than by leaving
  `zip_inertia` out of the base pipeline (it can't be left out — see below).
  To enable cursor inertia too, lower that back down and tune
  `move-decay-factor-int` etc.

## Why the module is vendored, not pulled via `west.yml`

This repo pins ZMK core to the `v0.3` tag
([config/west.yml](../config/west.yml)). Upstream `zmk-input-inertia`
targets ZMK's `main` branch API, which as of 2026-09 has renamed
`zmk_endpoints_send_mouse_report()` (the function that exists on `v0.3`) to
`zmk_endpoint_send_mouse_report()` (singular "endpoint" — used in main).
Pulling the module in as a normal west project compiles fine but fails to
**link**:

```
undefined reference to `zmk_endpoint_send_mouse_report'
```

There's no newer stable ZMK tag to bump to (`v0.3` is current as of writing;
`main` was 189 commits ahead of it, untested drift for a daily-driver
keyboard). So instead of bumping ZMK core, the module is vendored — its
source copied into this repo at
[modules/zmk-input-inertia/](../modules/zmk-input-inertia/) with the
function-name patch applied — and loaded via `ZMK_EXTRA_MODULES` (ZMK's own
CMake variable for a local, non-west-tracked module, *not* Zephyr's generic
`ZEPHYR_EXTRA_MODULES`) added to the `toucan_left`/`toucan_right` entries'
`cmake-args` in [build.yaml](../build.yaml):

```yaml
cmake-args: -DZMK_EXTRA_MODULES=$GITHUB_WORKSPACE\;$GITHUB_WORKSPACE/modules/zmk-input-inertia
```

This repo's root already has its own `zephyr/module.yml` (`board_root: .` — that's
what makes `boards/shields/toucan` discoverable at all), which makes ZMK's
`build-user-config.yml` auto-inject `-DZMK_EXTRA_MODULES=$GITHUB_WORKSPACE`
on every build, *before* `matrix.cmake-args` is appended. CMake takes the
**last** `-D` for a given variable, so a `cmake-args` entry that just sets
`-DZMK_EXTRA_MODULES=.../modules/zmk-input-inertia` on its own silently
clobbers the auto-injected one — and the whole repo (including
`boards/shields/toucan`) stops being discoverable, which fails the build
with `Invalid SHIELD`. That's why both paths are joined into one
semicolon-separated flag above instead of adding a second
`-DZMK_EXTRA_MODULES=` — if this repo ever adds more local modules under
`modules/`, extend that same list rather than adding another flag.

The `;` is escaped as `\;` rather than wrapped in quotes on purpose. A
`build.yaml` value that starts with `"` is parsed by YAML as a
double-quoted scalar, and YAML **strips those outer quotes** during
parsing — they never reach the shell, so an unescaped `;` inside them ends
up unprotected and gets read by bash as a command separator (this actually
happened — `$GITHUB_WORKSPACE/modules/zmk-input-inertia` got executed as
its own command and failed with "Permission denied"). `\;` sidesteps the
issue entirely: it's a literal two-character sequence in any YAML scalar
style, and bash strips the backslash and treats the `;` as a plain
character when building the argument.

Full patch details and the exact vendored commit are in
[modules/zmk-input-inertia/VENDORED.md](../modules/zmk-input-inertia/VENDORED.md).

## What to do when ZMK core gets a new version

Whenever `config/west.yml`'s `zmk` project `revision` moves off `v0.3`
(a newer tag, or `main`):

1. Check whether `zmk_endpoint_send_mouse_report()` (singular "endpoint")
   now exists in the new version's `app/include/zmk/endpoints.h`.
2. **If yes** — the vendored copy is no longer needed. Switch back to a
   normal upstream module:
   - Delete [modules/zmk-input-inertia/](../modules/zmk-input-inertia/).
   - Remove the `-DZMK_EXTRA_MODULES=...` flag from both entries in
     [build.yaml](../build.yaml).
   - Add back to `config/west.yml`:
     ```yaml
     remotes:
       - name: amgskobo
         url-base: https://github.com/amgskobo
     projects:
       - name: zmk-input-inertia
         remote: amgskobo
         revision: main
     ```
   - `boards/shields/toucan/toucan.dtsi` needs no changes either way — the
     `#include <zmk-input-inertia/input/processor/input_inertia.dtsi>` and
     `&zip_inertia { ... };` blocks work identically whether the module is
     vendored or west-fetched.
3. **If no** (endpoints API has moved again, or something else broke) —
   diff [modules/zmk-input-inertia/src/input_processor_inertia.c](../modules/zmk-input-inertia/src/input_processor_inertia.c)
   against the [current upstream file](https://github.com/amgskobo/zmk-input-inertia/blob/main/src/input_processor_inertia.c)
   to see what else changed, and re-patch as needed. Also worth checking
   upstream for bug fixes/improvements since the vendored commit
   periodically, independent of any ZMK version bump.

## First bug hit after a clean build: threshold vs. the `/20` scroll scaler

Firmware built and flashed fine, but inertia produced no perceptible effect
at all. Root cause: `zip_inertia` sits *after* `zip_scroll_scaler 1 20` in
the `scroller` pipeline (it has to — see above), and that scaler divides raw
trackpad deltas by 20, carrying the remainder across reports
(`zip_scroll_scaler` has `track-remainders;` by default in ZMK core). That
makes the scaled value `zip_inertia` actually sees almost always `0` or `1`
per report — reaching `2` requires ~40 raw counts in a *single* report,
which essentially never happens, especially as the finger decelerates right
before lift-off. At the module's default `scroll-threshold-start = 2`,
inertia essentially never armed.

Fix applied: lowered `scroll-threshold-start` to `1` (see table above).

**This is a minimal, untested-on-hardware hypothesis fix** — the theory is
solid (traced through both `zip_scroll_scaler`'s and `zip_inertia`'s actual
source), but confirm on hardware before assuming it's fully resolved. Also
worth knowing going in: even once it arms, the coast is inherently short at
this resolution — a value of `1` decaying at 85%/report only produces ~4
more report intervals (~250ms) of a single-count wheel tick before hitting
the `0` stop threshold. That's a real but subtle nudge, not the multi-second
glide of a real trackpad. If it's armed but still not satisfying, the
likely next lever is the `zip_scroll_scaler 1 20` divisor itself (more
resolution = smoother/longer decay curve), which trades off against the
sensitivity tuning from the "Reduce scroll sensitivity" commit — don't
change that without deciding it's worth revisiting that tradeoff.

If lowering the threshold doesn't fix it, the next diagnostic step is
reading the module's `LOG_DBG("Scroll Inertia triggered...")` line over
RTT/serial to see the actual raw/scaled values in real time — there's
already precedent for this in the `logging` branch ("Add logging to debug
trackpad freeze").

## Second bug hit: `zip_inertia` was in a pipeline that never ran

The threshold fix alone still produced *zero* difference — a strong signal
that the code path wasn't running at all, not just under-triggering. Read
ZMK v0.3's actual `input_listener.c` (`filter_with_input_config`) to check:

A `zmk,input-listener` node's child sub-nodes (like `scroller`, with
`layers = <1 2>`) are **overrides**, not additions. When the current layer
matches one, `apply_config` runs *only* that override's `input-processors`
list, then `return 0` immediately — the top-level (base) list is skipped
entirely (unless the override sets `process-next;`, which `scroller`
doesn't). Conversely, whenever the active layer does *not* match an
override — i.e. whenever you're not holding layer 1 (SYM) or 2 (NUM) — the
**base** list runs instead, unconditionally, for every event including
native two-finger scroll.

`zip_inertia` had only ever been added to the `scroller` override. Ordinary
scrolling — done from the base layer, like nearly all scrolling — went
through the base list the whole time, which never had `zip_inertia` in it.
The threshold-vs-scaling bug above was real, but it was a bug in a pipeline
that wasn't even being exercised.

Fix applied: added `&zip_inertia` to the end of the base `input-processors`
list too (see the wiring above). Because the base list also carries plain
`REL_X`/`REL_Y` cursor-movement events, and `zip_inertia` reacts to move and
scroll independently through the same node, this also required explicitly
neutralizing move inertia (`move-threshold-start = 32000`, effectively
unreachable) so the cursor doesn't start gliding after normal pointer
movement — see the tuning table above.

**Also unverified on hardware as of writing.** If this still doesn't
produce a noticeable effect, the next step is the RTT/serial `LOG_DBG`
route mentioned above, to directly confirm the scroll event even reaches
`zip_inertia` at all — at that point stop tuning blind and get real
runtime values before changing anything else again.

## Gotcha hit while building this (for future `toucan.dtsi` edits)

A `&label { ... };` devicetree override **must** be a top-level fragment,
a sibling of the file's `/ { ... };` root block — not nested inside another
still-open node's braces. Nesting one inside `input_processors { ... }` (or
any other open node) produces a devicetree parse error like:

```
parse error: expected node name, property name, or '}'
```

This is why the `&zip_inertia { ... };` tuning block sits *after* the
closing `};` of the root node at the bottom of `toucan.dtsi`, rather than
next to the other `input_processors` children.
