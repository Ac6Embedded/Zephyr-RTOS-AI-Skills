# /chosen, /aliases, and /zephyr,user

When to read this: you are pointing Zephyr at a device for a system role (console, shell UART, flash, entropy source), giving a node a portable name for application code, or passing application-specific values through the devicetree. Covers authoring all three special root nodes, the stable /chosen key subset, and how each node is consumed.

## The three nodes at a glance

| Node | Role | Property names | Typical value |
| --- | --- | --- | --- |
| `/chosen` | Zephyr-wide selections ("use THIS UART as the console") | fixed, version-specific Zephyr vocabulary | reference to a labelled node |
| `/aliases` | Portable shorthand names for nodes | entirely user-chosen, no curated list | reference to a labelled node |
| `/zephyr,user` | Application catch-all for arbitrary values | free-form | any devicetree value shape |

The devicetree specification binds neither `/aliases` nor `/chosen`, and `/zephyr,user` has no compatible, so none of the three has a binding YAML behind it. Nothing validates the property names: a misspelled key is not a build error, it simply never takes effect.

## Authoring form (all three)

These nodes carry no labels, so you cannot open them with `&chosen` or `&aliases`. In an overlay, declare them inside a root block; the build merges the block into the existing nodes:

```dts
/ {
    chosen {
        zephyr,console = &usart1;
    };
    aliases {
        led0 = &green_led;
    };
    zephyr,user {
        threshold-mv = <1250>;
    };
};
```

Rules:

- Root-block nesting is the canonical form. `&{/chosen}` path syntax is valid dtc but not idiomatic; do not use it.
- Values referencing nodes are written `name = &label;` (no angle brackets, no quotes).
- Merging follows the normal rules: when two files set the same key, the later file in apply order wins. Redeclaring a key in your app overlay is how you override a board default.
- Removal is explicit. Delete a key with `/delete-property/` inside the owning block:

```dts
/ {
    aliases {
        /delete-property/ led0;
    };
};
```

Delete verbs override definitions made earlier in the merge order (SoC dtsi, board DTS, earlier overlays). A key that exists only in your own overlay is removed by deleting the line, not with a verb.

For value formats, escaping, and general overlay mechanics, see references/overlay-authoring.md.

## /chosen

Each property selects one node for a Zephyr-wide role. Write the value as a reference to a labelled node:

```dts
/ {
    chosen {
        zephyr,console = &usart2;
        zephyr,shell-uart = &usart2;
    };
};
```

Selecting is not enabling. The chosen key only names the node: the node itself must still have `status = "okay"` (with working pinctrl where relevant), and the driver's Kconfig options must be on in `prj.conf`.

### The key catalog is version-specific

The canonical list of Zephyr chosen keys lives in your checkout, in the rST list-table titled "Zephyr-specific chosen properties" (columns Property and Purpose):

```shell
grep -n "list-table" "$ZEPHYR_BASE/doc/build/dts/api/api.rst"
grep -n -A 4 "zephyr,console" "$ZEPHYR_BASE/doc/build/dts/api/api.rst"
```

The same table is rendered on docs.zephyrproject.org (Build and Configuration Systems, Devicetree, API). Always trust the checkout over any snapshot, including the one below.

### Stable subset snapshot

Gap-fill: this subset was verified against the current upstream docs table (all eight keys present). The wider catalog changes across Zephyr versions, so double-check any key against your checkout before relying on it.

| Key | Purpose |
| --- | --- |
| `zephyr,console` | UART used by the console driver |
| `zephyr,shell-uart` | UART used by the serial shell backend |
| `zephyr,sram` | node whose `reg` gives the base address and size of SRAM available to the image |
| `zephyr,flash` | node whose `reg` sets defaults for flash configuration |
| `zephyr,code-partition` | flash partition the image's text section is linked into (see references/flash-partitions.md) |
| `zephyr,entropy` | device used as the system-wide entropy source |
| `zephyr,display` | default display controller |
| `zephyr,canbus` | default CAN controller |

Other keys you will meet in real trees include `zephyr,uart-mcumgr`, `zephyr,bt-mon-uart`, and `zephyr,bt-c2h-uart`; look them up in the catalog the same way.

### Consuming /chosen from C

Gap-fill (macro verified against the upstream devicetree API docs): `DT_CHOSEN(zephyr_console)` returns the node identifier for a chosen key. The argument is the key lowercased with non-alphanumeric characters (the comma, any dash) turned into underscores; cross-check the exact identifier in `build/zephyr/include/generated/zephyr/devicetree_generated.h` if in doubt.

```c
#include <zephyr/devicetree.h>

#define CONSOLE_NODE DT_CHOSEN(zephyr_console)
const struct device *console = DEVICE_DT_GET(CONSOLE_NODE);
```

Full C-side survival kit: references/dt-from-c.md.

## /aliases

Alias names are entirely user-chosen: there is no curated list. Their purpose is decoupling: application code names the alias, and the board (or your overlay) decides which node it points to. Boards commonly ship aliases such as `led0` and `i2c-0`, and upstream samples depend on them.

```dts
/ {
    aliases {
        sensor-uart = &uart1;
    };
};
```

To repoint an alias a board already defines, redeclare it in your overlay (the later file wins):

```dts
/ {
    aliases {
        led0 = &blue_led;
    };
};
```

### Consuming /aliases from C

Gap-fill (macro verified against the upstream devicetree API docs): `DT_ALIAS()` takes the alias name after dash-to-underscore conversion, so alias `sensor-uart` is read as `DT_ALIAS(sensor_uart)`.

```c
#include <zephyr/devicetree.h>

#define SENSOR_UART_NODE DT_ALIAS(sensor_uart)
const struct device *uart = DEVICE_DT_GET(SENSOR_UART_NODE);
```

If `DEVICE_DT_GET` on an alias target fails to link with an undefined `__device_dts_ord_<N>` symbol, the target node is disabled or has no driver bound: see references/debugging.md.

## /zephyr,user

The application catch-all: a conventional node with no compatible and no binding, where an application stores arbitrary values without writing a binding. It is often absent from a build entirely; it exists only when some file declares it. Gap-fill (verified against the upstream /zephyr,user docs page): it is meant for samples and applications, not for upstream drivers or subsystems.

Any value shape from the overlay serialization table works (see references/overlay-authoring.md), including boolean shorthand, where presence means true:

```dts
#include <zephyr/dt-bindings/gpio/gpio.h>

/ {
    zephyr,user {
        threshold-mv = <1250>;
        device-name = "unit-a";
        debug-mode;
        dac = <&dac1>;
        status-gpios = <&gpioa 5 GPIO_ACTIVE_HIGH>;
    };
};
```

Notes:

- The `#include` is needed only when a value uses macros such as `GPIO_ACTIVE_HIGH`.
- Name GPIO properties `<something>-gpios`; the GPIO helper macros require that suffix (details in references/dt-from-c.md).

### Consuming /zephyr,user from C

Gap-fill (all macros verified against the upstream /zephyr,user docs page):

```c
#include <zephyr/devicetree.h>
#include <zephyr/drivers/gpio.h>

#define ZEPHYR_USER_NODE DT_PATH(zephyr_user)

int threshold = DT_PROP(ZEPHYR_USER_NODE, threshold_mv);
const char *name = DT_PROP(ZEPHYR_USER_NODE, device_name);
bool debug = DT_PROP(ZEPHYR_USER_NODE, debug_mode);
const struct device *dac = DEVICE_DT_GET(DT_PROP(ZEPHYR_USER_NODE, dac));
const struct gpio_dt_spec status_led = GPIO_DT_SPEC_GET(ZEPHYR_USER_NODE, status_gpios);
```

## Verifying the result in a build

All three nodes obey the normal merge rules, and the final state is only knowable after a build. Read them straight out of the resolved tree:

```shell
grep -n -A 12 "chosen {" <build-dir>/zephyr/zephyr.dts
grep -n -A 12 "aliases {" <build-dir>/zephyr/zephyr.dts
grep -n -B 1 -A 10 "zephyr,user" <build-dir>/zephyr/zephyr.dts
```

An overlay edit is invisible until you rebuild. If a key you added is missing from `zephyr.dts`, the build is stale or your overlay was not consumed: see references/build-artifacts.md for how to list the overlays a build actually applied, and references/debugging.md for drift triage.

## Common mistakes

- Writing `&chosen { ... };` or `&aliases { ... };`: these nodes have no labels; use the root block form.
- Misspelling a chosen key: nothing errors, the selection silently never happens. Copy key names from the catalog in your checkout.
- Assuming a chosen key enables the device: it does not. Set `status = "okay"` and the subsystem `CONFIG_*` option as well.
- Pointing an alias at a disabled node and then calling `DEVICE_DT_GET`: expect the `__device_dts_ord_<N>` link error at build time.
- Wrapping node references in brackets or quotes: write `led0 = &green_led;`, not `led0 = "&green_led";`.
