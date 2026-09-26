# Debugging Kconfig Problems

When to read this: the configure step prints "was assigned the value 'y' but got the
value 'n'", the build aborts with "Aborting due to Kconfig warnings", a symbol will
not set no matter what you write in prj.conf, or you need to explain why CONFIG_X has
the value it has. For pre-edit syntax checks, see references/conf-syntax.md.

## Ground rules

- `build/zephyr/.config` is the truth. Every question of the form "is CONFIG_X on"
  is answered there, never by reading prj.conf alone:

```shell
grep -E "^CONFIG_<X>[=\"]|^# CONFIG_<X> is not set" build/zephyr/.config
```

  `# CONFIG_X is not set` is an assignment of `n`, not a comment. No match at all
  means the symbol does not exist in this build's Kconfig tree.
- The effective configuration only exists after a configure. If there is no build
  directory, build first, then debug.
- If the build directory contains `domains.yaml`, it is a sysbuild controller and has
  no top-level `zephyr/.config`. Each image has its own full config at
  `<build>/<image>/zephyr/.config`; diagnose inside the right image's directory and
  see references/sysbuild.md before anything else.
- Kconfig diagnostics appear in the CMake configure output, not in the compile
  output. To see them again without a full rebuild, re-run just that stage:

```shell
west build --cmake-only --build-dir <build>
```

## Error taxonomy

Message texts below were checked against Zephyr 4.4 (`$ZEPHYR_BASE/scripts/kconfig/`).
Match on the quoted fragments; surrounding wording can vary across versions.

| Symptom in configure output | Severity | Meaning | First move |
| --- | --- | --- | --- |
| `was assigned the value 'y' but got the value 'n'. Check these unsatisfied dependencies: <TERM> (=n)` | Warning only, build continues | Your assignment was accepted but a dependency is false, so the symbol stays `n`. The feature is silently missing from the firmware | Fix each listed `(=n)` term; see the unmet-dependency section |
| `attempt to assign the value 'y' to the undefined symbol <NAME>` followed by `error: Aborting due to Kconfig warnings` | Hard error | A `.conf` file assigns a symbol that does not exist in this tree | Typo, symbol from a newer Zephyr, or a module missing from the west manifest; see the unknown-symbol section |
| `is not directly user-configurable (has no prompt). It gets its value indirectly from other symbols.` followed by `error: Aborting due to Kconfig warnings` | Hard error | A `.conf` file assigns a promptless symbol | Delete the line; set the prompted symbol (or devicetree node) that drives it instead |
| A choice member you set `y` shows a warning and another member wins | Warning only | A later fragment selected a different member, or your member's dependencies are unmet (wording varies across versions) | Grep the fragments in merge order for every member of the choice; see references/symbols-and-dependencies.md |
| The same symbol assigned in several fragments, no message at all | Silent by design | Overriding across fragments is the normal mechanism; the last assignment in merge order wins | Not an error; if the final value surprises you, run the provenance procedure below |

The severity split is the critical fact: unknown symbols and promptless assignments
abort the build, but an unmet-dependency assignment only warns and the build
succeeds without the feature. Always read the configure output even when the build
is green.

## Unmet-dependency warning (build continues)

Full message shape:

```
warning: <NAME> (defined at <path>:<line>) was assigned the value 'y' but got the
value 'n'. Check these unsatisfied dependencies: <TERM> (=n). See
http://docs.zephyrproject.org/latest/kconfig.html#CONFIG_<NAME>
```

Read it mechanically:

- `defined at <path>:<line>` is the Kconfig definition site; open it to see the full
  `depends on` line.
- The listed terms are exactly the false AND terms of the symbol's effective
  dependency (its own `depends on` plus every enclosing `if` and `menu` condition).
  Each one must become `y` (or true) for your assignment to take effect.
- The docs link at the end resolves for upstream symbols; application-defined
  symbols are not on the docs site.

Reproduce it deliberately to learn the shape (both symbols exist upstream):

```conf
# prj.conf: backend requested, but the subsystem it depends on is off
CONFIG_SHELL=y
CONFIG_SHELL_LOG_BACKEND=y
```

The configure step warns `unsatisfied dependencies: LOG (=n)`, the build succeeds,
and `.config` contains `# CONFIG_SHELL_LOG_BACKEND is not set`. The fix is one line:

```conf
CONFIG_LOG=y
```

Fix each `(=n)` term the same way: if the term is a prompted symbol, set it in your
fragment; if it is promptless, chase what drives it; if it is a `DT_HAS_*` symbol,
the fix is in devicetree (worked example at the end of this file).

## Unknown symbol (build aborts)

```
warning: attempt to assign the value 'y' to the undefined symbol SHELLL
...
error: Aborting due to Kconfig warnings
```

Causes in likelihood order:

1. Typo. Compare against near misses in the parsed tree:

```shell
grep -i "config SHELL" $(cat build/zephyr/kconfig/sources.txt) 2>/dev/null | sort -u
```

2. The symbol exists only in a newer Zephyr than the one this workspace pins. Check
   the definition site in the docs or search your actual checkout
   (references/finding-options.md), not your memory of upstream.
3. The symbol belongs to a module (out-of-tree driver, MCUboot, ...) that is not in
   the west manifest, so its Kconfig files were never parsed. `sources.txt` lists
   every file the build did parse; if the module's Kconfig is absent there, fix the
   manifest, not the fragment.

## Promptless assignment (build aborts)

```
error: <NAME> is assigned in a configuration file, but is not directly
user-configurable (has no prompt). It gets its value indirectly from other symbols.
```

(Stable fragment: `not directly user-configurable (has no prompt)`.) A promptless
symbol is driven only by defaults and `select`. Delete the assignment, then find the
prompted symbol that drives it: open the symbol in the menuconfig info screen and
read its defaults and the symbols that select it (references/menuconfig-guiconfig.md).
`DT_HAS_*` symbols are the common promptless case: their value follows the
devicetree, so the "assignment" you want is an overlay change.

## Decision tree: symbol will not set

You wrote `CONFIG_X=y` (or a value) and `.config` disagrees. Check in order; each
step maps to one of the three per-symbol states (visibility, value, assignability)
described in references/symbols-and-dependencies.md.

1. Did the fragment actually merge? Creating a file is not registering it. Confirm
   it is listed under `cmake.kconfig.files` in `build/build_info.yml` (Zephyr 4.0+),
   or in the configure log's merged-configuration lines
   (references/build-artifacts.md). If missing, register it
   (references/merge-order.md) and reconfigure.
2. Is the symbol promptless? If the build aborted with the promptless error, or the
   info screen shows no prompt, stop: users and fragments can never set it. Drive it
   via its selectors or defaults.
3. Is a dependency false? The unmet-dependency warning names the blockers directly.
   No warning but still `n` usually means step 1 or step 5 instead.
4. Is it select-forced and you are trying to turn it OFF? A symbol forced `y` by
   `select` cannot be set to `n` from any `.conf` file, and no warning tells you so.
   Open the symbol's info screen and read the list of symbols currently selecting
   it, then disable the selector, not the symbol.
5. Is a later fragment overriding you? Overrides are silent. Grep every merged file
   in order (procedure below); the last assignment wins, and
   `# CONFIG_X is not set` counts as an assignment of `n`.
6. Is it a choice member? You can only set a member to `y`, never to `n`, and the
   last member set `y` across all fragments wins. Set the member you want to `y`
   instead of fighting the others (references/symbols-and-dependencies.md).
7. Is the blocking term a `DT_HAS_*` symbol or a DT-derived default? Then no `.conf`
   edit can fix it: change the devicetree overlay (node present, `status = "okay"`)
   with the zephyr-devicetree skill, then reconfigure and re-check.

## Provenance: why does CONFIG_X have this value

Generic procedure, most direct first:

1. Symbol info screen. Run `west build -t menuconfig --build-dir <build>`, search
   with `/`, open the symbol's info. It shows the value, each dependency term with
   its current value, the defaults with their conditions, select/imply in both
   directions, and every definition site (references/menuconfig-guiconfig.md). This
   answers most provenance questions alone.
2. Find the user assignment, if any. List the merged files in order, then grep each
   for both assignment forms; the LAST hit is the assignment that won:

```shell
# Ordered merge list (Zephyr 4.0+): board defconfig first, later files win
grep -A 30 "kconfig:" build/build_info.yml
```

```shell
grep -H -E "CONFIG_<X>[=\"]|CONFIG_<X> is not set" <each-listed-file-in-order>
```

   On older Zephyr, the configure log's merged-configuration lines give the same
   list (references/build-artifacts.md).
3. No hit anywhere: the value comes from a default, a `select`, or an `imply`. Read
   them off the info screen from step 1. First satisfied default wins; a `select`
   from any enabled symbol forces `y` regardless of defaults.
4. Default references `dt_*` functions or a `DT_HAS_*` symbol: the value is derived
   from the resolved devicetree and moves with overlays and board files, never with
   `.conf` edits. Hand off to the zephyr-devicetree skill for the DT side.
5. Re-read the configure output for warnings naming the symbol: an ignored
   unmet-dependency warning explains most "I set it but it is off" cases.

## Staleness

- `.config` is re-merged whenever any input changes: any merged fragment, any parsed
  Kconfig file, board or Zephyr revision. The build keeps a record of its config
  inputs and reconfigures when they differ; if in doubt, force it:

```shell
west build -p always
```

- Consequence 1: menuconfig edits are temporary. They survive incremental builds but
  are lost on a pristine build or whenever any fragment or Kconfig source changes.
  Persist them as a fragment (references/writing-fragments.md).
- Consequence 2: after ANY `.config` change, a rebuild must run before generated
  headers pick it up. Stale headers are the top cause of "I set it but the #ifdef is
  still false" (references/config-from-c.md).
- Consequence 3: updating Zephyr shifts defaults and dependency graphs. After an
  update, reconfigure and diff the old and new `.config` before trusting prior
  conclusions; symbols may appear, disappear, or change default.
- `build_info.yml` is only rewritten when `.config` is actually regenerated, so its
  kconfig keys can describe a previous configure. When file lists look wrong, go
  pristine and re-check (references/build-artifacts.md).

## Worked example: DT_HAS-gated driver symbol

Symptom: you want the BME280 sensor driver, so prj.conf says

```conf
CONFIG_SENSOR=y
CONFIG_BME280=y
```

The build succeeds, but the configure output shows

```
warning: BME280 ... was assigned the value 'y' but got the value 'n'. Check these
unsatisfied dependencies: DT_HAS_BOSCH_BME280_ENABLED (=n). ...
```

Triage, following the tree above:

1. The fragment merged (the warning proves the assignment was seen), the symbol
   exists, a dependency is false: step 3 of the decision tree.
2. The blocker is `DT_HAS_BOSCH_BME280_ENABLED`, a generated promptless symbol that
   is `y` only when an enabled devicetree node with compatible `bosch,bme280`
   exists. The build generates one such symbol per binding compatible into
   `<build>/Kconfig/Kconfig.dts` (the compatible maps to the name by uppercasing and
   replacing `-,.@/+` with `_`). No `.conf` line can change it: step 7, the fix is
   in devicetree.
3. Add the node in the app overlay (zephyr-devicetree skill for details):

```dts
&i2c0 {
    bme280@76 {
        compatible = "bosch,bme280";
        reg = <0x76>;
    };
};
```

4. Reconfigure and verify:

```shell
west build -b <board> <app-dir>
grep -E "^CONFIG_BME280|^CONFIG_I2C=" build/zephyr/.config
```

Expect `CONFIG_BME280=y` and `CONFIG_I2C=y`. Now delete `CONFIG_BME280=y` from
prj.conf and rebuild: the symbol STAYS `y`, because the driver's Kconfig follows the
upstream pattern (`default y` gated by `depends on DT_HAS_BOSCH_BME280_ENABLED`).
Drivers enable themselves when their node exists with status okay and the subsystem
toggle is on; the prj.conf line was never the real switch. Keep prj.conf minimal
(references/writing-fragments.md) and treat the devicetree as the source of truth
for which drivers exist.

The same pattern explains the inverse surprise: `CONFIG_SPI=y` with no enabled SPI
controller node turns the bus symbol on but compiles no controller driver. When a
subsystem seems enabled yet does nothing, check the `DT_HAS_*` symbols of its
drivers before checking anything else in Kconfig.
