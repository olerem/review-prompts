# Device Trees (DTS) in Barebox

## Two locations, two sets of rules

**`dts/`** is an unchanged snapshot of the Linux device trees, updated only by
a wholesale sync. Never edit it by hand.

**`arch/*/dts/`** holds the barebox-specific trees: overlays on the upstream
files, early board support, and barebox-only nodes. The normal form is a small
file that includes the upstream `.dts` and adds what barebox needs — deep
probe, environment, state.

## Review disposition

Getting this wrong is where the noise comes from, so apply it exactly.

### `dts/` (kernel snapshot)

- A patch touching `dts/` as a **standalone fix for one board or SoC** is a
  finding. That fix belongs upstream in Linux first and arrives here with the
  next sync. Never accept a targeted `dts/` edit.
- A large `dts: update to <tag>` sync changing many files is normal and is
  **not** reviewed file by file for board-specific intent.
- Reading `dts/` as *evidence* — which clocks does this SoC declare, what does
  the upstream binding say — is always legitimate, and it is the authoritative
  reference.

### `arch/*/dts/` (barebox-specific)

- **Modifying an existing barebox DT is not a regression.** Do not report it as
  one. The useful question is whether the patch is an opportunity to *shrink*
  the barebox-local delta — the board may be supported upstream by now, so part
  of the local copy could be replaced by including from `dts/`. That is a
  cleanup suggestion, not a defect.
- **Preferred form**: if board and SoC already exist in `dts/`, include the
  upstream file and add only barebox-specific overrides.
- **A patch adding a full new barebox-specific SoC or board DT is the one case
  to question**, and only with one question: is this board or SoC already in
  `dts/`? If it is, carrying a full local copy is wrong. If it genuinely is not
  upstream yet, a complete local `.dts`/`.dtsi` — including the whole SoC
  `.dtsi` — is allowed and correct. Accept it, and note that it should shrink
  to overrides once Linux upstreaming lands.
- **Do not ask for provenance that does not exist.** If the nodes are not
  upstream yet, "these nodes are not upstream yet" is the complete comment.
  There is no commit to pin, so no `SPDX-Comment: Origin-URL:` tag applies
  (`../deltas.md` §2.1), and naming the vendor or downstream tree they came
  from is noise to an upstream reader.

## Scope: board vs SoC

SoC-level peripherals, default IP blocks and memory maps go in the SoC
`.dtsi`. The board `.dts` carries board-specific peripherals (I2C chips,
PHYs), `status = "okay"` overrides and board pinmux. Do not hardcode SoC-wide
detail in a board file.

## Barebox-specific idioms

When reviewing `arch/*/dts/` overrides, expect:

- `barebox,deep-probe;` or `barebox,disable-deep-probe;` in the root node.
- `chosen` additions such as `environment-sd` / `environment-emmc` with
  `compatible = "barebox,environment"` and a `device-path`.
- `state` nodes with `compatible = "barebox,state"` for bootchooser counters
  and other variables that survive a reboot.
- Partition tables with `compatible = "barebox,fixed-partitions"`, usually
  inside the eMMC (`&sdhci`) or SD (`&sdmmc0`) node.

State layouts and partition layouts are frequently project-specific and may
never be upstreamed. That is expected; do not ask for it.

These are barebox's own bindings and they are documented in
`Documentation/devicetree/bindings/barebox/` — a new `barebox,*` property or
compatible should be documented there too (`../deltas.md` §1.4).

## Quick checks

- No manual edits to `dts/`; barebox changes and new boards go to
  `arch/*/dts/`.
- For any new full DT under `arch/*/dts/`, grep `dts/` for the same
  board/SoC compatible before accepting the copy.
- Full `.dts`/`.dtsi` additions for a board not yet in Linux are accepted;
  note the cleanup expectation once upstreaming completes.
