# Writing Configuration Fragments

When to read this: you are creating a `.conf` file (prj.conf, debug.conf,
`boards/<BOARD>.conf`), deciding which file a `CONFIG_` line belongs in, turning a
menuconfig session into a durable fragment, or trimming a bloated prj.conf.
Line-level value syntax is in references/conf-syntax.md; the full merge order is in
references/merge-order.md.

## The fragment model

- Every `.conf` file is a sparse delta over the merged configuration: it names only
  the symbols you intend to change. Everything unmentioned keeps its default.
- Fragments merge in a fixed order and later files win. prj.conf overrides the board
  defconfig because it merges later; a fragment registered via `EXTRA_CONF_FILE`
  merges after prj.conf AND after shield fragments, so it wins over both. To override
  a shield-provided setting, use an `EXTRA_CONF_FILE` fragment, not prj.conf
  (prj.conf merges before shield files). Full order: references/merge-order.md.
- The object you are shaping is `build/zephyr/.config`. Verify against that file,
  never against your intent (see Verifying below).

## Choosing a home for each line

| The setting is | Put it in |
| --- | --- |
| Needed by this app always, on every board | `prj.conf` |
| Board-specific for this app (console choice, board peripherals) | `boards/<BOARD>[_<qualifiers>].conf` (auto-picked, no registration needed) |
| A build profile: debug, release, test | a named fragment (`debug.conf`) registered via `EXTRA_CONF_FILE`, or `prj_<suffix>.conf` selected with `FILE_SUFFIX=<suffix>` |
| A one-off experiment you may throw away | a fragment passed once with `-DEXTRA_CONF_FILE` |
| Configuration of another sysbuild image (MCUboot, ...) | `sysbuild/<image>.conf`; see references/sysbuild.md |

Never edit board defconfig files, shield `.conf` files, or module-shipped fragments
to fix one application: those files apply to every build that uses them. Write an
application fragment that merges later and overrides them instead.

If the value you want to change is computed from the devicetree (a `DT_HAS_*`
gated driver, a `dt_*` derived default), no `.conf` file fixes it: change the
overlay (the zephyr-devicetree skill), then rebuild.

## The four authoring rules

Write fragments the way Zephyr's own minimal-configuration saver
(savedefconfig-style output, the `D` key in menuconfig) writes them. A fragment
that follows all four rules contains exactly the minimal intent and nothing else.

1. Write only prompted symbols. Assigning a promptless symbol aborts the build
   with a message containing "not directly user-configurable (has no prompt)"
   (wording varies across versions). Set the prompted symbol, or the devicetree
   node, that drives it instead.

```conf
CONFIG_CPU_CORTEX_M4=y     # WRONG: promptless, selected by the SoC; build aborts
```

2. Skip symbols already forced `y` by `select`. They cannot be turned off from any
   `.conf` file, and writing them as `=y` is dead weight. To turn one off, find and
   disable the selector (references/debugging.md).

```conf
CONFIG_SHELL_BACKEND_SERIAL=y
CONFIG_RING_BUFFER=y       # redundant: SHELL_BACKEND_SERIAL selects RING_BUFFER
```

3. Skip values equal to their default in the current context. Restating defaults is
   noise, and this rule also drops whole trains of cascaded dependents: enabling one
   subsystem symbol pulls in dozens of defaulted `LOG_*`, `SHELL_*`, ... lines that
   never belong in a fragment. Beware that defaults are conditional (ordered
   value/condition pairs, first satisfied wins), so "equal to the default" depends
   on the rest of the configuration.

```conf
CONFIG_LOG=y
CONFIG_LOG_MODE_DEFERRED=y   # redundant if it is already the default log mode
```

4. Skip the default selection of a non-optional choice. Only write a choice member
   when you pick a NON-default member, and write only that member as `=y`. Never
   write another member as n (references/conf-syntax.md).

## Deriving a fragment from a menuconfig session

menuconfig edits `build/zephyr/.config` directly. Those edits are temporary
experiments: they survive incremental builds but are lost on a pristine build or
whenever any fragment or Kconfig source file changes. Persist them with one of two
methods, then rebuild from the fragment.

Method 1: minimal save from inside menuconfig. Press `D` (save minimal
configuration) and give a file name. The saved file already applies the four rules.

```shell
west build -t menuconfig --build-dir <build-dir>
# make your changes, then press D and save to <app-dir>/debug.conf
```

Gap-fill: the minimal file reflects every non-default setting in the merged
configuration, not only your session's edits, so it can include lines that already
come from the board defconfig or your existing prj.conf. Strip lines you did not
change before shipping the fragment (verify against your checkout by comparing the
saved file with your board's defconfig).

Method 2: diff two `.config` snapshots. Copy `.config` before the session, diff
after, keep the changed lines, then apply the four rules to what remains.

```shell
cp <build-dir>/zephyr/.config <build-dir>/zephyr/.config.before
west build -t menuconfig --build-dir <build-dir>
diff <build-dir>/zephyr/.config.before <build-dir>/zephyr/.config
```

Remember that `# CONFIG_X is not set` lines in the diff are real assignments of n,
not comments; carry them into the fragment in that exact form when you turned
something off.

## Registering the fragment

A new fragment file has no effect until the build knows about it. Creating the file
is not enough. Three ways to register:

1. Use an auto-picked name. `boards/<BOARD>[_<qualifiers>].conf`,
   `socs/<SOC>_<QUALIFIERS>.conf`, and `prj_<suffix>.conf` (with
   `FILE_SUFFIX=<suffix>`) are found automatically in the application configuration
   directory (references/merge-order.md).

2. Pass `EXTRA_CONF_FILE` on the command line. Multiple files are semicolon or
   space separated and merge in the order given:

```shell
west build -b <board> <app-dir> -- -DEXTRA_CONF_FILE=debug.conf
west build -b <board> <app-dir> -- -DEXTRA_CONF_FILE="debug.conf;net.conf"
```

   Relative paths resolve against the application configuration directory (usually
   the application source directory). A missing file is a CMake FATAL_ERROR, so
   typos fail loudly. The value is cached by CMake: it stays active across
   incremental rebuilds without repeating it, and is dropped again on a pristine
   build (`west build -p`) that does not repeat it. When you change which config
   files a build uses, prefer a pristine build to avoid stale cache surprises.

3. Gap-fill: set it in CMakeLists.txt before `find_package(Zephyr ...)` when the
   fragment must always apply (same mechanism the Zephyr application docs show for
   `set(BOARD ...)`; verify against `doc/develop/application/index.rst` in your
   checkout):

```cmake
cmake_minimum_required(VERSION 3.20.0)
set(EXTRA_CONF_FILE always-on.conf)
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(my_app)
```

Do NOT register additive fragments through `CONF_FILE`: setting `CONF_FILE`
REPLACES prj.conf and disables the automatic pickup of `boards/` and `socs/`
fragments. `EXTRA_CONF_FILE` adds on top and is almost always what you want. This
is the classic confusion; details in references/merge-order.md.

## Style rules inside the fragment

- Group related lines under `#` comment headers; one concern per fragment
  (debug.conf holds debug settings only).
- Follow generated-file conventions: bool off as `# CONFIG_X is not set`, never
  `=n` (it parses but generated files never emit it); strings double-quoted; hex
  with `0x`; int in decimal (references/conf-syntax.md).
- No duplicate assignments. Within a file and across fragments the last assignment
  wins, silently by design; a duplicate hides the effective value from readers.
- Do not restate defaults (rule 3): a fragment is intent, not a snapshot.

## Verifying the fragment took effect

After a configure, check the merged result and the recorded file list:

```shell
grep -E "CONFIG_(DEBUG_OPTIMIZATIONS|DEBUG_THREAD_INFO|LOG|LOG_DEFAULT_LEVEL)[= ]" \
    <build-dir>/zephyr/.config
grep -A5 "extra-user-files" <build-dir>/build_info.yml   # Zephyr 4.0+
```

`build_info.yml` key `cmake.kconfig.extra-user-files` lists the `EXTRA_CONF_FILE`
entries the build consumed; `cmake.kconfig.files` is the full ordered merge list
(references/build-artifacts.md). If a symbol did not take the value you wrote, read
the configure output: an unmet-dependency assignment only WARNS and the feature
silently stays off (references/debugging.md).

## Worked example 1: a debug fragment end to end

Goal: a debug profile with debugger-friendly optimizations, thread info, and
verbose logging, without touching prj.conf.

Create `<app-dir>/debug.conf`:

```conf
# Debugger-friendly code generation
CONFIG_DEBUG_OPTIMIZATIONS=y
CONFIG_DEBUG_THREAD_INFO=y

# Verbose logging (level 4 = debug)
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=4
```

Register and build:

```shell
west build -b <board> <app-dir> -- -DEXTRA_CONF_FILE=debug.conf
```

Verify:

```shell
grep -E "CONFIG_(DEBUG_OPTIMIZATIONS|DEBUG_THREAD_INFO|LOG|LOG_DEFAULT_LEVEL)[= ]" \
    <build-dir>/zephyr/.config
# expect: CONFIG_DEBUG_OPTIMIZATIONS=y, CONFIG_DEBUG_THREAD_INFO=y,
#         CONFIG_LOG=y, CONFIG_LOG_DEFAULT_LEVEL=4
grep -A3 "extra-user-files" <build-dir>/build_info.yml
# expect the absolute path of debug.conf
```

Prove it is a profile, not a permanent change:

```shell
west build -p -b <board> <app-dir>
grep "CONFIG_DEBUG_OPTIMIZATIONS" <build-dir>/zephyr/.config
# expect: # CONFIG_DEBUG_OPTIMIZATIONS is not set
```

The pristine build without the flag dropped the fragment; repeat
`-DEXTRA_CONF_FILE=debug.conf` whenever you want the profile back.

## Worked example 2: shrinking a bloated prj.conf

Gap-fill: the three offending symbols below were verified against Zephyr 4.4
sources (`arch/arm/core/cortex_m/Kconfig`, `subsys/shell/backends/Kconfig.backends`,
`kernel/Kconfig`); definitions can move between versions, confirm in your checkout.

Before:

```conf
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_SERIAL=y
CONFIG_RING_BUFFER=y
CONFIG_CPU_CORTEX_M4=y
CONFIG_MAIN_STACK_SIZE=1024
CONFIG_MAIN_STACK_SIZE=4096
```

Delete three lines, each for a different rule:

- `CONFIG_CPU_CORTEX_M4=y`: rule 1 violation. Promptless, selected by the SoC
  configuration; this line ABORTS the build ("Aborting due to Kconfig warnings",
  wording varies across versions).
- `CONFIG_RING_BUFFER=y`: rule 2 violation. `CONFIG_SHELL_BACKEND_SERIAL` selects
  `RING_BUFFER`, so it is already forced `y`; the line is dead weight.
- `CONFIG_MAIN_STACK_SIZE=1024`: rule 3 violation AND a shadowed duplicate. 1024 is
  the usual default, and the later `=4096` line wins anyway (last assignment wins),
  so this line only misleads readers.

After:

```conf
# Shell over UART
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_SERIAL=y

# main() needs headroom for the parser
CONFIG_MAIN_STACK_SIZE=4096
```

Rebuild pristine and re-check `<build-dir>/zephyr/.config`: the effective values
are unchanged except that the build no longer aborts, and every remaining line now
states a real decision.
