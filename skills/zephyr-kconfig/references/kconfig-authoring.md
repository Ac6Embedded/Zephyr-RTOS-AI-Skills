# Authoring Kconfig Files (Applications and Modules)

When to read this: you are adding your own `CONFIG_` symbols to an application
or a west module, writing or fixing a `Kconfig` file, or adding a default to an
existing symbol. To set existing options, see `references/writing-fragments.md`;
for what the constructs mean at merge time, `references/symbols-and-dependencies.md`.

## Where the file goes, and its canonical shape

Create a file named `Kconfig` in the application source directory, next to the
top-level `CMakeLists.txt`. The build auto-detects it and uses it as the Kconfig
root instead of `$ZEPHYR_BASE/Kconfig`. No CMake changes are needed.

The canonical shape is: `mainmenu`, your symbols, then `source "Kconfig.zephyr"`
as the last line:

```kconfig
# <app>/Kconfig
mainmenu "My application"

config MYAPP_FEATURE
	bool "Enable my feature"
	help
	  What the feature does and when to turn it on.

source "Kconfig.zephyr"
```

The last line is mandatory. `source` paths resolve against `$ZEPHYR_BASE`, and
`Kconfig.zephyr` pulls in the entire Zephyr configuration tree. If you omit it,
every Zephyr symbol becomes undefined, so the assignments coming from the board
defconfig and prj.conf hit undefined symbols and the configure step aborts
("Aborting due to Kconfig warnings"; wording varies across versions).

Indent bodies with one tab (the convention throughout `$ZEPHYR_BASE`). The
extent of a `help` block is defined by its indentation, so keep its lines
aligned.

## Construct survival kit

### config blocks

One block per symbol. The type line usually carries the prompt; a symbol whose
type line has no prompt string is promptless and can never be set by users or
fragments (assigning it in a `.conf` file is a hard error).

```kconfig
config MYAPP_TELEMETRY
	bool "Send telemetry"
	depends on LOG                 # visibility gate; ANDs with enclosing conditions
	select MYAPP_TELEMETRY_CORE    # force a promptless helper to y
	imply SENSOR_ASYNC_API         # weak default on the target; user can still set n
	default y if DEBUG
	default n
	help
	  Periodically push measurements to the log backend.
```

`depends on` gates whether the symbol can be seen and set. `select` forces its
target to `y` and ignores the target's own dependencies (see the design rules
below before using it). `imply` sets a default the user can override. `default`
lines are ordered (value, condition) pairs: the first one whose condition holds
wins, and any user assignment beats them all.

All four Zephyr types (there is no tristate in Zephyr: every bool is `y` or `n`):

```kconfig
config MYAPP_RETRIES
	int "Retry count"
	range 0 255
	default 3

config MYAPP_BASE_ADDR
	hex "Buffer base address"
	default 0x20001000

config MYAPP_DEVICE_LABEL
	string "Device label"
	default "sensor0"
```

`range` applies to int and hex only and may be conditional (`range 1 8 if X`);
the first range whose condition holds is active. Out-of-range values assigned
from a `.conf` file are clamped silently (see `references/conf-syntax.md`).

`def_bool <expr>` is shorthand for `bool` plus `default <expr>`, typical for
promptless computed symbols.

### menu and the menuconfig keyword

`menu` groups entries; its `depends on` condition ANDs into every member.
`visible if` additionally hides a menu without affecting member values (menus
only).

```kconfig
menu "Application tuning"

config MYAPP_QUEUE_DEPTH
	int "Queue depth"
	default 8

endmenu
```

`menuconfig` declares a symbol that is rendered as a menu, with its dependents
as children. Guard the children with an `if` block so they disappear together:

```kconfig
menuconfig MYAPP_NET
	bool "Networking features"

if MYAPP_NET

config MYAPP_NET_RETRIES
	int "Connection retries"
	default 3

endif # MYAPP_NET
```

### choice blocks

A choice is a radio group: exactly one member is `y` while the choice is active,
unless the choice is marked `optional` (then all members may be `n`). Users
select a member by setting it to `y`; a member is never written as `n`.

```kconfig
choice MYAPP_TRANSPORT
	prompt "Telemetry transport"
	default MYAPP_TRANSPORT_UART

config MYAPP_TRANSPORT_UART
	bool "UART"

config MYAPP_TRANSPORT_BLE
	bool "Bluetooth LE"

endchoice
```

Do not write the default member of a non-optional choice into fragments; it is
implicit. Semantics and pitfalls: `references/symbols-and-dependencies.md`.

### if blocks

`if ... endif` is sugar: it is not a node, it simply ANDs its condition into the
dependencies of every enclosed entry. `if FOO` around ten symbols equals writing
`depends on FOO` on each of them.

### comment

`comment` emits a banner line in menuconfig and in the generated `.config`. It
has no value but can carry a condition:

```kconfig
comment "Tuning options (enable MYAPP_FEATURE to see more)"
	depends on !MYAPP_FEATURE
```

### source and rsource

```kconfig
source "Kconfig.zephyr"        # path resolves against $ZEPHYR_BASE
rsource "Kconfig.transport"    # path resolves against this file's directory
```

Use `rsource` to split a large app or module Kconfig into sibling files.

## Design rules

- Prompt every symbol a user should be able to set. Promptless symbols are
  driver-internal plumbing: only defaults and `select` can move them.
- Prefer `depends on` over `select`. `select` ignores the target's own
  dependencies and creates configurations the target's author never intended;
  a select-forced symbol also cannot be turned off from any `.conf` file.
- Reserve `select` for promptless helper symbols that have no dependencies of
  their own (the `MYAPP_TELEMETRY_CORE` pattern above).
- Use `imply` for soft enablement: "turning my feature on should turn X on by
  default, but the user may still say n".
- Order `default` lines most-specific-first: the first satisfied default wins.
- Make bools default `n` and let prj.conf or fragments opt in. Features that
  default `y` silently cost flash and RAM in every build.

## Namespace and discoverability

`config MYAPP_FEATURE` becomes `CONFIG_MYAPP_FEATURE` everywhere: in `.conf`
files, in `build/zephyr/.config`, and in C. App and module symbols share the one
global `CONFIG_` namespace with all of Zephyr, so prefix yours (`MYAPP_`,
`<MODULE>_`) to avoid collisions.

Your symbols appear in the menuconfig `/` search like any other symbol, but they
are NOT on the online docs site (docs.zephyrproject.org documents upstream
symbols only). Discovery for your users is menuconfig search and grep; see
`references/finding-options.md`.

## Adding defaults to existing symbols

A symbol may be defined in several places; an extra definition site can add
defaults without touching the original. Repeat the type, give no prompt, add
your default:

```kconfig
# In <app>/Kconfig, before source "Kconfig.zephyr"
config MAIN_STACK_SIZE
	int
	default 4096

source "Kconfig.zephyr"
```

Defaults accumulate in definition order, and the first satisfied default wins.

Gap-fill: because the application Kconfig root is parsed before
`source "Kconfig.zephyr"`, a default added there is seen before the upstream
one and wins when its condition holds. Verify in your checkout with the
menuconfig symbol info screen, which lists all defaults in order.

Do not overuse this: it changes behavior invisibly for anyone reading only
prj.conf. For one app, a plain `CONFIG_MAIN_STACK_SIZE=4096` line in prj.conf is
clearer; reserve extra definition sites for defaults that must stay conditional.

## Devicetree-derived defaults

Kconfig preprocessor functions named `dt_*` read the resolved devicetree, so a
default can follow the hardware:

```kconfig
config MYAPP_SYS_CLOCK_HZ
	int "System clock frequency"
	default $(dt_node_int_prop_int,/cpus/cpu@0,clock-frequency)
```

Gap-fill: this exact function appears in `Kconfig.defconfig` files under
`$ZEPHYR_BASE/soc/`; the full catalog of `dt_*` functions lives in
`$ZEPHYR_BASE/scripts/kconfig/kconfigfunctions.py`, so verify the name and
argument order there for your Zephyr version before relying on one.

Values produced this way move with overlays and board files, never with `.conf`
edits; the devicetree side belongs to the zephyr-devicetree skill.

## Module Kconfig

A west module contributes Kconfig through the path named in its
`zephyr/module.yml` (default `zephyr/Kconfig`):

```yaml
# <module>/zephyr/module.yml
build:
  kconfig: zephyr/Kconfig
```

That file is sourced into the main tree automatically for every module in the
workspace; module symbols behave exactly like app symbols. Module trees are
commonly guarded by a promptless root symbol so the options exist only when the
module is present:

```kconfig
config ZEPHYR_FOO_MODULE
	bool
```

If a fragment sets a module's symbol but the module is missing from the west
manifest, the symbol is undefined and the build aborts (unknown-symbol error;
see `references/debugging.md`).

Gap-fill: a module outside the manifest can be added per build with
`-DEXTRA_ZEPHYR_MODULES=<abs-path>` (alias `ZEPHYR_EXTRA_MODULES`); verify
against `$ZEPHYR_BASE/cmake/modules/zephyr_module.cmake` in your checkout.

## Verifying new symbols

Any change to a Kconfig source file triggers a re-merge of `.config` on the next
build (and discards menuconfig experiments). After editing:

```shell
west build -b <board> <app-dir>
grep "CONFIG_MYAPP" <build-dir>/zephyr/.config
```

Match both `CONFIG_X=` and `# CONFIG_X is not set` forms. To inspect prompts,
dependencies, and defaults interactively:

```shell
west build -t menuconfig --build-dir <build-dir>
```

then press `/` and search for your prefix.

## Out of scope

Board porting files (`<board>_defconfig`, board `Kconfig`, `Kconfig.defconfig`)
belong to the zephyr-build-west skill. Which subsystem options to combine for
BT, networking, or USB belongs to the zephyr-subsystems skill.

## Worked example

Goal: an optional sensor-sampling feature with a tunable, power-aware rate.

`<app>/Kconfig`:

```kconfig
mainmenu "My sensor application"

config MYAPP_USE_SENSOR
	bool "Enable periodic sensor sampling"
	help
	  Read the ambient sensor on a timer and log the values.

config MYAPP_LOW_POWER
	bool "Optimize for low power"

config MYAPP_SAMPLE_RATE_HZ
	int "Sample rate in Hz"
	depends on MYAPP_USE_SENSOR
	range 1 1000
	default 10 if MYAPP_LOW_POWER
	default 100
	help
	  How often to read the sensor. Kept low in low-power mode.

source "Kconfig.zephyr"
```

`prj.conf`:

```conf
CONFIG_MYAPP_USE_SENSOR=y
CONFIG_MYAPP_LOW_POWER=y
```

Resulting `<build-dir>/zephyr/.config` excerpt after `west build -b <board>`:

```conf
CONFIG_MYAPP_USE_SENSOR=y
CONFIG_MYAPP_LOW_POWER=y
CONFIG_MYAPP_SAMPLE_RATE_HZ=10
```

The rate is 10 because the first satisfied default wins. Add
`CONFIG_MYAPP_SAMPLE_RATE_HZ=250` to prj.conf and it becomes 250 (a user
assignment beats every default). Set it to 5000 and it is silently clamped to
1000 by the range, so never rely on clamping: write in-range values. Consume the
symbols from C with `IS_ENABLED(CONFIG_MYAPP_USE_SENSOR)` and
`CONFIG_MYAPP_SAMPLE_RATE_HZ` (see `references/config-from-c.md`).
