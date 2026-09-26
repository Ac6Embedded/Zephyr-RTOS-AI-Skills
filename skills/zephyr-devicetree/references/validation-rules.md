# Devicetree Validation Rules

When to read this: you are about to review a devicetree edit (yours or the user's) for
completeness and correctness, before or after a build. This file gives the rules a node must
satisfy against its binding, the hazards of adding child nodes, and why a clean build is not proof
the hardware will initialize.

## The two bars: binding-valid and boot-valid

Every devicetree edit has to clear two independent bars:

1. **Binding-valid**: the node satisfies its binding YAML (required properties present, values in
   the declared enums, correct types). The build enforces this; a failed check stops `west build`
   at configure time.
2. **Boot-valid**: the driver accepts the described configuration at runtime. Drivers often
   support a narrower set than the binding declares and reject the rest during init or the first
   API call. The build cannot catch this.

The authoritative binding validator is the build itself:

```shell
west build -b <board> <app-dir>
```

A clean configure proves binding-valid only. For boot-valid, see the last section and
`references/debugging.md`.

## Required properties

### Checks apply only when effective status is okay

Required-property enforcement is gated on status: a node whose effective status is `okay` must
carry every property its binding marks `required: true`; a disabled node's missing required
properties are not an error and not a boot problem.

Effective status means the node's own `status` and its ancestors: any node may carry `status`,
absent means `okay`, and children usually omit it, but a disabled ancestor makes the whole
subtree unusable regardless of what the children say.

```dts
&spi1 {
    status = "disabled";
    /* Required properties may be absent here; the build will not complain.
       The moment an overlay flips this node to "okay", every required
       property in the binding must be present. */
};
```

Practical consequence when reviewing: first establish the node's effective status in the resolved
tree, then check requirements. After a build:

```shell
grep -n -A 8 "<node-label-or-name>" <build-dir>/zephyr/zephyr.dts
```

Gap-fill: when a required property is missing on an okay node, the configure step fails with a
message of the shape `'<prop>' is marked as required in 'properties:' in <binding.yaml>, but does
not appear in <node>`. Double-check the exact wording against the user's Zephyr version; the
triage table in `references/debugging.md` covers the message families.

### Boolean "required" means present

A binding can mark a `boolean` property required. Booleans have no value payload in DTS; presence
is true, absence is false. So "required" for a boolean simply means the bare name must appear:

```dts
&gpioa {
    gpio-controller;    /* present and true; satisfies a required boolean */
};
```

Do not write `gpio-controller = <1>;` to "satisfy" a boolean; the bare-name form is the boolean
serialization (see the type table in `references/overlay-authoring.md`).

### status and compatible are special

Exclude `status` and `compatible` from any required-property sweep:

- `compatible` is what selected the binding in the first place; it is checked by matching, not as
  a required property.
- `status` is never required; absence means `okay`. Never add `status = "okay";` just to satisfy
  a checklist (add it when you actually need to enable a disabled node).

## Value and enum rules

### Integer literals and radix

Write `int` values as decimal or hex literals. The two radices are equivalent: `<13>` and `<0xd>`
are the same value, and any comparison (yours or the build's) must normalize radix before
deciding two values differ. Avoid leading zeros on decimal literals: C-style parsing treats them
as octal.

```dts
clock-frequency = <8000000>;    /* fine */
clock-frequency = <0x7A1200>;   /* same value, fine */
/* not "08000000": a leading zero reads as octal */
```

### Enum matching

Enums in a binding constrain the value set. The comparison rule depends on the property type:

- `int` enums compare **numerically with radix awareness**: `<0x4>` matches `enum: [2, 3, 4, 5]`
  because `0x4 == 4`. A naive string compare would wrongly reject the hex spelling.
- `string` enums compare **after quote-stripping**: the DTS value `"sram"` matches the YAML entry
  `sram` (or `"sram"`); the quotes are serialization, not part of the value.

```yaml
# binding excerpt
properties:
  prescaler:
    type: int
    enum: [2, 3, 4, 5]
  memory-region-kind:
    type: string
    enum: ["flash", "sram"]
```

```dts
prescaler = <0x4>;              /* valid: 0x4 is 4 */
memory-region-kind = "sram";    /* valid: quote-stripped match */
```

When an enum check fails at build time, print the value in both radices before concluding the
devicetree is wrong; see `references/debugging.md` for enum and const violation messages.

## Adding a new child node

Adding a child (an ADC channel, a sensor on a bus, a flash partition) has its own checklist.
Run all four checks every time:

1. The node name has a **non-empty unit name** (the part before `@`): `channel@3`, not `@3`.
2. The name does not **collide** with an existing child (see the silent-merge hazard below).
3. Every property the parent binding's `child-binding:` marks required is filled in.
4. If the child carries `reg`, the parent declares `#address-cells` and `#size-cells` (see
   below).

### The silent-merge hazard

Two same-name children under the same parent do not conflict and do not error: they **silently
merge** into one node, later file wins per property. This is how overlays extend shipped nodes
on purpose, and how an accidental name collision corrupts your edit without any warning.

```dts
/* Board dtsi already ships: */
&i2c1 {
    eeprom@50 {
        compatible = "atmel,at24";
        reg = <0x50>;
    };
};

/* Overlay: */
&i2c1 {
    eeprom@50 {                 /* same name: MERGES into the shipped node */
        status = "disabled";    /* this edits the existing eeprom */
    };
    sensor@50 {                 /* different name, same address: creates a
                                   SECOND node at 0x50, no merge */
        compatible = "vnd,sensor";
        reg = <0x50>;
    };
};
```

Rules that follow:

- To **edit** an existing child, reuse its exact name (unit name and address spelling included).
- To **replace** an existing child under a new name, `/delete-node/` the old one first in the
  parent's block (see `references/overlay-authoring.md`); otherwise both nodes exist.
- A renamed child does **not** replace the old one. Two differently named children sharing a unit
  address is a classic mistake. Gap-fill: `dtc` reports it as a duplicated unit address warning
  (`unique_unit_address`); verify the exact warning text against the user's toolchain, and see
  `references/debugging.md`.

### Children with reg: declare the parent's cell counts

`reg` is split into address and size fields using the parent's `#address-cells` and
`#size-cells`. Bindings can pin these with `const:` or supply a `default:`, but do not rely on
inheritance or fallbacks for nodes you add: **declare both explicitly on the parent** when you
add a reg-bearing child and the parent does not already set them literally.

```dts
&adc1 {
    #address-cells = <1>;
    #size-cells = <0>;          /* ADC channels have an address, no size */
    status = "okay";

    channel@3 {
        reg = <3>;              /* unit address mirrors the first reg cell */
        zephyr,gain = "ADC_GAIN_1";
        zephyr,reference = "ADC_REF_INTERNAL";
        /* Gap-fill: the generic ADC child-binding also requires
           zephyr,acquisition-time; read the child-binding in
           $ZEPHYR_BASE/dts/bindings/adc/ in the user's checkout for the
           authoritative required list on their version. */
    };
};
```

The unit address (`@3`) must mirror the first `reg` cell; a mismatch between the two is a
mistake even where the build tolerates it. For the full `reg` and cell-count semantics, see
`references/bindings.md`.

## Binding-valid is not boot-valid

Bindings describe what the devicetree machinery accepts. Drivers frequently hard-code a narrower
subset and reject the rest at boot, typically with `-EINVAL` from the configuring API call, even
though the build was clean.

Concrete example: the STM32 ADC binding accepts `differential`, any `zephyr,gain`, and any
`zephyr,reference` on channel children, but the driver (`drivers/adc/adc_stm32.c`) rejects
`differential`, any gain other than `ADC_GAIN_1`, and any reference other than
`ADC_REF_INTERNAL`, returning `-EINVAL` at channel setup. See `references/pinctrl/stm32.md` for
the full quirk list.

How to check boot-validity before flashing:

1. Find the driver source for the node's `compatible` under `$ZEPHYR_BASE/drivers/` and read the
   init and setup functions for explicit rejections of DT-described values.
2. Treat a runtime `-EINVAL` from a device API as "the driver rejected a devicetree-described
   configuration" and diff your values against what the driver's setup path accepts.

```c
/* A clean build plus this returning -EINVAL usually means a
   binding-valid but driver-rejected configuration: */
int err = adc_channel_setup_dt(&adc_channel);
if (err == -EINVAL) {
    /* re-read the driver's channel_setup for hard-coded constraints */
}
```

The full boot-failure decision tree (`device_is_ready()` false, link errors, `-EINVAL`) lives in
`references/debugging.md`.

## Where related checks live

This file covers node-against-binding rules. Adjacent checks are deliberately elsewhere:

- Which pins a peripheral family requires (I2C pair, UART TX/RX rules, CAN, SPI) and pin-conflict
  taxonomy: `references/pinctrl/model.md`.
- Value serialization by type (what a valid `phandle-array` or `uint8-array` looks like):
  `references/overlay-authoring.md`.
- Build-error message triage and the boot-failure decision tree: `references/debugging.md`.
- Bus-child specifics (I2C address in `reg`, SPI chip-select indexing):
  `references/buses-and-devices.md`.
