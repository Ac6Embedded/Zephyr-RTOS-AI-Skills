# Devicetree Bindings

When to read this: you need to understand what a binding YAML says about a node,
find which binding a node matched, decode `reg`, `interrupts`, or `*-cells`
specifier properties, or write a minimal binding for custom hardware.

## What a binding is

A binding is a YAML file that gives structure to devicetree nodes: which
properties a node may or must carry, their types, and the meaning of specifier
cells. During CMake configuration the build system matches each node to a
binding, validates the node against it, and generates the C property macros
from it. A node with no matching binding still exists in the tree, but no
driver binds to it and most property macros are not generated.

## How nodes match bindings

Matching is by the node's `compatible` property. Each string is tried in
listed order; the first one that has a binding wins.

```dts
adc1: adc@40012000 {
    compatible = "st,stm32f4-adc", "st,stm32-adc";
    /* ... */
};
```

- Derivative-compatible gotcha: the node above matches through
  `st,stm32-adc.yaml`. There is no `st,stm32f4-adc.yaml`; the more specific
  compatible exists so a dedicated binding can be added later. When hunting for
  a node's binding, search every string in its `compatible` list, not just the
  first.
- Nodes without their own `compatible` can be covered by the parent binding's
  `child-binding:` (see below).
- Gap-fill: a binding may carry `on-bus: i2c` (or `spi`, ...) so the same
  compatible resolves to a different binding per bus; a bus-specific binding
  takes precedence over a bus-agnostic one. Details and bus-device authoring:
  references/buses-and-devices.md.

### Where binding files live

Bindings sit in `dts/bindings/` trees. The category is the subdirectory name
and is stable across vendors: `adc`, `serial`, `i2c`, `gpio`, `sensor`, `mtd`,
`base`, and so on. Gap-fill (verified against the Zephyr docs): file names
match the compatible by convention (`st,stm32-adc.yaml` for
`compatible: "st,stm32-adc"`).

```shell
# Find the binding file for a compatible string
grep -rl 'compatible: "<vendor>,<device>"' "$ZEPHYR_BASE/dts/bindings/"
```

After a build, the generated header records the exact binding file each node
matched; see references/build-artifacts.md.

## Anatomy of a binding file

```yaml
description: STM32 ADC

compatible: "st,stm32-adc"

include: [adc-controller.yaml, pinctrl-device.yaml]

properties:
  vref-mv:
    type: int
    default: 3300
    description: Reference voltage in millivolts
```

### include chains

`include:` pulls other bindings' content in before the file's own keys apply.
Base files like `base.yaml`, `pinctrl-device.yaml`, `uart-controller.yaml`
carry the standard properties, so most vendor bindings are thin layers on top
of an include chain. Three syntactic forms:

```yaml
include: base.yaml                       # single file
include: [base.yaml, pinctrl-device.yaml]  # list of files
include:
  - name: base.yaml                      # map form with property filters
    property-blocklist: [interrupt-parent]
  - name: adc-controller.yaml
    property-allowlist: ["#io-channel-cells"]
```

Gap-fill (verified against the binding loader): when several included files
declare the same property, their `required:` flags are OR'ed together, so
`required: true` wins; and a binding cannot relax an inherited
`required: true` back to `false` (that is a build error).

### Property specs

Each entry under `properties:` can declare:

```yaml
properties:
  <name>:
    type: <see type table below>
    required: true            # node must set it (when its status is okay)
    enum: [1, 2, 4, 8]        # value must be one of these
    const: 0                  # value is pinned to exactly this
    default: 3300             # used when the node omits the property
    description: what it means
```

- `required:` is enforced only for nodes whose effective status is okay, and
  for booleans "required" means "present". Full enforcement rules:
  references/validation-rules.md.
- `enum:` on `int` properties compares numerically, so `<0x4>` matches an enum
  listing `4`; string enums compare after quote stripping.
- `default:` never appears in the DTS source; it surfaces only in the build
  output and in C.

### child-binding

`child-binding:` describes children that have no `compatible` of their own:
ADC channels, PWM channels, pinctrl groups, partition entries.

```yaml
compatible: "st,stm32-adc"
child-binding:
  description: ADC channel
  properties:
    zephyr,gain:
      type: string
      required: true
```

`child-binding:` can nest two levels (`child-binding.child-binding:`). Pinctrl
controller bindings do this: the state node is level one and the group leaf
that owns the real properties is level two. See references/pinctrl/model.md.

## Property types and their DTS shapes

The binding `type:` decides the only legal DTS syntax for the value. Full
serialization and formatting rules: references/overlay-authoring.md.

| type | DTS shape |
| --- | --- |
| `string` | `label = "value";` |
| `int` | `clock-frequency = <16000000>;` |
| `boolean` | `hw-flow-control;` (presence is true, absence is false) |
| `array` / `int-array` | `pinmux = <A>, <B>;` (multi-cell groups keep their commas) |
| `uint8-array` | `local-mac-address = [DE AD BE EF 12 34];` |
| `string-array` | `pinctrl-names = "default", "sleep";` (never angle brackets) |
| `phandle` | `clock-source = <&pll>;` |
| `phandles` | `resets = <&rst_a &rst_b>;` |
| `phandle-array` | `cs-gpios = <&gpioa 4 GPIO_ACTIVE_LOW>, <&gpiob 0 0>;` (one group per controller) |
| `path` | `ref = &label;` or `ref = "/full/path";` (two shapes, never mixed) |
| `compound` | catch-all; copy the source shape verbatim |

## reg and cell counts

`reg = <address... size...>;` is split using the parent node's
`#address-cells` and `#size-cells`. Those two properties are inherited up the
tree, and a binding may pin them with `const:` (for example,
`adc-controller.yaml` in `$ZEPHYR_BASE/dts/bindings/adc/` pins
`#size-cells` to 0, so ADC channels are `reg = <address>` only).

Rules when you add nodes:

- Declare `#address-cells` and `#size-cells` explicitly on any container node
  you create; never rely on defaults you have not checked.
- The unit address (the part after `@` in the node name) mirrors the first
  `reg` cell. Radix does not matter for matching (`@3`, `@0x3`, and `@03` are
  the same node), but keep name and `reg` in sync.

```dts
&i2c1 {
    #address-cells = <1>;
    #size-cells = <0>;
    bme280@76 {
        compatible = "bosch,bme280";
        reg = <0x76>;   /* first reg cell == unit address 0x76 */
    };
};
```

## status semantics

- Any node may carry `status`. Absent means `okay`. Values seen in practice:
  `okay`, `disabled`, `reserved`, `fail`, `fail-sss`.
- Children usually omit `status` and are effectively enabled with their
  parent; a disabled parent makes every descendant unusable at boot even if a
  child says `okay`.
- `status` is orthogonal to configuration: a disabled node with full pinctrl
  routing is valid devicetree, just inert. Required-property checks only apply
  when the effective status is okay (references/validation-rules.md).

```dts
&usart1 {
    status = "okay";   /* enable a peripheral the board ships disabled */
};
```

## Specifier spaces and named cells

Phandle-array properties like `gpios`, `clocks`, `pwms`, `dmas`, and
`io-channels` carry extra cells after each controller reference. The number of
cells comes from the controller node (`#gpio-cells = <2>;` in DTS) and the
meaning of each cell comes from the controller binding's `*-cells:` list:

```yaml
# In the GPIO controller's binding
gpio-cells:
  - pin
  - flags
```

```dts
/* Consumer: cells read as pin=4, flags=GPIO_ACTIVE_LOW */
cs-gpios = <&gpioa 4 GPIO_ACTIVE_LOW>;
```

This is one generic mechanism: to know what `pwms = <&pwm0 0 20000 0>;` means,
open `pwm0`'s binding and read its `pwm-cells:` names. Never guess cell order;
it differs per controller.

## Interrupts

`interrupts` is resolved against the node's interrupt controller, found via
`interrupt-parent` (inherited up the tree, overridable per node). The cell
names come from the controller binding's `interrupt-cells:`, so the same
property reads differently per SoC:

- NVIC (Arm Cortex-M): `irq, priority`, so `interrupts = <37 0>;` is IRQ 37 at
  priority 0.
- GIC: `type, irq, flags`.
- PLIC (RISC-V): `irq`.

`interrupts-extended` names the controller inline per specifier
(`interrupts-extended = <&gpio0 5 0>;`), which lets one node target several
controllers. Check the controller's binding before editing either form.

## Classifying nodes in an unfamiliar tree

When walking a board's devicetree, sort nodes into four buckets:

- Housekeeping: `/cpus`, `/memory`, `/reserved-memory`, `/chosen`, `/aliases`,
  the root itself. Not devices.
- Containers: nodes that hold devices but are not devices, with compatibles
  like `simple-bus`, `simple-mfd`, `mmio-sram`, or no binding at all. Check
  the whole `compatible` list: a vendor SoC node may match a vendor binding
  first while listing `simple-bus` as fallback.
- Child-binding children: no own `compatible`, described by the parent
  binding's `child-binding:` (ADC channels, pinctrl groups).
- Compatible-bearing sub-devices: real devices nested under another device,
  such as `st,stm32-pwm` under an `st,stm32-timers` wrapper, or a sensor node
  on an I2C bus.

The distinction matters when editing: enable and configure the device node,
not its container; fill child-binding required properties on the child, not
the parent.

## Writing a minimal binding

Gap-fill (verified against docs.zephyrproject.org, bindings introduction and
syntax pages): use this when an app node needs validated custom properties and
`/zephyr,user` (references/chosen-aliases-user.md) is not enough.

Place the file under the application's own bindings tree; it is picked up
automatically, with no CMake changes:

```shell
mkdir -p <app>/dts/bindings/<category>
# e.g. <app>/dts/bindings/sensor/acme,widget.yaml
```

The build system searches the `dts/bindings/` subdirectories of the Zephyr
repository, the application source directory, the board directory, any shield
directories, any extra `DTS_ROOT` entries, and modules that declare a
`dts_root`. The file must be somewhere inside `dts/bindings/` (any
subdirectory depth); a YAML file elsewhere in the app is ignored.

Minimal template:

```yaml
description: Acme widget controller

compatible: "acme,widget"

include: base.yaml

properties:
  threshold-mv:
    type: int
    required: true
    description: Trigger threshold in millivolts
```

- Gap-fill (verified against the binding loader): `description:` is required
  for top-level bindings in current Zephyr (the loader errors without it);
  `child-binding:` blocks may omit it. Double-check against your checkout if
  it is old.
- `include: base.yaml` brings in the standard properties (`status`, `reg`,
  `compatible`, ...). Include the matching controller base
  (`i2c-device.yaml`, `spi-device.yaml`, `sensor-device.yaml`, ...) instead
  when the node sits on a bus.
- Name properties in lowercase with dashes, not underscores.
- Rebuild after adding the binding, then confirm the node matched it via the
  generated header (references/build-artifacts.md). A "no binding found"
  style warning at configure time means the compatible strings do not match;
  see references/debugging.md.

## Related and out of scope here

- Value formatting, deletions, and where overlay files go:
  references/overlay-authoring.md.
- `bus:` / `on-bus:` and adding devices on I2C or SPI:
  references/buses-and-devices.md.
- Enforcement details for required properties, enums, and new children:
  references/validation-rules.md.
- Out of scope: rules for contributing bindings upstream to the Zephyr
  project, nexus nodes (`gpio-map`, `interrupt-map`), and Kconfig gating
  generated from bindings (`DT_HAS_*`), which belongs to a Kconfig-focused
  skill.
