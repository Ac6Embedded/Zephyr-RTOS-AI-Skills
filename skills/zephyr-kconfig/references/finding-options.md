# Finding the Right Kconfig Option

When to read this: you know the behavior you want (enable logging, grow a stack, turn on a driver) but not which CONFIG_ symbol controls it, or you have a symbol name and need to understand it before setting it. For what the symbol attributes mean, read references/symbols-and-dependencies.md; for a symbol that refuses to take a value, read references/debugging.md.

## Two search paths

- Interactive: menuconfig search. Needs a configured build directory; shows live values, resolved dependencies, and the symbol info screen. Best when you also need to know WHY a symbol has its current value.
- grep over Kconfig sources. Needs no build and is faster for scripted work, but it shows definitions, not resolved values: defaults, ranges, and devicetree gating only resolve at configure time. Confirm with a build afterwards.

## Searching inside menuconfig

Start it against an existing build directory:

```shell
west build -t menuconfig --build-dir <build-dir>
```

Press `/` to open the search dialog (this also works in guiconfig). Behavior:

- The query is split on whitespace into tokens. Each token is a regular expression, and ALL tokens must match (AND).
- Only the symbol NAME and its PROMPT text are searched. Help text is never searched: if you only remember a phrase from the help, use grep instead (below).
- Results are ordered by kind: symbols first, then choices, menus, comments; alphabetical within each group.

Query examples: `log buffer` finds `LOG_BUFFER_SIZE` (both tokens match the name); `^LOG_` anchors to names starting with `LOG_`; `uart console` finds entries whose name or prompt contains both.

Gap-fill (verified against Zephyr 4.4 docs, double-check on your checkout): selecting a result jumps straight to that entry in the tree. If it is not currently visible, show-all mode turns on so you can still inspect it (toggle with `A` in menuconfig, `Ctrl-A` in guiconfig). `Ctrl-F` shows the help of the highlighted result without leaving the dialog, and `?` on any entry opens its full info screen.

## Read the symbol info screen before setting anything

Open the info screen on the symbol you found (see references/menuconfig-guiconfig.md for the UI). It shows:

- The prompt and help text of every definition site (a symbol can be defined in several places).
- Direct dependencies, split per AND term, with each term's current value. The terms currently `n` are exactly what blocks the symbol.
- Every `default`, with its condition. The first satisfied default wins.
- Selects and implies in both directions: what this symbol forces, and what forces it ("selected by" a `y` symbol means nothing you write can turn it off).
- Definition locations (file and line) and the menu path.

This screen is the primary provenance tool. Read it before writing any `.conf` line.

## Decision checklist before writing the line

Walk this list top to bottom, using the info screen or the grepped definition:

1. Does it have a prompt? No prompt: never assign it, the build aborts. Set the prompted symbol (or the devicetree node) that drives it instead.
2. Are all dependency terms currently `y`? A false term means your assignment only WARNS at configure time and the feature silently stays off (references/debugging.md).
3. Is it already forced `y` by a `select`? Then no line is needed, and no `.conf` line can turn it off (disable the selector instead).
4. Is its default already the value you want in this context? Skip the line entirely (references/writing-fragments.md).
5. Is it a choice member? Set the member you want to `y`; never write `n` for the others (references/conf-syntax.md).
6. Is its default computed from devicetree (`DT_HAS_*` symbols, `dt_*` functions)? The fix belongs in an overlay, not a `.conf` file (the zephyr-devicetree skill).

Then write the line per references/conf-syntax.md and make sure the file it lives in is actually merged, per references/merge-order.md.

## Grep discovery (no build needed)

Kconfig source files define symbols WITHOUT the `CONFIG_` prefix. Grep for `config <NAME>`, not `CONFIG_<NAME>`:

```shell
grep -rn "config LOG_BUFFER" "$ZEPHYR_BASE/subsys" --include="Kconfig*"
```

Useful variants:

```shell
# You only know the menu text the option shows
grep -rn 'bool "Logging"' "$ZEPHYR_BASE" --include="Kconfig*"

# Show a definition together with its help text (menuconfig search cannot)
grep -rn -A 8 "config LOG_BUFFER_SIZE" "$ZEPHYR_BASE/subsys/logging" --include="Kconfig*"
```

Where to search: `$ZEPHYR_BASE` (subsys, drivers, kernel, arch, soc, boards), your application's own Kconfig file, and any module trees in the workspace. With a built tree, skip the guessing: `<build>/zephyr/kconfig/sources.txt` lists every Kconfig file the build actually parsed (absolute paths, one per line), including modules and generated files such as `<build>/Kconfig/Kconfig.dts`:

```shell
xargs grep -n "config LOG_BUFFER" < <build>/zephyr/kconfig/sources.txt
```

See references/build-artifacts.md for the rest of the artifact map.

If a symbol appears in an article or a forum post but nowhere in your tree, it usually comes from a newer Zephyr version or from a module your manifest does not pull in. Assigning it anyway is a hard error: the configure step stops with "Aborting due to Kconfig warnings" (wording varies across versions).

## The online option reference

Every symbol in the upstream Zephyr tree has a documentation page at a predictable URL:

```shell
# open in a browser, substituting the symbol name
https://docs.zephyrproject.org/latest/kconfig.html#CONFIG_<NAME>
```

The unmet-dependency warning printed at configure time links to this same page. Symbols you define in your own application do not appear there (they do appear in menuconfig search). Gap-fill: the kconfig.html page itself is a searchable index of all upstream symbols; double-check availability for your Zephyr version.

## Samples show idiomatic combinations

A symbol rarely works alone. To see what usually accompanies it, grep the upstream samples' configuration files and read whole files from the hits:

```shell
grep -rln "CONFIG_LOG=y" "$ZEPHYR_BASE/samples" --include="prj.conf" | head
```

Widen to `--include="*.conf"` to catch debug/release fragments and board-specific files. Treat samples as style references for combinations, then still run the decision checklist on each symbol before copying it.

## Driver symbols: check the devicetree first

Most device driver symbols are written as `default y` gated on a generated `DT_HAS_<COMPAT>_ENABLED` symbol: the driver enables itself when a matching devicetree node exists with status okay and the bus or subsystem is on. Two consequences when searching:

- If the option you found refuses to turn on and the blocker is `DT_HAS_..._ENABLED (=n)`, the lever is a devicetree overlay, not a `.conf` line. Triage in references/debugging.md, fix with the zephyr-devicetree skill.
- If an option turned on that you never wrote anywhere, a DT-gated `default y` probably fired; that is normal.

## Worked example: make CONFIG_LOG produce output

Goal: enable logging, with debug verbosity for the I2C driver only.

Step 1, find the switch. In menuconfig press `/` and type `logging`, or grep:

```shell
grep -rn "config LOG$" "$ZEPHYR_BASE/subsys/logging" --include="Kconfig*"
# -> $ZEPHYR_BASE/subsys/logging/Kconfig: config LOG, prompt "Logging"
```

The info screen shows a prompted bool with no blockers on a typical board: this is the global subsystem toggle. Without `CONFIG_LOG=y`, every log call compiles out no matter what else you set.

Step 2, find the verbosity knobs. Searching `log level` returns `LOG_DEFAULT_LEVEL` (int, range 0 to 4, default 3) plus many per-module symbols such as `I2C_LOG_LEVEL`. Gap-fill (verified in a Zephyr 4.4 checkout, double-check on yours): each subsystem or driver instantiates a shared template, `$ZEPHYR_BASE/subsys/logging/Kconfig.template.log_config`, like this:

```kconfig
module = I2C
module-str = i2c
source "subsys/logging/Kconfig.template.log_config"
```

The template generates a choice (`I2C_LOG_LEVEL_OFF/ERR/WRN/INF/DBG/DEFAULT`) and a promptless int `I2C_LOG_LEVEL` derived from whichever member is `y`; the `DEFAULT` member falls back to `LOG_DEFAULT_LEVEL`. So the settable knob is the choice member, not the int.

Step 3, run the checklist. `LOG`: prompted, no blockers, default n, write it. `LOG_DEFAULT_LEVEL`: default 3 already; write it only if you want another value. `I2C_LOG_LEVEL_DBG`: a choice member (set it to `y`, do not touch its siblings), and it depends on both `LOG` and `I2C`, so the I2C driver must be enabled (usually automatic via its DT gating). `I2C_LOG_LEVEL`: promptless, never assign it.

Step 4, write prj.conf:

```conf
CONFIG_LOG=y
CONFIG_I2C_LOG_LEVEL_DBG=y
```

Step 5, build and verify against the merged truth:

```shell
west build -b <board> <app-dir>
grep -E "CONFIG_LOG(=| )|CONFIG_I2C_LOG_LEVEL" <build>/zephyr/.config
```

Expected result:

```conf
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=3
CONFIG_I2C_LOG_LEVEL_DBG=y
CONFIG_I2C_LOG_LEVEL=4
```

The promptless `CONFIG_I2C_LOG_LEVEL=4` was derived for you from the choice member. If instead the configure output warns that your symbol "was assigned the value 'y' but got the value 'n'" (wording varies across versions), one of the checklist items above failed: go to references/debugging.md.
