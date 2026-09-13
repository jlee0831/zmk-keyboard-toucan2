# Vendored copy

This directory is a vendored, patched copy of
[amgskobo/zmk-input-inertia](https://github.com/amgskobo/zmk-input-inertia),
commit `38b921f9a3c7cc08fae7302ed672b30f59871502` (2026-08-08).

## Why vendored instead of pulled in via west.yml

Upstream targets ZMK `main`'s current endpoints API. This repo pins ZMK to
the stable `v0.3` tag (see `config/west.yml`), which still has the
pre-rename API — pulling the module in as a normal west project fails to
link with:

```
undefined reference to `zmk_endpoint_send_mouse_report'
```

## Patch applied

In `src/input_processor_inertia.c`, all 5 calls to
`zmk_endpoint_send_mouse_report()` were renamed to
`zmk_endpoints_send_mouse_report()` (note the extra "s"), which is the
symbol that actually exists in `zmk/app/include/zmk/endpoints.h` on the
`v0.3` tag. Upstream renamed this function (dropping the "s") at some point
after `v0.3`; no functional change otherwise.

## Re-vendoring

If ZMK core ever gets bumped past the point where `zmk_endpoint_send_mouse_report`
exists (e.g. bumping `zmk` in `config/west.yml` off `v0.3`), re-pull the
module fresh from upstream and drop this patch — at that point it's likely
easier to go back to a normal west.yml project entry pointing at upstream
instead of vendoring.
