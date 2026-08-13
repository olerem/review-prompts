# Barebox Review Deltas

Barebox borrows most of its code, style and process from Linux, so the generic
review protocol lives in `../kernel/`. This file records where barebox
**differs**. Where this file contradicts a kernel guide, **this file wins**.

Sources: barebox maintainer/reviewer feedback on the lore list, plus the barebox
tree itself. Each rule below cites how it was established.

---

## 1. NOT a defect in barebox — and how not to invent one

Findings in this section are correct for Linux and wrong for barebox. Suppress
them; do not spend verification effort on them.

### 1.1 Allocation-failure leaks in probe paths

Barebox policy, stated by Sascha Hauer (maintainer):

> "Usually we ignore the memory leaks in the probe paths as there's no real gain
> in freeing a few bytes from a functions error path that is executed only once."

- Do **not** report "this error path leaks `foo`" for `probe()`.
- Do **not** ask for `goto err_free` ladders in probe.
- The idiomatic fix is `xzalloc()`, which cannot return NULL, so the error path
  ceases to exist. Prefer suggesting that over cleanup code.
- Leaks in long-lived or repeatedly-executed paths are still real findings.

### 1.2 `devm_*` does not exist

Barebox has no devres. Absence of `devm_` is not a finding, and "use devm_ to
fix the leak" is not valid advice.

### 1.3 checkpatch `MAINTAINERS need updating?`

**Barebox has no MAINTAINERS file.** This warning fires on every added file and
is always noise. Never turn it into a review comment.

### 1.4 checkpatch "DT compatible string undocumented"

Barebox **does** carry `Documentation/devicetree/bindings/`, but only for its
own extensions — a couple of dozen files, among them `nvmem/barebox,tlv.yaml`,
`nvmem/barebox,environment.yaml`, `barebox/barebox,state.rst` and
`barebox/barebox,deep-probe.rst`. Which way the warning goes depends on who
owns the compatible:

- A Linux compatible (`fsl,imx8mm-usdhc`, `snps,dwmac`) is documented upstream
  and must not be re-documented here. Not a finding.
- A **new `barebox,*` compatible or property** belongs in
  `Documentation/devicetree/bindings/`, next to the ones already there. That
  warning is real; check `git grep -l '<the-compatible>' -- Documentation`
  before deciding.

### 1.5 Concurrency and locking findings

**Barebox is single-threaded and non-preemptive. The locking primitives are
empty macros.** From `include/linux/spinlock.h`:

```c
typedef int   spinlock_t;
#define spin_lock(lock)
#define spin_unlock(lock)
#define spin_lock_irqsave(lock, flags) do { flags = 0; } while (0)
```

`include/linux/mutex.h` is the same — `mutex_lock(lock)` is `((void)0)`.

Consequences for review:

- "Race condition between X and Y", "missing lock", "lock ordering / ABBA
  deadlock", "unlocked access to shared state" are **not** findings. There is no
  second context to race against.
- A `DEFINE_SPINLOCK()` kept in ported code is dead weight by design — it keeps
  the file diffable against Linux. Not a finding.
- Do **not** load `../kernel/subsystem/locking.md`. It generates pure noise here.
- Exception worth checking: code that genuinely runs from an interrupt handler
  or a poller (`poller_call`, `poller_async`). Barebox drivers are overwhelmingly
  poll-based, so confirm such a context actually exists before reporting.

### 1.6 Style deviations inside files imported from Linux

Long lines, unusual table formatting and macro-generated blocks in a file ported
from Linux are intentional: they keep the file diffable against its origin so
future fixes backport cleanly. Reflowing them is a regression, not a cleanup.
Example: `drivers/clk/rockchip/clk-rk3568.c`, imported and tagged with its
Origin-URL, carries some 85 lines past 80 columns — `PNAME()` parent lists and
composite-clock table entries — verbatim from Linux.

### 1.7 Scope creep dressed up as review

Judge a patch by whether it correctly provides the functionality it claims,
not by what a larger version of it might look like. These are not findings:

- **"Move this into a new subdirectory / new Kconfig file / new mach-* dir."**
  If the tree has nothing else in that directory yet, one symbol does not
  justify creating it. It can be refactored when there is a second user.
- **"Rename this file to match a convention"** that the tree does not
  actually follow. Check first: `images/Makefile.*` alone contains company
  names (`rockchip`, `loongson`), vendor product lines (`mvebu`, `socfpga`,
  `exynos`, `at91`) and SoC families (`imx`, `tegra`). There is no single
  rule there to violate.
- **"Add support for X while you are here."** Anything not needed by the
  functionality the patch provides is a separate patch, and usually not the
  submitter's obligation at all.
- **"This will not scale when board number two arrives."** Speculative.

Ask instead: does it work, is it correct, does it break anyone else. If a
structural change is genuinely worth making, say so as an explicit aside and
mark it as not blocking - do not file it as a defect.

### 1.8 Evidence discipline: "it probes" is not "it works"

barebox artifacts show binding, not function. Do not upgrade one into the
other, in either direction - not to justify a finding, and not to dismiss one.

A `clk_dump`, a `devinfo`, or a driver appearing in the boot log proves only
that **probe returned 0** and the driver took its resources. It says nothing
about whether the device transfers data. A host controller happily binds,
enables its clocks and registers with no card, no flash and no link.

What each artifact actually supports:

| Artifact | Proves | Does not prove |
|----------|--------|----------------|
| clk dump shows a consumer at rate X | driver bound, clock enabled | the peripheral works |
| driver in `devinfo` / boot log | probe succeeded | any I/O happened |
| `detect -a` plus `ls /dev/` showing a device node | enumeration worked | reads return correct data |
| a successful `md`/`mtd` read of real content | the path works end to end | nothing beyond that path |

When a patch enables a peripheral, it is fair to ask what evidence exists that
it was exercised - and equally important not to *assert* it works from a
binding artifact. Say precisely which of the rows above is backed by data.

### 1.9 Device tree properties barebox does not implement

barebox implements a **subset** of the Linux DT bindings, so a property can be
correct, upstream, and still do nothing here. Before reporting - or accepting -
a DT property, grep for a reader:

```
git grep -n '"the-property-name"' -- drivers common
```

Two different outcomes, do not conflate them:

- **Describes the hardware, unread today** - e.g. `spi-rx-bus-width`. Harmless
  and correct to keep: it documents the wiring and becomes live when barebox
  gains support. Not a finding.
- **Advertises a capability the driver cannot honour** - e.g. `mmc-hs200-1_8v`
  where the host has no `.execute_tuning` and its `set_ios` ignores
  `ios->timing`. The core switches the card into a mode the controller was
  never put into. That *is* a finding, even when gated behind a config that is
  currently off.

The distinguishing question: does the property cause barebox to *do* something
the driver cannot complete? If it only sits unread, leave it alone.

---

## 2. Barebox conventions to REQUIRE

### 2.1 Imported files must carry `SPDX-Comment: Origin-URL`

Any file taken from Linux gets, on the line directly below the license tag:

```c
// SPDX-License-Identifier: GPL-2.0-or-later
// SPDX-Comment: Origin-URL: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/clk/clk-lan966x.c?id=<full-sha>
```

- `<full-sha>` pins the exact Linux commit the file was taken from, so a later
  re-sync can diff against a known base. A tag name is not enough.
- Some 40 files in the tree already use it
  (`git grep -l 'SPDX-Comment: Origin-URL'`); it is **not** documented anywhere
  in `Documentation/`, so authors routinely miss it.
- Requested by Sascha Hauer.

**Precondition - check before asking for it.** The tag pins an upstream
commit, so it only applies to code that *is* upstream. Verify the content
actually exists in Linux at some sha:

```
git -C <linux> log -S'<distinctive line>' -- <path>
git -C <linux> describe --contains <sha>       # in a tag, or downstream only?
```

If the content is not upstream, there is nothing to pin and **the tag must not
be requested**. Demanding it produces a URL pointing at a tree that does not
contain the code - the exact false confidence the tag exists to prevent. This
happened for real: a reviewer asked for the tag on a barebox-local dtsi whose
nodes are in no upstream tag. See `subsystem/dts.md` for the DT case.

### 2.2 One driver per compatible

Do not dispatch on the compatible inside a single `probe()`:

```c
/* rejected in review */
if (of_device_is_compatible(dev->of_node, "vendor,thing-bank"))
        return thing_bank_probe(dev);
```

Register two `struct driver` instances with separate match tables instead.
Barebox makes this free: `register_driver()` walks devices that already exist,
so a child driver registered *after* the parent's `of_platform_populate()` still
binds the children. Requested by Sascha Hauer.

### 2.3 Per-compatible behaviour belongs in `of_device_id.data`

Attach a per-SoC data struct to **every** entry in the match table. A
`if (!x) x = DEFAULT;` fallback in `probe()` is a review finding — it hides
which SoC gets which value. Requested by Marco Felsch.

### 2.4 Never relax a required resource globally to accommodate one SoC

Turning `clk_get()` into `clk_get_optional()` (or the reset/regulator/gpio
equivalent) because *one* new SoC lacks it silently removes the check for every
SoC that needs it. Gate it on match data instead. Marco Felsch:

> "This commit may introduce bugs in case the clock is not found and you're
> running on a SoC which requires it. This is no longer covered."

### 2.5 Do not restate in the driver what the device tree already says

Barebox honours `assigned-clocks` / `assigned-clock-rates` via
`drivers/clk/clk-conf.c`. A hardcoded `clk_set_rate()` in probe that duplicates
(or silently contradicts) the DT is a finding. Raised by Sascha Hauer.

### 2.6 Kconfig: keep existing defaults simple when adding an arch

When extending an existing driver to a new architecture, widen `depends on` and
leave the default alone. `default y` scoped by `depends on ARCH_A || ARCH_B`
beats `default y if ARCH_A`. Raised by Marco Felsch.

### 2.7 `Fixes:` tags follow one rule, not per-subsystem policy

`../kernel/review-core.md` Task 2.1 decides whether to look for a `Fixes:`
tag from Linux subsystem policy — no check for networking, a check for BPF,
and so on. None of that transfers: barebox is one tree with one maintainer.
Replace that step with:

- A commit that fixes a real bug wants a `Fixes:` tag, wherever in the tree
  it is. Hardening, new features and cleanups do not.
- There is **no `Cc: stable`** in barebox — not one commit in the tree
  carries one. Drop every stable-tag step from `../kernel/fixes-tag.md`.
- The in-tree tag style is mixed: 8 to 12 hex digits, subject quoted or
  bare.

```
Fixes: 2edcc72c (phy: rockchip: naneng-combphy: Add RK3562 support)
Fixes: 33e3bc4782 ("pbl: fix panic() message printing")
```

Report a tag that names the wrong commit. Do not report one that differs
only in sha length or quoting — the tree has no single form to violate
(§1.7).

---

## 3. Linux -> barebox API mapping

Knowing these prevents false "this is broken" findings on ported code.

| Linux | barebox |
|-------|---------|
| `probe(struct platform_device *)` | `probe(struct device *)` |
| `devm_kzalloc()` | `xzalloc()` — cannot fail, no NULL check needed |
| `kcalloc(n, size, GFP)` | `xzalloc(n * size)` — **no `xcalloc()` in barebox proper**; the one in `scripts/kconfig/` is imported host code and is not available to the target |
| `clk_ops.determine_rate` | `clk_ops.round_rate` |
| `module_platform_driver()` | `device_platform_driver()` / `coredevice_platform_driver()` / `postcore_platform_driver()` — the prefix picks the initcall level |
| `MAINTAINERS` entry | nothing — file does not exist |
| `spin_lock()` / `mutex_lock()` | empty macros — single-threaded, see §1.5 |

Other facts worth knowing while reviewing:

- `clk_enable(NULL)` returns 0 (`drivers/clk/clk.c`), so an optional clock left
  as NULL is safe to enable unconditionally.
- `xzalloc` is the dominant idiom: roughly three times as many files under
  `drivers/` use it as use `kzalloc`.
- Device trees: see `subsystem/dts.md` for the `dts/` (Linux-synced, never edit)
  vs `arch/*/dts/` (barebox-specific) split.
- New boards: see `subsystem/board.md` for the lowlevel/board/images/Kconfig
  checklist.
- Driver model specifics: see `subsystem/drivers.md`.
- Networking: barebox has no `sk_buff` and no Linux network stack at all; see
  `subsystem/networking.md` before reviewing anything under `net/` or
  `drivers/net/`.
