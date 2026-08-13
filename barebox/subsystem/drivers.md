# Barebox Driver Model

Applies to anything under `drivers/`. Barebox's driver model looks like Linux'
but is smaller and differs in ways that generate false findings if assumed
identical. Read together with `../deltas.md`.

## Registration and initcall levels

A driver is registered by one of:

| Macro | When it probes | Typical use |
|-------|----------------|-------------|
| `postcore_platform_driver()` | earliest | clock controllers, pinctrl |
| `coredevice_platform_driver()` | early | GPIO, reset, watchdog |
| `device_platform_driver()` | normal | MCI, SPI, net, most peripherals |

Review checks:

- A provider must register no later than its consumers. A clock controller at
  `device_platform_driver()` with consumers at `coredevice_` is a real ordering
  bug.
- `register_driver()` iterates over devices that already exist, so registration
  order within one initcall level does **not** have to match probe order. A
  parent that creates children via `of_platform_populate()` may be registered
  before the child driver; the children bind when the child driver registers.

## probe()

- Signature is `int probe(struct device *dev)` — not `platform_device`.
- Allocation: `xzalloc()`. It cannot fail, so no NULL check and no unwind. See
  `../deltas.md` §1.1 before reporting any probe-path leak.
- Per-compatible data comes from `device_get_match_data(dev)`. Every entry in
  the match table should carry `.data`; a default fallback in probe is a
  finding (`../deltas.md` §2.3).
- Resource lookups (`clk_get`, `reset_control_get`, `dev_request_mem_resource`)
  that are mandatory on some SoCs must not be blanket-converted to their
  `_optional` variant (`../deltas.md` §2.4).

## Match tables

- One `struct driver` per compatible family; do not branch on the compatible
  inside probe (`../deltas.md` §2.2).
- Keep entries in the same order and spelling as Linux where the driver is a
  port, so the tables stay diffable.

## Clocks

- `clk_ops.round_rate` replaces Linux' `.determine_rate`. A port that keeps
  `.determine_rate` will silently not be called — a real finding.
- `clk_enable(NULL)` is a no-op returning 0.
- Rates that the device tree already sets via `assigned-clock-rates` should not
  be re-applied with `clk_set_rate()` in probe (`../deltas.md` §2.5).
- barebox's clk core does not resolve `clk_parent_data.fw_name` through
  `clock-names`; ports that rely on it must also set `.name`.

## Ported drivers

- Require the `SPDX-Comment: Origin-URL` tag — but only once you have confirmed
  the code really is upstream. `../deltas.md` §2.1 has the precondition and the
  commands; do not ask for a tag that would pin a tree without the code in it.
- Do not demand style fixes that increase the delta against the Linux original
  (`../deltas.md` §1.6).
- Do call out *semantic* divergence from the original: dropped error handling,
  changed register sequences, altered masks. Those are the real risks in a port,
  and they are what review time should be spent on.
