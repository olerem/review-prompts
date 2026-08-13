# Barebox False Positive Guide

Overlay on `../kernel/false-positive-guide.md`. Load that file first and apply
its full verification discipline, then apply the barebox exceptions below.

## Barebox exceptions

The kernel guide's resource-leak sections ("Resource Leak Misconceptions", the
alloc-to-free ownership chain protocol, and the "ask as the author" prompts for
leaks) are correct for Linux and **wrong for barebox probe paths**.

In barebox:

- An allocation-failure path in `probe()` that does not free earlier
  allocations is **not** a finding. It is stated project policy — see
  `deltas.md` §1.1 for the maintainer's wording. Do not build an ownership
  chain for it, do not report it, do not list it as "minor".
- The absence of `devm_*` is not a leak and not a finding — barebox has no
  devres.
- checkpatch `MAINTAINERS need updating?` is structural noise in barebox, never
  a finding (`deltas.md` §1.3).
- checkpatch's undocumented-DT-compatible warning is noise for Linux
  compatibles, which are documented upstream — but barebox does carry bindings
  for its own `barebox,*` extensions, so read `deltas.md` §1.4 before
  discarding it.
- Style deviations inside files imported from Linux are deliberate
  (`deltas.md` §1.6). Reflowing them increases the delta against the origin.

Leaks outside probe, in paths that run repeatedly or in long-lived allocations,
remain real findings and are verified exactly as the kernel guide describes.

## Barebox-specific false positive: assumed Linux API

Before reporting that ported code "drops" or "breaks" something, check the
Linux -> barebox API mapping in `deltas.md` §3. Common mistakes:

- Reporting a missing NULL check after `xzalloc()` — it cannot fail.
- Reporting `xcalloc()` should be used — it does not exist in barebox.
- Reporting a missing `.determine_rate` — barebox uses `.round_rate`.
- Reporting `clk_enable()` on a possibly-NULL clock — it returns 0 for NULL.
