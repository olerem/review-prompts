# Barebox Review Prompts

An **overlay** on the kernel prompt set, not a fork of it.

Barebox shares its process, coding style and a large amount of code with Linux,
so duplicating the kernel prompts here would mean maintaining two copies of the
same protocol and letting them drift. Instead:

- The generic review/debug/coccinelle protocol is read from `../kernel/`.
- This directory carries only what is **different** about barebox.

## Layout

| File | Purpose |
|------|---------|
| `deltas.md` | **The core file.** Where barebox differs from Linux: findings that are not defects here, conventions barebox requires, and the Linux→barebox API mapping. Overrides anything loaded from `../kernel/`. |
| `review-core.md` | Thin entry point: load order and the barebox-specific overrides of the kernel protocol. |
| `false-positive-guide.md` | Barebox exceptions to `../kernel/false-positive-guide.md`. |
| `subsystem/subsystem.md` | Index: barebox-specific guides, which kernel guides to reuse, and which to never load. |
| `subsystem/drivers.md` | Driver model, probe, match tables, clocks. |
| `subsystem/board.md` | New board bringup: lowlevel/board/images/Kconfig. |
| `subsystem/dts.md` | `dts/` (Linux-synced) vs `arch/*/dts/` (barebox). |
| `subsystem/networking.md` | The barebox net stack: `eth_device`, `net_receive()`, DMA, slices, PHY. There is no `sk_buff`. |
| `agent/orc.md` | Overlay on `../kernel/agent/orc.md`: same workflow, plus the lines every spawned agent needs so the subagents see `deltas.md`. |
| `slash-commands/` | `/breview`, `/bseries`, `/bverify`, `/bdebug`, `/bcocci`, `/bslop`, `/borcreview`. |
| `skills/barebox.md` | Auto-loaded skill for barebox trees. |
| `scripts.md` | How to drive `../kernel/scripts/` against a barebox tree. There is no `barebox/scripts/`. |

Everything else comes from `../kernel/` and is referenced by explicit path:
`../kernel/inline-template.md`, `../kernel/callstack.md`,
`../kernel/coccinelle.md`, `../kernel/fixes-tag.md`,
`../kernel/lore-thread.md`, `../kernel/debugging.md`,
`../kernel/technical-patterns.md`, `../kernel/slop-indicators.md`,
`../kernel/agent/*.md` other than `orc.md`,
`../kernel/scripts/*`, and the reusable subsystem guides listed in
`subsystem/subsystem.md`.

A file named by a bare relative path in a kernel prompt resolves against this
directory first, and against `../kernel/` only if it is not here.

Nothing here is a copy of a kernel file. A file that would differ from its
kernel counterpart only by a search-and-replace does not belong in this
directory — reference the kernel file and record the difference instead.

## Install

```
./setup.sh <agent> barebox
```

## Rule of precedence

When a kernel prompt and `deltas.md` disagree, **`deltas.md` wins.** It exists
because the kernel prompts confidently generate findings that barebox
maintainers reject: memory leaks in probe paths, missing locking, MAINTAINERS
updates. Those are settled policy in barebox, not oversights.

## Maintaining this set

When a barebox maintainer rejects or corrects a review finding on the list, that
is a delta — add it to `deltas.md` with the quote and who said it. Do not fix it
by editing a copy of a kernel prompt; that is how the fork started.
