# Reading Devicetree from C

> Gap-fill: this entire file is general upstream Zephyr knowledge, not project-mined.
> Macro spellings, the `devicetree_generated.h` path, the `__device_dts_ord_NN`
> explanation, and the `DT_NODE_EXISTS` probe were verified against
> docs.zephyrproject.org (Devicetree API reference and troubleshooting pages).
> Version-varying details say so inline; when precision matters, confirm against
> `$ZEPHYR_BASE/include/zephyr/devicetree.h` in the user's checkout.

When to read this: you are writing or fixing application C code that consumes
devicetree data (get a device handle, read a property, blink an LED), or you hit
`undefined reference to __device_dts_ord_NN`. For authoring the DTS side, see
`references/overlay-authoring.md`; for boot-time failures, `references/debugging.md`.

## How devicetree reaches C

There is no devicetree parsing at runtime. The build compiles the merged DT and
generates one macro per fact into:

```
<build>/zephyr/include/generated/zephyr/devicetree_generated.h
```

The `<zephyr/devicetree.h>` API (`DT_*` macros) expands onto those generated
macros at compile time. Everything is resolved before the compiler runs, which is
why most mistakes surface as compile or link errors, not runtime errors.

Version note: the `zephyr/` subdirectory in that path is the current layout
(verified on the upstream troubleshooting page). Older Zephyr trees generated the
header directly under `include/generated/`. Locate it in any build with:

```shell
find <build>/zephyr/include/generated -name devicetree_generated.h
```

Include `<zephyr/devicetree.h>` for the `DT_*` macros and `<zephyr/device.h>` for
`DEVICE_DT_GET` and `device_is_ready`.

## Node identifiers

A node identifier is a preprocessor token that names a DT node. Get one with:

| Macro | Looks up | Example |
| --- | --- | --- |
| `DT_NODELABEL(label)` | a node label (`usart1: serial@...`) | `DT_NODELABEL(usart1)` |
| `DT_ALIAS(alias)` | a property of `/aliases` | `DT_ALIAS(led0)` |
| `DT_CHOSEN(prop)` | a property of `/chosen` | `DT_CHOSEN(zephyr_console)` |
| `DT_PATH(...)` | the full path, one argument per level | `DT_PATH(soc, serial_40001000)` |

Name conversion rule (applies to every argument above and to property names):
lowercase all letters and turn every non-alphanumeric character (`-`, `@`, `,`)
into `_`. So the alias `my-serial` becomes `DT_ALIAS(my_serial)`, the chosen key
`zephyr,console` becomes `DT_CHOSEN(zephyr_console)`, and node name
`serial@40001000` becomes the `DT_PATH` component `serial_40001000`.

```dts
/ {
    aliases { my-serial = &usart1; };
    chosen  { zephyr,console = &usart1; };
};
```

```c
/* All four refer to the same node; pick whichever you have. */
#define MY_SERIAL DT_NODELABEL(usart1)
#define MY_SERIAL DT_ALIAS(my_serial)
#define MY_SERIAL DT_CHOSEN(zephyr_console)
#define MY_SERIAL DT_PATH(soc, serial_40001000)
```

Prefer `DT_ALIAS` in application code (the app or board overlay controls the
alias, so the C code survives board changes). `DT_NODELABEL` is fine for
board-specific code. Avoid `DT_PATH` unless nothing else identifies the node.

## Probe before you use

A wrong identifier expands to an undefined token and produces cryptic errors
deep inside macro expansion. Fail early with the official probe (exact pattern
from the upstream troubleshooting page):

```c
#if !DT_NODE_EXISTS(DT_NODELABEL(my_serial))
#error "whoops"
#endif
```

Related checks:

- `DT_NODE_EXISTS(node_id)`: 1 if the node exists at all, enabled or not.
- `DT_NODE_HAS_STATUS(node_id, okay)`: 1 if it exists and is effectively enabled.
  The second argument is a token (`okay` or `disabled`), not a string.
- `DT_HAS_COMPAT_STATUS_OKAY(compat)`: 1 if at least one enabled node matches the
  lowercase-and-underscores compatible, e.g.
  `DT_HAS_COMPAT_STATUS_OKAY(bosch_bme280)`.

## Property access

```c
DT_PROP(DT_NODELABEL(usart1), current_speed)   /* int: 115200 */
DT_PROP(node_id, some_boolean)                 /* boolean: 0 or 1, absent == 0 */
DT_PROP_OR(node_id, clock_frequency, 100000)   /* fallback if prop absent */
```

- Property names follow the same lowercase-and-underscores conversion:
  `current-speed` is read as `current_speed`.
- Reading a property requires the node to have a matching binding that declares
  the property; it does not require `status = "okay"`.
- Do NOT read `reg` with `DT_PROP`. Use the dedicated macros, which split the
  cells using the parent's `#address-cells`/`#size-cells`:

```c
DT_REG_ADDR(DT_NODELABEL(usart1))   /* only register block: base address */
DT_REG_SIZE(DT_NODELABEL(usart1))   /* only register block: size */
```

For nodes with several register blocks use `DT_REG_ADDR_BY_IDX(node_id, idx)`
with a literal index.

## From node to struct device

Drivers allocate one `struct device` per enabled, driver-matched node. Fetch it
at build time and always check readiness before use:

```c
#include <zephyr/device.h>

static const struct device *const uart_dev = DEVICE_DT_GET(DT_CHOSEN(zephyr_console));

int init(void)
{
    if (!device_is_ready(uart_dev)) {
        return -ENODEV;   /* driver init failed or never ran */
    }
    /* safe to call the device's API now */
    return 0;
}
```

Variants:

| Macro | Argument | If no match |
| --- | --- | --- |
| `DEVICE_DT_GET(node_id)` | node identifier | link error `__device_dts_ord_NN` (see below) |
| `DEVICE_DT_GET_OR_NULL(node_id)` | node identifier | `NULL` when status is not okay |
| `DEVICE_DT_GET_ANY(compat)` | compatible token | `NULL`; arbitrary pick if several |
| `DEVICE_DT_GET_ONE(compat)` | compatible token | compile error; arbitrary pick if several |

`DEVICE_DT_GET` has zero runtime cost, but the pointer says nothing about whether
init succeeded: `device_is_ready()` is mandatory before first use.

## Spec-struct helpers (the daily pattern)

Driver classes ship `*_DT_SPEC_GET` macros that bundle the device pointer with
the cells of a phandle-array property into one static struct. Canonical GPIO use:

```dts
/ {
    leds {
        compatible = "gpio-leds";
        led0: led_0 { gpios = <&gpioa 5 GPIO_ACTIVE_HIGH>; };
    };
    aliases { led0 = &led0; };
};
```

```c
#include <zephyr/drivers/gpio.h>

static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios);

int main(void)
{
    if (!gpio_is_ready_dt(&led)) {
        return 0;
    }
    gpio_pin_configure_dt(&led, GPIO_OUTPUT_INACTIVE);
    gpio_pin_set_dt(&led, 1);   /* logical: turn the LED on */
    return 0;
}
```

`GPIO_DT_SPEC_GET(node_id, prop)` fills `.port` (controller device), `.pin`, and
`.dt_flags` from element 0 of the property; `GPIO_DT_SPEC_GET_BY_IDX` selects
another element, and `GPIO_DT_SPEC_GET_OR` takes a fallback initializer.

Same idea for the other classes:

| Helper | Reads | Struct highlights |
| --- | --- | --- |
| `PWM_DT_SPEC_GET(node_id)` | `pwms` on the consumer node | `.dev`, `.channel`, `.period`, `.flags` |
| `ADC_DT_SPEC_GET(node_id)` | `io-channels` on the consumer node (plus the controller's `channel@N` config) | `.dev`, `.channel_id`, `.channel_cfg` |
| `I2C_DT_SPEC_GET(node_id)` | the device node itself (its bus and `reg`) | `.bus`, `.addr` |
| `SPI_DT_SPEC_GET(node_id, operation, delay)` | the device node itself (bus, `spi-max-frequency`, `cs-gpios`) | `.bus`, `.config` |

Readiness: use `gpio_is_ready_dt(&spec)` and `spi_is_ready_dt(&spec)` where they
exist; otherwise `device_is_ready(spec.dev)` (or `spec.bus` for I2C/SPI) is
always correct. Note the argument difference: I2C/SPI specs take the identifier
of the device node on the bus (see `references/buses-and-devices.md` for
authoring those nodes), while GPIO/PWM/ADC specs take the consumer node plus,
for GPIO, the property name.

## GPIO consumer properties and active level

- Name GPIO phandle-array properties `gpios` or `<something>-gpios`
  (`cs-gpios`, `int-gpios`, `enable-gpios`). The `DT_GPIO_*` and
  `GPIO_DT_SPEC_*` machinery keys on that naming convention.
- Flag macros (`GPIO_ACTIVE_HIGH`, `GPIO_ACTIVE_LOW`, `GPIO_PULL_UP`, ...) live in
  `zephyr/include/zephyr/dt-bindings/gpio/gpio.h`. Board DTS files include this
  header, and all DTS sources are preprocessed together, so overlays can normally
  use the macros directly; if the build reports the symbol undefined, add
  `#include <zephyr/dt-bindings/gpio/gpio.h>` at the top of the overlay.
- `GPIO_ACTIVE_LOW` moves the inversion into the driver: the `_dt` calls speak
  logical levels. With `gpios = <&gpioa 5 GPIO_ACTIVE_LOW>;`,
  `gpio_pin_set_dt(&spec, 1)` means "active" and drives the wire LOW.
  `gpio_pin_get_dt` returns logical level too. Use the `_raw` variants only when
  you genuinely need the physical wire level.

Encode the polarity once, in the devicetree, and never write
`if (active_low) value = !value;` in application code.

## Application data: /zephyr,user

`/zephyr,user` is the binding-free node for app-specific data (authoring rules in
`references/chosen-aliases-user.md`). Access it with `DT_PATH(zephyr_user)`:

```dts
/ {
    zephyr,user {
        signal-gpios = <&gpioa 5 GPIO_ACTIVE_HIGH>;
        io-channels = <&adc1 3>;
        threshold = <500>;
    };
};
```

```c
#define ZEPHYR_USER DT_PATH(zephyr_user)

static const struct gpio_dt_spec sig = GPIO_DT_SPEC_GET(ZEPHYR_USER, signal_gpios);
static const struct adc_dt_spec  ch  = ADC_DT_SPEC_GET(ZEPHYR_USER);
static const int threshold = DT_PROP(ZEPHYR_USER, threshold);
```

## Link error: undefined reference to __device_dts_ord_NN

`DEVICE_DT_GET` expands to a reference to a device object named by the node's
dependency ordinal. The linker error means no driver ever created that object.
Three causes, checked in order:

1. The node is not enabled: it must have effective `status = "okay"`. Verify in
   the resolved tree, `<build>/zephyr/zephyr.dts` (see
   `references/build-artifacts.md`).
2. No driver matched: the node has no `compatible` with an in-tree driver, or no
   binding at all, so nothing called the device-definition machinery for it.
3. The driver is not compiled: its Kconfig options are off. Enable the subsystem
   and driver symbols in `prj.conf` (for example `CONFIG_I2C=y`); this is the
   most common cause.

Map the ordinal `NN` back to a node via the generated header's
"Node dependency ordering" comment, which lists `ordinal /node/path` pairs:

```shell
grep -n "Node dependency ordering" -A 400 \
  <build>/zephyr/include/generated/zephyr/devicetree_generated.h | grep " <NN> "
```

(Adjust the path for pre-move trees as shown in the first section.) Then confirm
that node's `status` and `compatible` in `<build>/zephyr/zephyr.dts`. If the
device exists but `device_is_ready()` returns false at boot, that is a different
failure class: follow the decision tree in `references/debugging.md`.

## Out of scope here

- Driver authoring: `DT_DRV_COMPAT`, the `DT_INST_*` family, and
  `DEVICE_DT_DEFINE` are driver-side machinery, planned for the `zephyr-app-dev`
  skill. `DT_INST(n, compat)` instance numbers are not stable across builds;
  application code should stick to aliases, labels, and chosen keys.
- Flash partition access from C (`FIXED_PARTITION_*`, `flash_area_open`):
  covered in `references/flash-partitions.md`.
- Runtime pinctrl state switching and power management APIs: `zephyr-app-dev`.
