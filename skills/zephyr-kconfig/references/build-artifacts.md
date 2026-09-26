# Kconfig Build Artifacts

When to read this: you have (or can produce) a Zephyr build directory and need ground truth
about the effective configuration: what every CONFIG_ symbol resolved to, which fragments
were merged and in what order, and whether what you are reading is current. Everything here
needs only a shell and greppable text files.

## Artifact map

Paths are relative to the build directory (commonly `<app>/build`):

| Path | What it is | Availability |
| --- | --- | --- |
| `zephyr/.config` | The merged, authoritative configuration. The primary inspection surface. | Every successful configure |
| `zephyr/kconfig/sources.txt` | Every Kconfig file the build parsed, absolute paths, one per line | Every successful configure |
| `zephyr/include/generated/zephyr/autoconf.h` | The macro form C code sees | Zephyr 3.7 or newer; before 3.7 the same file is at `zephyr/include/generated/autoconf.h` |
| `build_info.yml` | Machine-readable build record; `cmake.kconfig.*` keys list the merged fragments | Zephyr 4.0 or newer |
| `Kconfig/Kconfig.dts` | Generated devicetree-gated symbols (`DT_HAS_<COMPAT>_ENABLED`, one per binding compatible) | Every successful configure |
| `CMakeCache.txt` | CMake cache; keeps `-DCONFIG_X`, `CONF_FILE`, and `EXTRA_CONF_FILE` values across rebuilds | Every build |
| `domains.yaml` | Marks a sysbuild controller directory, not an application build | Sysbuild only |

If `domains.yaml` exists at the top of the directory you were given, stop: you are looking
at a sysbuild controller, and it has no `zephyr/.config` of its own. Each image is a
complete build tree at `<build>/<image>/` with its own `.config` and `build_info.yml`. Read
`references/sysbuild.md` first.

Scope note: this file covers configuration artifacts only. Devicetree artifacts
(`zephyr.dts`, generated devicetree headers) belong to the zephyr-devicetree skill; binary
outputs (`zephyr.elf`, map files) are not configuration surfaces.

## zephyr/.config: the effective configuration

`<build-dir>/zephyr/.config` is the result of the full merge (board defconfig, prj.conf,
every registered fragment, in the order `references/merge-order.md` describes). It is the
only file that tells you what a symbol actually resolved to. Grep it before reasoning from
prj.conf or any fragment:

```shell
# One symbol, matching both value forms: CONFIG_LOG=y and "# CONFIG_LOG is not set"
grep -E "CONFIG_LOG(=| )" <build-dir>/zephyr/.config

# Everything under a prefix (subsystem sweep)
grep "CONFIG_SHELL" <build-dir>/zephyr/.config
```

What to expect in the file:

- Symbols appear in Kconfig tree order (the same order menuconfig displays), each exactly
  once: merge-time duplicates are already resolved, last assignment wins.
- Menu banners appear as comment blocks (`#`, `# <menu prompt>`, `#`) and menus close with
  `# end of <menu prompt>`.
- Bool `y` is `CONFIG_X=y`; bool `n` is `# CONFIG_X is not set`. That line is an assignment
  of `n`, not a comment, and generated files never write `CONFIG_X=n`. Int is decimal, hex
  keeps its `0x`, strings are double-quoted. Full grammar: `references/conf-syntax.md`.
- A symbol absent from the file entirely was not part of this build's Kconfig tree at all
  (module not in the manifest, or the defining Kconfig file was never sourced).

Never persist a change by editing this file. Hand-editing is legal for a quick experiment,
but it is dependency-unaware: an assignment that violates dependencies is silently dropped
the next time the build reprocesses the configuration. menuconfig and guiconfig also write
here directly (`references/menuconfig-guiconfig.md`); durable changes go in prj.conf or a
registered fragment (`references/writing-fragments.md`).

## zephyr/kconfig/sources.txt: what the tree was built from

Every Kconfig file the configure stage parsed, as absolute paths, one per line. Use it to
find a symbol's definition site without guessing directory layouts, and to check whether a
module's Kconfig entered the tree at all:

```shell
# Where is this symbol defined? Search only files the build actually parsed
xargs grep -lE "^(menu)?config LOG_BUFFER_SIZE$" < <build-dir>/zephyr/kconfig/sources.txt

# Did my module's Kconfig get sourced?
grep "<module-name>" <build-dir>/zephyr/kconfig/sources.txt
```

For search strategy (menuconfig `/`, docs URLs, grep patterns) read
`references/finding-options.md`.

## Which fragments were merged

Two authoritative records exist. Use either; never re-derive the merge list by hand.

On Zephyr 4.0 or newer, `build_info.yml` at the top of the build directory records the
merge under `cmake.kconfig`:

```yaml
cmake:
  kconfig:
    files:
      - /home/me/zephyrproject/zephyr/boards/nordic/nrf52840dk/nrf52840dk_nrf52840_defconfig
      - /home/me/apps/blinky/prj.conf
      - /home/me/apps/blinky/debug.conf
    user-files:
      - /home/me/apps/blinky/prj.conf
    extra-user-files:
      - /home/me/apps/blinky/debug.conf
```

- `cmake.kconfig.files`: the full ordered merge list, board defconfig first, later entries
  win. This is the list to grep, in order, when tracing where a value came from
  (`references/debugging.md`).
- `cmake.kconfig.user-files`: the `CONF_FILE` slot contents (prj.conf when auto-picked).
- `cmake.kconfig.extra-user-files`: the `EXTRA_CONF_FILE` entries.

Caveat: `build_info.yml` is only rewritten when `.config` is actually regenerated. After a
configure that did not re-merge (nothing changed), the file simply still matches; but never
read it from a half-configured or failed build without re-running configure first.

Fallback for any version: the configure log. Each merged fragment prints one line
containing `Merged configuration` followed by the file path (the first file logs as
`Loaded configuration`); wording varies across versions, so grep loosely:

```shell
west build -b <board-target> <app-dir> 2>&1 | grep -E "(Loaded|Merged) configuration"
```

Gap-fill: the `Loaded configuration '<file>'` first line and the closing
`Configuration saved to '<file>'` line were checked at Zephyr 4.4.1; verify against your
checkout's configure output if you script against them.

`CMakeCache.txt` is the last-resort fallback and also explains "settings I never typed":
`-DCONFIG_X=y` command-line assignments, `CONF_FILE`, and `EXTRA_CONF_FILE` persist in the
cache across rebuilds, so a fragment registered once stays registered until a pristine
build. Gap-fill: exact cache entry names depend on how the build was configured; grep your
own cache and verify against your checkout:

```shell
grep -E "^(CONFIG_|CONF_FILE|EXTRA_CONF_FILE)" <build-dir>/CMakeCache.txt
```

## Lifecycle: when .config regenerates

- `.config` is re-merged from scratch whenever any input changes: any merged fragment, any
  parsed Kconfig file, the board, or the devicetree-derived symbols. The build keeps a
  checksum record of its configuration inputs to decide this; the consequence is all you
  need: touch an input and the next build re-merges, discarding any direct `.config` edits
  (including menuconfig edits).
- A pristine build (`west build -p`) always regenerates `.config` and also wipes the CMake
  cache, forgetting cached `EXTRA_CONF_FILE` and `-DCONFIG_X` values. If in doubt about
  staleness, pristine is the reliable reset.
- Editing a fragment does NOT update `.config` until the next build runs configure. Reading
  `.config` between the edit and the rebuild shows the old values: always rebuild before
  concluding.
- After any `.config` change, a rebuild is required before `autoconf.h` and the compiled
  code pick it up. Stale generated headers are the top cause of "I set it but the #ifdef is
  still false" (`references/config-from-c.md`).

## autoconf.h: what C sees

`.config` is translated into `<build-dir>/zephyr/include/generated/zephyr/autoconf.h`
(Zephyr 3.7 or newer; older trees lack the inner `zephyr/`). It is force-included into
every C, C++, and assembly compile via `-imacros`: never include it manually. Bool `y`
becomes `#define CONFIG_X 1`, bool `n` is absent (not defined 0), int and hex become
literals, strings become quoted C literals. Grep it when you need to confirm what the
compiler saw, macro forms and pitfalls live in `references/config-from-c.md`:

```shell
grep "CONFIG_LOG " <build-dir>/zephyr/include/generated/zephyr/autoconf.h
```

## Kconfig/Kconfig.dts: the devicetree bridge

At the top of the build directory (not under `zephyr/`), the generated `Kconfig/Kconfig.dts`
defines one `DT_HAS_<COMPAT>_ENABLED` symbol per devicetree binding compatible. These
symbols gate driver defaults, which is why enabling a devicetree node can flip CONFIG_
symbols with no `.conf` line anywhere:

```shell
grep -A 1 "DT_HAS_BOSCH_BME280_ENABLED" <build-dir>/Kconfig/Kconfig.dts
```

If a wanted symbol is blocked by a `DT_HAS_..._ENABLED (=n)` term, the fix is an overlay
(zephyr-devicetree skill), not a `.conf` edit. Triage flow: `references/debugging.md`.

## Worked example: prove a fragment was consumed, then catch it stale

Create and register a debug fragment:

```shell
cat > <app-dir>/debug.conf << 'EOF'
CONFIG_DEBUG_OPTIMIZATIONS=y
CONFIG_DEBUG_THREAD_INFO=y
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=4
EOF

west build -b <board-target> <app-dir> -- -DEXTRA_CONF_FILE=debug.conf
```

Prove it was consumed, with both records:

```shell
# 1. The registration record (Zephyr 4.0+)
grep -A 2 "extra-user-files" <build-dir>/build_info.yml
#     extra-user-files:
#       - /home/me/apps/blinky/debug.conf

# 2. The effect on the merged config
grep -E "CONFIG_DEBUG_OPTIMIZATIONS(=| )" <build-dir>/zephyr/.config
# CONFIG_DEBUG_OPTIMIZATIONS=y
```

Now the stale case. Edit the fragment and read `.config` immediately, without rebuilding:

```shell
sed -i.bak 's/CONFIG_LOG_DEFAULT_LEVEL=4/CONFIG_LOG_DEFAULT_LEVEL=3/' <app-dir>/debug.conf
grep "CONFIG_LOG_DEFAULT_LEVEL" <build-dir>/zephyr/.config
# CONFIG_LOG_DEFAULT_LEVEL=4      <- stale: the edit is not merged yet
west build --build-dir <build-dir>
grep "CONFIG_LOG_DEFAULT_LEVEL" <build-dir>/zephyr/.config
# CONFIG_LOG_DEFAULT_LEVEL=3      <- the input change triggered a re-merge
```

Finally, the pristine trap: `west build -p` wipes the cached `EXTRA_CONF_FILE`, so the
fragment silently drops out unless you pass it again:

```shell
west build -p -b <board-target> <app-dir>
grep -E "CONFIG_DEBUG_OPTIMIZATIONS(=| )" <build-dir>/zephyr/.config
# # CONFIG_DEBUG_OPTIMIZATIONS is not set
```

Re-register the fragment on every pristine build, or make it permanent per
`references/writing-fragments.md`.

## Out of scope here

- The full merge order and fragment registration rules: `references/merge-order.md`.
- Deciding what belongs in a fragment and authoring one: `references/writing-fragments.md`.
- Interpreting configure warnings and "symbol won't set": `references/debugging.md`.
- Sysbuild controller layout and per-image configs: `references/sysbuild.md`.
- Consuming CONFIG_ macros from C and CMake: `references/config-from-c.md`.
