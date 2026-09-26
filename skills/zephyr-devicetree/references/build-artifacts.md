# Build Artifacts and the Resolved Devicetree

When to read this: you have (or can produce) a Zephyr build directory and need ground truth
about the final devicetree: what it contains, which overlays were applied, and where boards,
HALs, and bindings live on disk. Everything here needs only a shell and greppable text files.

## How the final devicetree is composed

The build preprocesses and merges these sources, in order:

1. The SoC `.dtsi` chain (under `$ZEPHYR_BASE/dts/` plus vendor HAL dtsi files).
2. The board DTS: `$ZEPHYR_BASE/boards/<vendor>/<board>/<board>.dts`. It includes the board
   pinctrl shell `boards/<vendor>/<board>/<board>-pinctrl.dtsi`, the file that instantiates
   vendor pin data for that board.
3. Every overlay, in apply order.

When two files set the same property on the same node, the later file wins:

```dts
/* board .dts, merged earlier */
&usart1 { status = "disabled"; };

/* app overlay, merged later: this value wins */
&usart1 { status = "okay"; };
```

The effective devicetree is only knowable after a build. Before a build you can enumerate
possibilities (for example, every alternate function a pad could carry) but not what is
actually wired. Label pre-build conclusions as such, and build when the answer matters:

```shell
west build -b <board-target> <app-dir>
```

## Artifact map

Paths are relative to the build directory (commonly `<app>/build`):

| Path | What it is | Availability |
| --- | --- | --- |
| `zephyr/zephyr.dts` | The resolved, merged devicetree. The primary inspection surface. | Every successful configure |
| `zephyr/zephyr.dts.pre` (older trees: `zephyr.dts.pre.tmp`) | Preprocessed intermediate, keeps `#include` linemarkers | Every successful configure |
| `zephyr/include/generated/zephyr/devicetree_generated.h` | The macro form C code sees, plus node ordinals | Zephyr 3.7 or newer; before 3.7 the same file is at `zephyr/include/generated/devicetree_generated.h` |
| `build_info.yml` | Machine-readable build record (board, overlays, include roots) | Zephyr 4.0 or newer |
| `CMakeCache.txt` | CMake cache; metadata fallback for pre-4.0 builds | Every build |
| `domains.yaml` | Marks a sysbuild controller directory, not an application build | Sysbuild only |

If `domains.yaml` exists at the top of the directory you were given, stop: you are looking at
a sysbuild controller, and each image has its own complete build directory underneath. Read
`references/sysbuild.md` first.

Scope note: this file covers human-readable artifacts only. Binary outputs (`zephyr.elf`,
`zephyr.bin`, map files) and Kconfig outputs (`.config`) are not devicetree surfaces and are
not covered here.

## zephyr.dts: the primary inspection surface

`<build-dir>/zephyr/zephyr.dts` is the post-preprocessor, post-merge devicetree in source
form. Grep it before reasoning from any pre-build file:

```shell
# Full node body by label (labels survive into zephyr.dts)
grep -n -A 12 "usart1: " <build-dir>/zephyr/zephyr.dts

# Which node is the console? (chosen keys store resolved paths)
grep -n "zephyr,console" <build-dir>/zephyr/zephyr.dts

# Effective status and properties of a node found by unit address
grep -n -B 2 -A 10 "serial@40013800" <build-dir>/zephyr/zephyr.dts
```

What to expect in the compiled file:

- Labels are preserved, and phandle references stay readable:
  `pinctrl-0 = < &usart1_tx_pa9 &usart1_rx_pa10 >;`.
- Macros are gone. Pinmux cells appear as packed integers, for example
  `pinmux = < 0x127 >;`. For group-style pinctrl vendors (NXP, ESP32, Silabs) grep numeric
  values; for STM32-style direct references grep labels. See `references/pinctrl/model.md`.
- A node with no `status` property is enabled (absent means okay).
- Source-file provenance comments appear only when `dtc` ran with `-A`; do not expect them
  by default.

To recover which files fed the preprocessor (board pinctrl dtsi, vendor macro headers), use
the preprocessed intermediate: it keeps GCC linemarkers of the form `# <line> "<file>"` at
every include boundary, with absolute paths:

```shell
grep -oE '"[^"]*-pinctrl\.(dtsi|h)"' <build-dir>/zephyr/zephyr.dts.pre | sort -u
```

The intermediate's name varies across Zephyr versions (`zephyr.dts.pre` vs
`zephyr.dts.pre.tmp`); check which one your build directory contains.

## Which overlays were applied

Two authoritative records exist. Use either; never guess.

1. The configure log prints one line per consumed overlay (simplest, works on any version
   with a fresh configure):

```shell
west build -b <board-target> <app-dir> 2>&1 | grep "Found devicetree overlay"
# -- Found devicetree overlay: /home/<you>/<app>/boards/<board>.overlay
```

   Gap-fill: message text verified against `$ZEPHYR_BASE/cmake/modules/dts.cmake`
   (`message(STATUS "Found devicetree overlay: ...")`, checked at v3.7.0). Grep loosely if
   the wording drifts in your version.

2. On Zephyr 4.0 or newer, `cmake.devicetree.user-files` in `build_info.yml` is the
   machine-readable record: the resolved overlay list, in apply order (later entries win).

```shell
grep -A 10 "user-files" <build-dir>/build_info.yml
```

Do not re-derive Zephyr's overlay search order by hand (board qualifiers, `FILE_SUFFIX`,
`socs/` variants, legacy short names): the order is version-fragile, and the records above
already tell you what was consumed. For deciding where a new overlay should live, read
`references/overlay-authoring.md`.

## build_info.yml (Zephyr 4.0 or newer)

Written by CMake at configure time; requires Zephyr 4.0.0 or newer. Trimmed example (exact
keys vary slightly across versions, so read the user's own file):

```yaml
cmake:
  application:
    source-dir: /home/me/apps/blinky
  board:
    name: frdm_mcxn947
    qualifiers: mcxn947/cpu0
    path:
      - /home/me/zephyrproject/zephyr/boards/nxp/frdm_mcxn947
  devicetree:
    files:
      - /home/me/zephyrproject/zephyr/boards/nxp/frdm_mcxn947/frdm_mcxn947_mcxn947_cpu0.dts
    user-files:
      - /home/me/apps/blinky/boards/frdm_mcxn947_mcxn947_cpu0.overlay
    bindings-dirs:
      - /home/me/zephyrproject/zephyr/dts/bindings
      - /home/me/zephyrproject/modules/hal/nxp/dts/bindings
    include-dirs:
      - /home/me/zephyrproject/zephyr/dts
      - /home/me/zephyrproject/modules/hal/nxp/dts
  zephyr:
    zephyr-base: /home/me/zephyrproject/zephyr
    version: 4.1.0
west:
  topdir: /home/me/zephyrproject
```

How to use each key:

- `cmake.board.qualifiers`: the first `/`-segment (here `mcxn947`) identifies the silicon
  vendor and family even when the board name is custom. This is the vendor-identification
  input the skill's router uses.
- `cmake.board.name`, `.qualifiers`, `.revision`: the board target split into parts
  (`revision` appears only when the board declares revisions). `cmake.board.path[0]` is the
  board directory.
- `cmake.devicetree.files`: the DTS sources consumed (board DTS first).
- `cmake.devicetree.user-files`: the overlays consumed, in apply order.
- `cmake.devicetree.include-dirs`: every DTS include root CMake configured, including every
  HAL DTS root. Consult this list instead of guessing HAL paths; it is also how you resolve
  the `#include <...>` line an overlay needs (match the header's absolute path against the
  longest prefix here).
- `cmake.devicetree.bindings-dirs`: the binding YAML roots (pair with
  `references/bindings.md`).
- `cmake.application.source-dir` (and `.configuration-dir`): where the app lives.
- `cmake.zephyr.zephyr-base`, `cmake.zephyr.version`, `west.topdir`: the Zephyr tree, its
  version, and the workspace root.
- A sysbuild controller's `build_info.yml` instead carries named `cmake.images[]` entries
  and has no `devicetree` section: see `references/sysbuild.md`.

Quick sanity check that a directory is a real application build:

```shell
grep -E "zephyr-base|source-dir|name:" <build-dir>/build_info.yml
```

## CMakeCache.txt: the pre-4.0 fallback

Builds older than Zephyr 4.0 have no `build_info.yml`. Recover the basics from the CMake
cache instead:

```shell
grep -E "^(BOARD|CACHED_BOARD|ZEPHYR_BASE|APPLICATION_SOURCE_DIR):" <build-dir>/CMakeCache.txt
```

- `ZEPHYR_BASE:PATH=<path>` and `APPLICATION_SOURCE_DIR:PATH=<path>` are cache entries set
  by the Zephyr CMake package (verified at v3.7.0).
- The board target is stored as `CACHED_BOARD:STRING=<board-target>` by the build system; a
  plain `BOARD:...=<board-target>` entry also appears when the value was passed on the
  command line (the normal `west build -b` flow). Gap-fill: which BOARD entries exist
  depends on how the build was configured; check your CMakeCache.txt and prefer
  `CACHED_BOARD` when both are present.

## devicetree_generated.h: what C sees

Gap-fill section (path move verified against the upstream Zephyr 3.7 migration guide).

- Zephyr 3.7 or newer: `<build-dir>/zephyr/include/generated/zephyr/devicetree_generated.h`.
  Before 3.7: `<build-dir>/zephyr/include/generated/devicetree_generated.h`.
- It holds every node's properties as generated `DT_N_...` macros; the `<zephyr/devicetree.h>`
  API resolves to these. Never include the generated header directly from application code.
- Each node has a dependency ordinal. The header records it both in an opening comment block
  (ordinal and node path pairs) and as `_ORD` defines, which is how you decode a
  `__device_dts_ord_<NN>` linker error back to a node:

```shell
grep -nE "_ORD +42$" <build-dir>/zephyr/include/generated/zephyr/devicetree_generated.h
```

  Gap-fill: the exact comment layout varies; check the top of the header in your build. For
  the full triage of that linker error, read `references/dt-from-c.md`; for the general
  error decision tree, `references/debugging.md`.

## West workspace layout: finding HALs and the Zephyr tree

Do not hardcode `<topdir>/modules/hal/<vendor>`. The west manifest (`west.yml` `path:`
entries) controls placement, and real workspaces put the `modules/hal/<vendor>` tail under
any of `<topdir>/`, `<topdir>/deps/`, `<topdir>/deps/modules/`, `<topdir>/workspace/`, or
`<topdir>/workspace/deps/`. The Zephyr repo itself moves the same way.

- On Zephyr 4.0 or newer, skip the guessing entirely: `cmake.devicetree.include-dirs` lists
  every HAL DTS root the build actually used, `cmake.zephyr.zephyr-base` locates the tree,
  and `west.topdir` the workspace root.
- Gap-fill: with west on PATH you can also resolve any module directly (verify the flag
  syntax with `west help list` on your version):

```shell
west list -f "{name} {abspath}" hal_stm32
```

## Board target composition

A board target is `<name>[@<revision>][/<qualifiers>]`, the string `west build -b` accepts
(composition verified against the Zephyr glossary):

```shell
west build -b frdm_mcxn947/mcxn947/cpu0 <app-dir>
west build -b nrf9160dk@0.14.0/nrf9160 <app-dir>
```

`build_info.yml` records the parts separately (`board.name`, `board.revision`,
`board.qualifiers`). In file names (board DTS, auto-picked overlays) each `/` becomes `_`
and the revision is dropped; `references/overlay-authoring.md` covers the overlay naming
rules built on this.

## Out of scope here

- Sysbuild controller layout and image selection: `references/sysbuild.md`.
- Interpreting devicetree build errors and boot failures: `references/debugging.md`.
- Writing overlays and choosing their location: `references/overlay-authoring.md`.
- Consuming the generated macros from C: `references/dt-from-c.md`.
