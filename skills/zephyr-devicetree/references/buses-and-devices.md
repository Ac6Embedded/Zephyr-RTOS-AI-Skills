# Buses and Devices: Adding I2C and SPI Devices

When to read this: you are adding a device (sensor, EEPROM, flash, display
controller) as a child node of an I2C or SPI controller, you need to find the
right `compatible`, or the device you added does not show up. For muxing the
bus pins themselves, read `references/pinctrl/model.md` and the vendor page.

## The bus-device pattern

A device on a bus is a child node of the bus controller. It carries its own
`compatible` (so it gets its own binding and driver) and a `reg` whose meaning
depends on the bus: an address on I2C, a chip-select index on SPI. This is the
"compatible-bearing sub-device" pattern: the child is a real device, not a
fragment of its parent (unlike ADC channels or pinctrl groups, which inherit a
`child-binding:` and have no `compatible` of their own).

The generic recipe:

1. Find the device's `compatible` and read its binding (below).
2. Re-open the bus controller by `&label` in your overlay and add the child.
3. Make sure the controller's effective status is `okay`.
4. Enable the driver's subsystem in `prj.conf` (`CONFIG_I2C=y`, ...).
5. Rebuild, then confirm the child in `build/zephyr/zephyr.dts`.

## Finding the compatible and its binding

In-tree bindings live under `$ZEPHYR_BASE/dts/bindings/<category>/`, named
`<vendor>,<part>.yaml`. The category is the parent directory name (`sensor`,
`i2c`, `spi`, `mtd`, `display`, ...), which is stable across vendors.

```shell
# List binding categories
ls $ZEPHYR_BASE/dts/bindings/

# Find a part by name
ls $ZEPHYR_BASE/dts/bindings/sensor/ | grep -i <part-name>

# Find every binding file declaring a compatible
grep -rl "<vendor>,<part>" $ZEPHYR_BASE/dts/bindings/
```

West modules and the application can also ship bindings (`dts/bindings/`
under each). On Zephyr 4.0 or newer, the full search list a build actually
used is recorded in `build_info.yml` under `cmake.devicetree.bindings-dirs`
(see `references/build-artifacts.md`). Binding YAML semantics (property
types, `include:` chains, required properties) are covered in
`references/bindings.md`.

### One part, two buses: `on-bus:` matching

Gap-fill (verified against Zephyr main): a part that speaks both I2C and SPI
ships two binding files with the same `compatible`, and the build picks the
one whose `on-bus:` matches the bus of the parent controller. The BME280 is
the canonical example:

```yaml
# $ZEPHYR_BASE/dts/bindings/sensor/bosch,bme280-i2c.yaml
compatible: "bosch,bme280"
include: [sensor-device.yaml, i2c-device.yaml]   # i2c-device.yaml sets on-bus: i2c

# $ZEPHYR_BASE/dts/bindings/sensor/bosch,bme280-spi.yaml
compatible: "bosch,bme280"
include: [sensor-device.yaml, spi-device.yaml]   # spi-device.yaml sets on-bus: spi
```

So the same `compatible = "bosch,bme280";` line works on either bus; what
changes is the parent node and the child properties the matched binding
requires. Placing a device under the wrong controller type fails **silently**
at DT-generation time: no binding matches, the node gets no binding, and its
driver never materializes. Gap-fill: very recent Zephyr (devicetree scripts
since late 2025) can emit an on-bus mismatch warning, but only when the build passes
`--warn-bus-mismatch` via `EXTRA_GEN_EDT_ARGS`; do not rely on a build-time
warning (see `references/debugging.md`).

## I2C devices

Child shape (unit address = device address = first `reg` cell):

```dts
&i2c1 {
    status = "okay";

    bme280@76 {
        compatible = "bosch,bme280";
        reg = <0x76>;
    };
};
```

Rules:

- The unit address after `@` mirrors the first `reg` cell and is written as
  bare hex digits: `bme280@76` pairs with `reg = <0x76>`. Do not write
  `bme280@0x76`.
- Gap-fill (verified against `i2c-device.yaml`, Zephyr main): `reg` is
  required and is the device address on the bus, as the plain 7-bit address
  (not shifted, no read/write bit). Gap-fill: the BME280 answers at 0x76 or
  0x77 depending on its SDO strap; take the address from the datasheet and
  the board wiring.
- Bus speed lives on the controller, not the child:
  `clock-frequency = <I2C_BITRATE_STANDARD>;` with
  `#include <zephyr/dt-bindings/i2c/i2c.h>` at the top of the overlay
  (Gap-fill: macro and header verified in the FRDM-MCXN947 board files on
  Zephyr main).
- Two children with the same node name merge silently instead of erroring.
  Gap-fill: two different children with the same unit address surface at most
  as a dtc `unique_unit_address_if_enabled` warning, and only when both nodes
  are enabled and dtc is installed (Zephyr suppresses the plain
  `unique_unit_address` check, and dtc is an optional lint step). Check
  existing children before picking a name and address.

## SPI devices

On SPI, `reg` is not an address: it is the index into the controller's
`cs-gpios` array that selects this device's chip-select line. Gap-fill
(verified against `spi-controller.yaml` and `spi-device.yaml`, Zephyr main):
`cs-gpios` is a phandle-array on the controller, "the index in the array
corresponds to the child node that the CS gpio controls", and
`spi-max-frequency` (int, Hz) is required on every SPI child.

```dts
#include <zephyr/dt-bindings/gpio/gpio.h>

&spi1 {
    status = "okay";
    cs-gpios = <&gpio0 4 GPIO_ACTIVE_LOW>;

    bme280@0 {
        compatible = "bosch,bme280";
        reg = <0>;                      /* index 0 in cs-gpios */
        spi-max-frequency = <1000000>;  /* device limit, from the datasheet */
    };
};
```

Rules:

- The app or board must provide the `cs-gpios` entry for each child; if the
  board DTS already defines `cs-gpios`, remember that an overlay assignment
  replaces the whole array (later file wins). Repeat the existing entries and
  append yours: `cs-gpios = <&gpio0 4 GPIO_ACTIVE_LOW>, <&gpio1 2 GPIO_ACTIVE_LOW>;`.
- GPIO flag macros (`GPIO_ACTIVE_LOW`, ...) come from
  `<zephyr/dt-bindings/gpio/gpio.h>`; overlays using them need that include.
- A clock-only or CS-less setup is possible but driver-specific; check the
  controller and device bindings before omitting `cs-gpios`.

## Enablement and cell counts

- The controller's effective status must be `okay`. Boards often ship bus
  controllers disabled; add `status = "okay";` in the same overlay block.
- Children of a bus usually omit `status`. An absent `status` means okay, so
  a child of an enabled controller is enabled by default. A disabled parent
  makes every descendant unusable regardless of the child's own status.
- Required-property checks (like `spi-max-frequency`) only bite when the
  node's effective status is okay: see `references/validation-rules.md`.
- SoC dtsi files normally set the controller's `#address-cells` and
  `#size-cells` already. If you add a `reg`-bearing child to a parent that
  has no literal cell-count properties, declare them explicitly on the parent
  (Gap-fill: `#address-cells = <1>; #size-cells = <0>;` is the standard pair
  for I2C and SPI buses; double-check against the controller's binding in
  your checkout, which may pin them with `const:`).

## Worked example: BME280 on FRDM-MCXN947 (LPI2C)

Values below are verified against the board files on Zephyr main; re-check
against your checkout if it is older.

Board context: on NXP MCX, a FlexComm instance is a wrapper that can act as
UART, SPI, or I2C. The I2C function node of FlexComm 2 has the label
`flexcomm2_lpi2c2`, and the board dtsi attaches the extra label `arduino_i2c`
to it (it serves the Arduino header). On `frdm_mcxn947/mcxn947/cpu0`, both
`&flexcomm2` and `&flexcomm2_lpi2c2` are already `status = "okay"`, and the
board dtsi already wires pins and speed:

```dts
/* Shipped: zephyr/boards/nxp/frdm_mcxn947/frdm_mcxn947.dtsi (excerpt).
 * The real block head carries an extra label:
 * nxp_8080_touch_panel_i2c: &flexcomm2_lpi2c2 { ... }
 * and arduino_i2c is attached by a separate empty block:
 * arduino_i2c: &flexcomm2_lpi2c2 {};
 */
nxp_8080_touch_panel_i2c: &flexcomm2_lpi2c2 {
    pinctrl-0 = <&pinmux_flexcomm2_lpi2c>;
    pinctrl-names = "default";
    clock-frequency = <I2C_BITRATE_STANDARD>;
};
```

The referenced pinctrl group, from
`zephyr/boards/nxp/frdm_mcxn947/frdm_mcxn947-pinctrl.dtsi`, is NXP group
style (see `references/pinctrl/nxp.md`); note the I2C electrical properties
on the group child:

```dts
#include <nxp/mcx/MCXN947VDF-pinctrl.h>

&pinctrl {
    pinmux_flexcomm2_lpi2c: pinmux_flexcomm2_lpi2c {
        group0 {
            pinmux = <FC2_P0_PIO4_0>,
                     <FC2_P1_PIO4_1>;
            slew-rate = "fast";
            drive-strength = "low";
            input-enable;
            bias-pull-up;
            drive-open-drain;
        };
    };
};
```

Since the board already routes and enables the bus, the app overlay only adds
the sensor. Put it at
`<app>/boards/frdm_mcxn947_mcxn947_cpu0.overlay` (placement rules:
`references/overlay-authoring.md`):

```dts
&arduino_i2c {
    status = "okay";    /* redundant on this board, but explicit and harmless */

    bme280@76 {
        compatible = "bosch,bme280";
        reg = <0x76>;
    };
};
```

To move the bus to different pads instead, re-open the shipped group by label
and replace its `pinmux` cells. The overlay then needs the chip pinctrl
header, and the macros are cell values, never phandles (`<&FC2_...>` is
wrong):

```dts
#include <nxp/mcx/MCXN947VDF-pinctrl.h>

&pinmux_flexcomm2_lpi2c {
    group0 {
        /* Replace with the FC2_* macros for your pads; the header's
         * trailing comments carry the datasheet pad names. */
        pinmux = <FC2_P0_PIO4_0>,
                 <FC2_P1_PIO4_1>;
    };
};
```

Macro naming is `FC<instance>_P<function-pin>_PIO<port>_<pin>`; decode and
conventions are in `references/pinctrl/nxp.md`.

## The Kconfig one-liner

Devicetree alone enables nothing at the software level. For an I2C sensor:

```
CONFIG_I2C=y
CONFIG_SENSOR=y
```

For an SPI device, `CONFIG_SPI=y` plus the consuming subsystem (for flash
chips `CONFIG_FLASH=y`, for sensors `CONFIG_SENSOR=y`). Device-specific
driver symbols usually default on once the DT node exists and the subsystem
is enabled; symbol triage beyond this belongs to the planned
`zephyr-kconfig` skill.

## Verify after building

```shell
grep -n -A 4 "bme280@76" <build-dir>/zephyr/zephyr.dts
```

Expect the child under the bus controller with your `reg` value. If it is
missing, the overlay was not consumed or the app was not rebuilt: check the
`-- Found devicetree overlay:` build-log lines and
`references/build-artifacts.md`. From C, grab the device and gate on
readiness (details in `references/dt-from-c.md`):

```c
const struct device *dev = DEVICE_DT_GET_ONE(bosch_bme280);
if (!device_is_ready(dev)) {
    /* init failed, or the bus/pinctrl underneath is not ready:
       see references/debugging.md */
}
```

## Out of scope here

- Driver-specific child properties (`int-gpios`, `drdy-gpios`, per-part
  tuning knobs): read the part's binding YAML; the property-type syntax is
  in `references/bindings.md`.
- Sensor/display/USB/Bluetooth subsystem conventions beyond this generic
  bus-device pattern: planned `zephyr-subsystems` skill.
- Muxing the bus pins (SCL/SDA, SCK/MOSI/MISO pin sets and conflicts):
  `references/pinctrl/model.md` and the vendor pages.
- Packaging a bus device as a reusable shield: see the placement decision
  table in `references/overlay-authoring.md`.
