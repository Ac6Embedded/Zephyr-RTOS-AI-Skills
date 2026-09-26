---
name: zephyr-kconfig
description: Zephyr RTOS Kconfig and software configuration work. Use when editing or creating prj.conf or any .conf Kconfig fragment; setting, searching, or debugging CONFIG_ options; the build reports "was assigned the value 'y' but got the value 'n'" or "Aborting due to Kconfig warnings"; deciding what belongs in a fragment (debug.conf, prj_release.conf, EXTRA_CONF_FILE); inspecting or explaining build/zephyr/.config; running menuconfig, guiconfig, or hardenconfig; understanding config merge order (board defconfig, CONF_FILE, shields, EXTRA_CONF_FILE); writing Kconfig files for an application or module (config, choice, depends on, select, imply, default); using CONFIG_ symbols from C code (#ifdef, IS_ENABLED, autoconf.h); or configuring sysbuild images (SB_CONFIG, sysbuild/mcuboot.conf).
---

# Zephyr Kconfig

Help the developer control Zephyr's software configuration through Kconfig. Ground every answer in the user's own files: prj.conf, fragments, the Kconfig tree, and (when a build exists) `build/zephyr/.config`.

## Core model

- The effective configuration is `build/zephyr/.config`: the merge of board defconfig, prj.conf, and every fragment, in a fixed order where later files win. It is only knowable after a configure; label pre-build conclusions as such.
- Zephyr uses no tristate: every bool symbol is `y` or `n`, `m` never appears.
- Each symbol has three distinct states: visibility (can it be shown and edited), value (always resolved, even for invisible symbols, via defaults and select), and assignability (which values can be set right now). Every "symbol won't set" case is one of these failing.
- `depends on` gates visibility; `select` forces its target to `y` while ignoring the target's own dependencies; `imply` is a weak default the user can still override. Defaults are ordered (value, condition) pairs and the first satisfied one wins.
- Promptless symbols can never be set by users or fragments; you change whatever drives them instead.
- menuconfig edits the already-merged `build/zephyr/.config` directly. Those edits are temporary experiments: they survive incremental builds but are lost on a pristine build or whenever any fragment or Kconfig file changes.
- Some defaults are computed from the resolved devicetree (`dt_*` preprocessor functions, generated `DT_HAS_*` symbols); they move with overlays and board files, never with `.conf` edits.
- Unsetting a symbol (removing its line so it falls back to its default) is different from setting it to `n`.

## First moves

1. Locate the build directory (commonly `<app>/build/`). If it contains `domains.yaml` it is a sysbuild controller: there is no top-level `zephyr/.config`, each image has its own at `<build>/<image>/zephyr/.config`; see `references/sysbuild.md`.
2. If built: `grep "CONFIG_<X>[=\" ]" build/zephyr/.config`, also matching `# CONFIG_<X> is not set` (which is a real value, not a comment). That file is the effective truth.
3. To explain a value: open the symbol's info in `west build -t menuconfig`, then grep the fragments in merge order from `build_info.yml` key `cmake.kconfig.files` (Zephyr 4.0+); see `references/debugging.md`.
4. If not built or the symbol is unknown: search name and prompt (`references/finding-options.md`), then configure, because defaults, ranges, and DT gating only resolve at configure time.

## Routing

| Situation | Read |
| --- | --- |
| Writing any `.conf` line: value formats, comments, `is not set`, duplicates | `references/conf-syntax.md` |
| Which files feed `.config`, in what order; registering a fragment; profiles | `references/merge-order.md` |
| Creating a fragment; deciding what goes in prj.conf vs a fragment; minimal configs | `references/writing-fragments.md` |
| Finding which CONFIG_ option controls a behavior; searching; reading symbol info | `references/finding-options.md` |
| What depends on / select / imply / default / range / choice actually do | `references/symbols-and-dependencies.md` |
| Interactive editing; persisting menuconfig changes; hardenconfig; size reports | `references/menuconfig-guiconfig.md` |
| "assigned 'y' but got 'n'"; unknown symbol; symbol won't set; why does X have this value | `references/debugging.md` |
| Adding your own symbols to an app or module; Kconfig file syntax | `references/kconfig-authoring.md` |
| Using CONFIG_ from C or CMake; IS_ENABLED; symbol absent in code | `references/config-from-c.md` |
| Build dir has `domains.yaml`; MCUboot or other image config; SB_CONFIG | `references/sysbuild.md` |
| `.config`, `sources.txt`, `build_info.yml` kconfig keys; is `.config` stale | `references/build-artifacts.md` |

## Non-negotiable rules

- Never persist configuration by editing `build/zephyr/.config`: the next pristine build (or any fragment/Kconfig change) regenerates it and discards the edit. Durable changes go in prj.conf or a registered fragment.
- `# CONFIG_X is not set` is an assignment of `n`, not a comment. Last assignment wins, within one file and across fragments in merge order.
- Never assign a promptless symbol in a `.conf` file: the build aborts. Set the prompted symbol (or devicetree node) that drives it instead.
- A symbol forced `y` by `select` cannot be turned off from any `.conf` file; find and disable the selector.
- Error severity differs: an unknown CONFIG_ name or a promptless assignment ABORTS the build ("Aborting due to Kconfig warnings"), but an unmet-dependency assignment only WARNS and the feature silently stays off. Always read the configure output.
- Follow generated-file conventions in fragments: bools as `=y` or `# CONFIG_X is not set` (never `=n`, although it parses), strings double-quoted, hex with `0x`, int in decimal.
- A new fragment file has no effect until it is registered in the build (`EXTRA_CONF_FILE`, or an automatically picked-up name like `boards/<BOARD>.conf`); creating the file is not enough.
- When a value comes from a devicetree-derived default (`DT_HAS_*` gating), fix it in the overlay (zephyr-devicetree skill), not in a `.conf` file.
