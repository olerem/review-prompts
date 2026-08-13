# Barebox Board Support

A new board is more than a device tree. A full bringup touches four places, at
three different initialisation stages, and a review should check that all of
them are present and consistent.

## 1. Lowlevel initialisation (PBL)

**Path**: `arch/*/boards/*/lowlevel.c`

Compiled into the uncompressed pre-bootloader. It prepares the SoC to load the
main barebox image. Typical idioms:

- `ENTRY_FUNCTION(...)` or `ENTRY_FUNCTION_WITHSTACK(...)` defines the entry
  point.
- Basic hardware init — SDRAM setup, early IO domain configuration.
- Early debug output with `putc_ll()` or `pbl_set_putc()`.
- Relocation with `relocate_to_current_adr()` or `relocate_to_adr_full()`.
- C environment setup with `setup_c()`.
- A tail call into the SoC entry, passing the device tree blob:
  `imx8mm_barebox_entry(...)`, `rk3568_barebox_entry(...)`.

## 2. Main barebox initialisation

**Path**: `arch/*/boards/*/board.c`

Runs in the full C environment, modelled as an ordinary driver binding to the
board's compatible. Typical idioms:

- Registered with `device_platform_driver()`, `coredevice_platform_driver()`
  or `postcore_platform_driver()`.
- `BAREBOX_DEEP_PROBE_ENABLE(...)` for the board compatible.
- Boot source via `bootsource_get()` and `bootsource_get_instance()`.
- The matching environment partition enabled with `of_device_enable_path()`.
- Barebox update handlers: `imx8m_bbu_internal_mmc_register_handler(...)`,
  `stm32mp_bbu_mmc_fip_register(...)`.

`barebox_set_model()` and `barebox_set_hostname()` do **not** belong here in
the normal case. The OF core already sets them: the model from the root
`model` property (`drivers/of/base.c`, `of_set_root_node()`) and the hostname
from the machine compatible (`of_init_late_vars()`, via
`barebox_set_hostname_no_overwrite()`). Call them only to override the device
tree dynamically — from an ADC or GPIO board ID, say.

## 3. Image generation

**Path**: `images/Makefile.*`

Every new board needs an entry using the architecture's image builder macro
(`build_imx_habv4img`, `build_rockchip_image`, `pblb-y`). Without it nothing
flashable is produced, however complete the rest of the patch is.

Do not demand that the file be renamed to match a convention: `images/`
contains company names, vendor product lines and SoC families side by side
(`deltas.md` §1.7).

## 4. Kconfig and build system

- `arch/*/mach-*/Kconfig` — the `MACH_*` symbol, alongside the other boards of
  that SoC family. If the family directory already exists, that is where the
  symbol goes; if it does not, a single new symbol does not justify creating
  one (`deltas.md` §1.7).
- `arch/*/boards/Makefile` — `obj-$(CONFIG_MACH_...) += board-dir/`.
- `arch/*/dts/Makefile` — build the board device tree.

Additions to Kconfig, Makefiles and `#include` blocks keep alphabetical order.
Group contextually where the file already does, and sort alphabetically within
the group.

## Scope: board vs SoC

Always ask whether a change is really board-specific. Anything that applies to
the whole SoC — a generic clock workaround, CPU feature setup, SoC-wide init —
belongs in `arch/*/mach-*/` or the SoC driver, not in `board.c` or
`lowlevel.c`. Board files stay limited to board-level wiring, environment
setup, BBU handlers and project-specific requirements.

## Quick checks

- `lowlevel.c` present: `ENTRY_FUNCTION`, PBL init, `setup_c()`, correct
  `*_barebox_entry()` tail call.
- `board.c` present: environment selected from `bootsource`, deep probe
  enabled, BBU handlers registered.
- An `images/Makefile*` target actually produces the bootable image.
- `MACH_*` symbol, `boards/Makefile` and `dts/Makefile` all hooked up.
- Alphabetical order maintained in each of them.
