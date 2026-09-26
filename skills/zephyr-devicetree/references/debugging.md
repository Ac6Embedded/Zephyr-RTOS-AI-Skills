# Debugging Devicetree Problems

When to read this: the build stops with a devicetree error or a dtc warning, the
application fails at boot (link error, `device_is_ready()` false, an API returning
-EINVAL), or an overlay edit does not seem to take effect. For pre-edit checks, see
references/validation-rules.md.

## Ground rules

- You cannot know the effective devicetree without building at least once. With no
  build directory there is nothing authoritative to debug against; build first:

```shell
west build -b <board> <app-dir>
```

- The primary inspection surface is the resolved tree, `build/zephyr/zephyr.dts`.
  Grep it instead of guessing:

```shell
grep -n -A 12 '<label>:' build/zephyr/zephyr.dts     # node body, status, pinctrl
grep -n 'zephyr,console' build/zephyr/zephyr.dts     # what a chosen key points to
```

- Version-specific vocabularies vary: the chosen-key catalog, the presence of
  `build_info.yml` (requires Zephyr 4.0 or newer), the `domains.yaml` schema. Read
  them from the user's own checkout and build directory, never from memory.
- If the build directory contains `domains.yaml`, it is a sysbuild controller and
  every image has its own `zephyr.dts`. Diagnose inside the right image's build dir:
  see references/sysbuild.md.

## Reading the error source

Three producers, three shapes:

- Lines starting with `devicetree error:` come from Zephyr's DTS and bindings
  processor (`$ZEPHYR_BASE/scripts/dts/`). File positions in these messages point into
  `build/zephyr/zephyr.dts.pre.tmp` (the preprocessed intermediate), not into your
  overlay. Open that file at the quoted line to see the merged source. On a parse
  failure `zephyr.dts` is not produced, so grep the `.pre.tmp` file instead.
- Lines like `Warning (<check_name>): /path: ...` come from dtc linting the compiled
  tree. The build usually still succeeds; do not ignore them.
- `undefined reference to '__device_dts_ord_NN'` at link time means C code consumed a
  node that never produced a device object. See the boot-failure tree below.

## Build-error triage table

Gap-fill: message wording below was checked against Zephyr's devicetree scripts
(`$ZEPHYR_BASE/scripts/dts/`) on the main branch;
older releases phrase some of these differently. Match on the quoted fragments, not
the full line, and confirm against the user's checkout when precision matters.

| Message fragment | Meaning | First move |
| --- | --- | --- |
| `parse error: undefined node label '<x>'` | An overlay writes `&<x>` but no file in the merge defines that label | Wrong board target, missing shield or snippet, typo, or stale build; see example 1 |
| `'<prop>' is marked as required in 'properties:' in '<binding>.yaml', but does not appear in <Node /path>` | The node is effectively okay and its binding requires `<prop>` | Add the property in your overlay; disabling the node also silences the check, since required checks apply only to status-okay nodes |
| `<x> controller <Node /a> for <Node /b> lacks binding` | A phandle in `gpios`, `clocks`, `pwms`, `dmas`, or `io-channels` targets a controller with no binding, so its `#<x>-cells` cannot be resolved | Verify the phandle target label; either the wrong `&label` or a controller compatible with no binding YAML in scope |
| `is not in 'enum' list in` or `is different from the 'const' value` | The value violates the binding's `enum:` or `const:` | Pick a value the binding allows; matching rules are in references/validation-rules.md |
| Cell-count mismatch naming a phandle-array property and its controller | The cells after a `&controller` do not match that controller's `#<x>-cells` | Regroup the array per controller and fix the cell count; see references/overlay-authoring.md (exact wording varies by version) |
| `is on bus` plus `No binding will be applied to this node` (warning; opt-in, recent Zephyr only, requires passing `--warn-bus-mismatch` via `EXTRA_GEN_EDT_ARGS`; a default build stays silent and the node simply gets no binding) | The device's binding declares `on-bus:` for a bus other than its parent (for example an I2C-only binding under SPI) | Move the child under the right controller type; see references/buses-and-devices.md |
| `Warning (unique_unit_address_if_enabled): /path: duplicate unit-address (also used in enabled node /other)` (fires only when both nodes are enabled and dtc is installed; Zephyr suppresses the plain `unique_unit_address` check) | Two children of one parent share the same `@<addr>` | Usually an overlay added a child under a new name instead of reopening the existing node; see example 2 |

### Example 1: undefined node label

```
devicetree error: <build>/zephyr/zephyr.dts.pre.tmp:1204 (column 9):
parse error: undefined node label 'i2c4'
```

Check in order:

1. Board target: does the board being built actually define the label?

```shell
grep -rn 'i2c4:' $ZEPHYR_BASE/boards/<vendor>/<board>/
```

2. Shield or snippet: labels contributed by a shield or snippet exist only when it is
   on the build command line. Gap-fill: shields are selected with
   `west build ... -- -DSHIELD=<name>`, snippets with `west build -S <name> ...`;
   verify the flags against the user's Zephyr docs if in doubt.
3. Typo: grep the merged intermediate for near-misses:

```shell
grep -n 'i2c' build/zephyr/zephyr.dts.pre.tmp
```

4. Previously fine overlay: an unknown `&label` in an overlay that used to build
   usually means a stale build or a changed board target, not a broken overlay. See
   the drift section below.

### Example 2: duplicate unit address

The board DTS already ships a device at the address, and the overlay adds a child
with a different node name instead of reopening it:

```dts
/* Board DTS ships: */
&i2c1 {
    bme280: bme280@76 { compatible = "bosch,bme280"; reg = <0x76>; };
};

/* Overlay (wrong): new name, same address */
&i2c1 {
    sensor@76 { compatible = "bosch,bme280"; reg = <0x76>; };
};
```

```
Warning (unique_unit_address): /soc/i2c@40013000/sensor@76: duplicate unit-address
(also used in node /soc/i2c@40013000/bme280@76)
```

Fix by reopening the existing node (`&bme280 { ... };`, or the same name `bme280@76`
which merges), or by deleting it first:

```dts
&i2c1 {
    /delete-node/ bme280@76;
    sensor@76 { compatible = "bosch,bme280"; reg = <0x76>; };
};
```

## Boot-failure decision tree

Start from the symptom.

### 1. Link error: `undefined reference to '__device_dts_ord_NN'`

The device struct was never built. Gap-fill (matches the official troubleshooting
docs): the causes, in likelihood order, are

- the node's effective status is not okay (its own `status`, or a disabled ancestor),
- no driver or binding matched the node's compatible,
- the driver's Kconfig option is off (`CONFIG_I2C=y`, `CONFIG_SENSOR=y`, ...).

Map the ordinal NN back to a node path:

```shell
grep -n 'dts_ord_<NN>' build/zephyr/include/generated/zephyr/devicetree_generated.h
```

Then grep that node in `zephyr.dts` and check its status. For the C-side
`#if !DT_NODE_EXISTS` probe and the full macro survival kit, see
references/dt-from-c.md. Deeper Kconfig triage (why a symbol will not set) is out of
scope for this skill.

### 2. `device_is_ready()` returns false

The device exists but its init function failed, or a dependency (bus, clock domain)
is not ready. Triage in order:

1. Gap-fill: enable logging (`CONFIG_LOG=y` in `prj.conf`); most drivers log the
   failing init step.
2. Gap-fill: confirm the boot banner prints (`*** Booting Zephyr OS ... ***`). If it
   does not, the failure is earlier than your driver.
3. Verify the parent bus node is status okay in the resolved tree:

```shell
grep -n -A 10 'i2c1:' build/zephyr/zephyr.dts
```

4. Verify the pinctrl state name the driver expects exists. Most drivers require a
   `"default"` state, and `pinctrl-names` is positional, paired one-to-one with
   `pinctrl-N` (references/pinctrl/model.md):

```shell
grep -n -B 4 'pinctrl-names' build/zephyr/zephyr.dts
```

### 3. An API call returns -EINVAL

Binding-valid is not boot-valid: bindings describe what the kernel accepts, but
drivers often hard-code a narrower subset and reject the rest at runtime with
-EINVAL. Example: the STM32 ADC driver rejects `differential` channels, any gain
other than 1, and any non-internal reference, even though the binding allows all of
them (see references/pinctrl/stm32.md). When a DT-described configuration is rejected
at runtime, read the driver source under `$ZEPHYR_BASE/drivers/<class>/` for the
accepted subset rather than the binding alone.

## Overlay-vs-build drift

The build reflects only the overlays it consumed at configure time. When an overlay
and `zephyr.dts` disagree, run this checklist before touching the overlay:

1. Rebuilt since the edit? An overlay edit is invisible until `west build` runs
   again. This is the answer surprisingly often.
2. Did the build consume this overlay at all? The build log prints one line per
   consumed file:

```
-- Found devicetree overlay: <path>/boards/<board>.overlay
```

   On Zephyr 4.0 or newer, `cmake.devicetree.user-files` in `build/build_info.yml` is
   the machine-readable record (references/build-artifacts.md).
3. Same board? A changed board target invalidates label assumptions; check the board
   recorded in the build (references/build-artifacts.md) against the overlay's
   intended target.

A property counts as "not built yet" when its `&label` matches no node in
`zephyr.dts`, when the property is missing on the matched node, or when the values
disagree after normalization. Normalize before declaring a mismatch:

- Macros compile to integers. The overlay writes symbols; `zephyr.dts` keeps only the
  packed numbers, so string-comparing the two produces permanent false positives:

```dts
/* Overlay source */
group0 { pinmux = <FC0_P0_PIO0_16>, <FC0_P1_PIO0_17>; };
```

```dts
/* zephyr.dts after the build (values illustrative): macros are gone */
group0 { pinmux = <0x4000200>, <0x4400200>; };
```

  Compare resolved values instead: grep numeric cells for group-style vendors, and
  grep labels for STM32-style direct references (references/pinctrl/model.md).
- Radix is not a difference: `<13 2>` and `<0xd 0x2>` are the same value.
- Boolean shorthand equals true: `input-enable;` in the overlay resolving to a true
  boolean in the build is agreement, not drift.
- Delete verbs cannot be confirmed positively. After `/delete-node/` or
  `/delete-property/`, absence in `zephyr.dts` is indistinguishable from
  never-existed. Confirm by rebuilding and checking that the grep for the deleted
  node or property comes back empty:

```shell
west build && grep -n 'bme280' build/zephyr/zephyr.dts   # expect no output
```

## Rebuild loop

```shell
west build                    # incremental: picks up overlay and prj.conf edits
west build -p always          # pristine: wipes the build dir and reconfigures
```

Gap-fill: use `-p always` when incremental behavior looks inconsistent (cascading
errors after moving or renaming overlay files, a changed board target, a switched
Zephyr checkout). After any board-target change, always go pristine.
