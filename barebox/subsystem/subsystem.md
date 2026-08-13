# Barebox Subsystem Guide Index

Load subsystem guides based on what the code touches. Guides listed as
`../../kernel/subsystem/<file>` are used unchanged from the kernel prompt set;
guides without a path prefix are barebox-specific and live next to this file.

`../deltas.md` applies to every guide below and overrides it on conflict.

## Barebox-specific guides

| Subsystem | Triggers | File |
|-----------|----------|------|
| Driver model | drivers/, `probe(`, `device_platform_driver`, `coredevice_platform_driver`, `postcore_platform_driver`, `of_device_id`, `device_get_match_data`, `clk_`, `reset_control_` | drivers.md |
| Board support | arch/*/boards/, images/Makefile*, lowlevel.c, board.c, `ENTRY_FUNCTION`, `barebox_entry`, bbu handlers | board.md |
| Device trees | arch/*/dts/, dts/, *.dts, *.dtsi, `barebox,deep-probe`, `barebox,environment`, `barebox,state` | dts.md |
| Networking | net/, drivers/net/, `eth_device`, `edev`, phy, mdio | networking.md |

## Reused kernel guides

Relevant because barebox carries the corresponding code:

| Subsystem | Triggers | File |
|-----------|----------|------|
| Open Firmware (DT) | drivers/of/, `of_node`, `of_find_`, `of_get_`, `of_parse_`, `for_each_child_of_node`, `of_node_put` | ../../kernel/subsystem/of.md |
| I/O accessors | `writel`, `readl`, `__raw_writel`, `writesl`, `readsl`, FIFO | ../../kernel/subsystem/io-accessors.md |
| Kconfig | Kconfig, `config `, `select `, `depends on `, `bool ` | ../../kernel/subsystem/kconfig.md |
| Build system | Makefile, `obj-$(CONFIG_`, `KBUILD_`, `pblb-y` | ../../kernel/subsystem/build.md |
| Cleanup API | `__free`, `guard(`, `scoped_guard`, `DEFINE_FREE`, `no_free_ptr` | ../../kernel/subsystem/cleanup.md |
| Alignment helpers | `ALIGN(`, `ALIGN_DOWN(`, `IS_ALIGNED(`, `PAGE_ALIGN` | ../../kernel/subsystem/alignment.md |
| Block/NVMe | common/block.c, drivers/block/, drivers/nvme/ | ../../kernel/subsystem/block.md |
| VFS | fs/, `inode`, `dentry`, `struct fs_driver` | ../../kernel/subsystem/vfs.md |
| USB storage | drivers/usb/storage/ | ../../kernel/subsystem/usb-storage.md |
| ATA | drivers/ata/ | ../../kernel/subsystem/ata.md |
| PCI | drivers/pci/ | ../../kernel/subsystem/pci.md |
| I2C | drivers/i2c/, `i2c_transfer`, `struct i2c_adapter` | ../../kernel/subsystem/i2c.md |
| Input | drivers/input/, `input_report_`, keymaps | ../../kernel/subsystem/input.md |
| Console/TTY | drivers/serial/, `console_device`, `putc`, `getchar` — the framing and termios parts only | ../../kernel/subsystem/tty.md |
| ARM64 | `CONFIG_CPU_64`, `arch/arm/cpu/`, exception levels, cache maintenance | ../../kernel/subsystem/arm64.md |
| MIPS | arch/mips/ | ../../kernel/subsystem/mips.md |

barebox has no `arch/arm64/`; 64-bit ARM lives under `arch/arm/` behind
`CONFIG_CPU_64`, so `arm64.md` is triggered by the code, not by the path.

## Core code: `common/`, `lib/`, `crypto/`, `fs/`

No barebox-specific guide covers these yet, and every trigger table above is
driver, board, dts or network shaped — so a patch to `common/tlv/`,
`common/image-fit.c`, `lib/` or `crypto/` matches nothing and the review runs
on `../deltas.md` plus the generic protocol alone. That is the correct
fallback, but load these by trigger first:

| Trigger | File |
|---------|------|
| fs/, `struct fs_driver`, `inode`, `dentry` | ../../kernel/subsystem/vfs.md |
| common/block.c, drivers/block/ | ../../kernel/subsystem/block.md |
| `__free`, `guard(`, `DEFINE_FREE` | ../../kernel/subsystem/cleanup.md |
| `ALIGN(`, `IS_ALIGNED(` | ../../kernel/subsystem/alignment.md |

There is no kernel guide for `crypto/`, and **none for packed on-media
formats** — `alignment.md` is about `ALIGN()` and pageblock arithmetic, not
about unaligned access, so do not load it for a `__packed` struct. Review
both from the generic protocol, knowing:

- On-media and on-wire structs here are `__packed`, so reading a member
  directly is safe: the compiler knows the member's alignment is 1. The
  `get_unaligned_be*()` spelling of the same access is the older idiom and
  both appear in the same files, `common/tlv/` among them. Neither spelling
  is a finding.
- What is worth checking is a cast of a `void *` buffer to a struct that is
  **not** packed, and any access indexed past the length the format declared.

What actually goes wrong here, and what the driver-shaped guides do not
prompt for:

- **A length or offset that came from storage and is used to bound a read.**
  barebox parses TLV, FIT, device trees, filesystems and boot images off
  media the user can rewrite. Follow every declared length through to the
  buffer it indexes, and check who validated it.
- **`size_add()` / `check_add_overflow()`** (`include/linux/overflow.h`)
  return `SIZE_MAX` on overflow. A caller that does not test for `SIZE_MAX`
  has an unchecked length. In use in `common/resource.c`,
  `include/tlv/format.h`, `fs/cramfs/`, `fs/ext4/`, `fs/jffs2/` and
  `fs/pstore/`.
- **Validation split across layers.** One function bounds the blob, another
  derives a pointer from a second field of the same header. Check that every
  field the later layer trusts was covered by the earlier check — a bound on
  the total length says nothing about the fields inside it.
- **`../deltas.md` §1.1 does not apply here.** It covers probe and one-shot
  registration. A parser that runs per file, per boot entry or per received
  packet is neither; leaks and error paths there are real findings.

When reusing a kernel guide, discount anything that assumes SMP, preemption,
sleeping contexts, devres, sysfs or a MAINTAINERS file — see `../deltas.md`.

## Do NOT load

These kernel guides describe subsystems barebox does not have. Loading them
wastes context and invites invented findings:

`locking.md` (see ../deltas.md §1.5), `rcu.md`, `scheduler.md`, `workqueue.md`,
`tracing.md`, `timers.md`, `bpf.md`, `btf.md`, `libbpf.md`, `io_uring.md`,
`syscall.md`, `sysfs.md`, `perf.md`, `selftests.md`, `objtool.md`, `kho.md`,
`dax.md`, `cxl.md`, `mm-alloc.md`, `mm-folio.md`, `mm-largepage.md`,
`mm-pagetable.md`, `mm-reclaim.md`, `mm-vma.md`, `fscrypt.md`, `btrfs.md`,
`nfsd.md`, `sunrpc.md`, `smb-ksmbd.md`, `wireless.md`, `bluetooth.md`,
`drm.md`, `pm.md`, `pmdomain.md`, `hid.md`, `rust.md`, `netlink.md`,
`kvm.md`, `kvm-arm64.md`, `hyp-arm64.md`.

Also never load `networking-core.md` or `networking-drivers.md`: barebox has no
`sk_buff` and no Linux network stack. Use `networking.md` above.

`dt-bindings.md` describes the upstream binding-review process, which barebox
patches do not go through — but barebox does carry bindings for its own
`barebox,*` extensions, so read `../deltas.md` §1.4 rather than treating every
binding comment as noise.

`subsystem-template.md` is for authoring new guides, not for review.

## Optional Patterns

Load only when explicitly requested in the prompt:

- **Subjective Review** (`../../kernel/subsystem/subjective-review.md`): subjective
  general assessment. Paired with `../../kernel/slop-indicators.md`, which
  `/bslop` runs standalone.
