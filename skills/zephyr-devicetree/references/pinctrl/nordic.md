# Nordic nRF pinctrl

When to read this: you are assigning or moving pins on a Nordic nRF SoC (nRF52, nRF53, nRF54, nRF91 series), or decoding an existing `psels` value from a build. Read references/pinctrl/model.md first for the vendor-independent model (states, groups, conflicts); this page covers only what Nordic does differently.

> Gap-fill (whole page): unlike the other vendor pages, this page was not built from the collection's verified backbone. Every claim was instead verified against upstream Zephyr sources on the main branch: the `nordic,nrf-pinctrl` binding page on docs.zephyrproject.org, `include/zephyr/dt-bindings/pinctrl/nrf-pinctrl.h`, `dts/vendor/nordic/nrf_common.dtsi`, and the nRF52840 DK board files. Version-sensitive claims say so inline; double-check those against the user's own Zephyr checkout.

## Identification

- The first `/`-segment of the board qualifiers starts with `nrf`: `nrf52832`, `nrf52840`, `nrf5340/cpuapp`, `nrf5340/cpunet`, `nrf54l15/cpuapp`, `nrf9160`. On Zephyr 4.0 or newer, read `cmake.board.qualifiers` from `build/build_info.yml`; pre-build, the board target string carries the same segments (`west build -b nrf52840dk/nrf52840`).
- Board name patterns: `nrf*dk*` (nrf52840dk, nrf5340dk, nrf9160dk), `thingy52`, `thingy53`, plus many third-party boards built on nRF SoCs (`xiao_ble`, `adafruit_feather_nrf52840`, ...). When the board name does not say `nrf`, the qualifier still does.

## Where pin data lives

- There is no per-chip generated pin header (unlike STM32, NXP, or Silabs). One header serves every nRF SoC: `$ZEPHYR_BASE/include/zephyr/dt-bindings/pinctrl/nrf-pinctrl.h`. It defines the `NRF_PSEL` packing macro, all `NRF_FUN_*` function tokens, and the `NRF_DRIVE_*` drive-mode constants.
- Board pin assignments live in the board's pinctrl shell dtsi, for example `$ZEPHYR_BASE/boards/nordic/nrf52840dk/nrf52840dk_nrf52840-pinctrl.dtsi`, with all groups nested under `&pinctrl`.
- The pinctrl node itself is declared at the root of the tree in `$ZEPHYR_BASE/dts/vendor/nordic/nrf_common.dtsi` (path on main; older checkouts keep this file elsewhere, match by label, not path):

```dts
pinctrl: pin-controller {
    /* Virtual device: nRF pin control is distributed across GPIO
     * registers and per-peripheral PSEL registers. */
    compatible = "nordic,nrf-pinctrl";
};
```

  Label `pinctrl`, path `/pin-controller`, no `reg`. Always reference it as `&pinctrl`.
- Pads are named `P<port>.<pin>` (P0.06, P1.02); GPIO controllers are `gpio0`, `gpio1`, ... There are no alternate-function tables to consult: routing is close to a flat matrix, most functions can reach most pins (per-family exceptions in Quirks).

## Authoring format and macro

Group style. A `pinctrl-N` on the peripheral points at a state node under `&pinctrl`; its children `group1`, `group2`, ... each carry a `psels` array (required by the binding) plus electrical properties that apply to every pin in that group.

- The cell property is `psels`. Not `pinmux` (NXP, ESP32), not `pins` (Silabs).
- `NRF_PSEL(fun, port, pin)`: `fun` is a bare token pasted onto `NRF_FUN_`, so write `UART_TX`, never `NRF_FUN_UART_TX`. From the header:

```c
#define NRF_PSEL(fun, port, pin)                                    \
    ((((((port) * 32U) + (pin)) & NRF_PIN_MSK) << NRF_PIN_POS) |    \
     ((NRF_FUN_ ## fun & NRF_FUN_MSK) << NRF_FUN_POS))
/* NRF_PIN_POS 0, NRF_PIN_MSK 0x1FF, NRF_FUN_POS 24, NRF_FUN_MSK 0xFF */
```

  Bits 0-8 hold the absolute pin number (port * 32 + pin); bits 24-31 hold the function.
- Common function tokens (values from the main-branch header; the set grows per release, grep your checkout): `UART_TX` 0, `UART_RX` 1, `UART_RTS` 2, `UART_CTS` 3; `SPIM_SCK` 4, `SPIM_MOSI` 5, `SPIM_MISO` 6; `SPIS_*` 7 to 10; `TWIM_SCL` 11, `TWIM_SDA` 12; `PWM_OUT0` to `PWM_OUT3` 22 to 25; `SPIM_CSN` 78. Many more exist (I2S, PDM, QDEC, QSPI, GRTC, CAN, ...).
- `NRF_PSEL_DISCONNECTED(fun)` puts `NRF_PIN_DISCONNECTED` (0x1FF) in the pin field: it explicitly disconnects that signal. Use it when a driver expects the state entry but the wire is unused.
- The macro is a cell value, never a phandle: `psels = <NRF_PSEL(UART_TX, 1, 2)>;` is right, `<&NRF_PSEL(...)>` is wrong.
- After the build the macro is gone; `build/zephyr/zephyr.dts` stores packed integers. Decode: pin number = value & 0x1FF (port = n / 32, pin = n % 32); function = (value >> 24) & 0xFF.

```shell
grep -n "psels" build/zephyr/zephyr.dts
# psels = < 0x22 >;       decodes as NRF_PSEL(UART_TX, 1, 2): 1*32+2 = 0x22, fun 0
# psels = < 0x1000021 >;  decodes as NRF_PSEL(UART_RX, 1, 1): fun 1 << 24, pin 0x21
```

## Node shape example

Worked example (nRF52840 DK): enable UART1 on alternate pins, name it with an alias, use it from C.

Starting point on this board: the SoC dtsi declares `uart1` (compatible `nordic,nrf-uarte`) with `status = "disabled"`. The board DTS wires `pinctrl-0 = <&uart1_default>` and `pinctrl-1 = <&uart1_sleep>` (and labels the node `arduino_serial`) but does not enable it. The shipped groups, from `nrf52840dk_nrf52840-pinctrl.dtsi`:

```dts
uart1_default: uart1_default {
    group1 {
        psels = <NRF_PSEL(UART_RX, 1, 1)>;
        bias-pull-up;
    };
    group2 {
        psels = <NRF_PSEL(UART_TX, 1, 2)>;
    };
};
uart1_sleep: uart1_sleep {
    group1 {
        psels = <NRF_PSEL(UART_RX, 1, 1)>,
                <NRF_PSEL(UART_TX, 1, 2)>;
        low-power-enable;
    };
};
```

To keep the shipped pins (P1.01/P1.02), the minimal overlay is only `status` plus the alias. To move the pins (here TX to P1.07, RX to P1.08, both free header pins on the DK; check your own wiring), define fresh state nodes in `<app>/boards/nrf52840dk_nrf52840.overlay`:

```dts
#include <zephyr/dt-bindings/pinctrl/nrf-pinctrl.h>

&pinctrl {
    uart1_alt: uart1_alt {
        group1 {
            psels = <NRF_PSEL(UART_TX, 1, 7)>;
        };
        group2 {
            psels = <NRF_PSEL(UART_RX, 1, 8)>;
            bias-pull-up;
        };
    };
    uart1_alt_sleep: uart1_alt_sleep {
        group1 {
            psels = <NRF_PSEL(UART_TX, 1, 7)>,
                    <NRF_PSEL(UART_RX, 1, 8)>;
            low-power-enable;
        };
    };
};

&uart1 {
    status = "okay";
    current-speed = <115200>;
    pinctrl-0 = <&uart1_alt>;
    pinctrl-1 = <&uart1_alt_sleep>;
    pinctrl-names = "default", "sleep";
};

/ {
    aliases {
        sensor-uart = &uart1;
    };
};
```

In `prj.conf` (the UART driver is on by default for most Nordic boards, but state it explicitly):

```
CONFIG_SERIAL=y
```

C side: the alias `sensor-uart` becomes `sensor_uart` after dash-to-underscore conversion (see references/dt-from-c.md):

```c
#include <zephyr/device.h>
#include <zephyr/devicetree.h>
#include <zephyr/drivers/uart.h>

static const struct device *const sensor_uart =
    DEVICE_DT_GET(DT_ALIAS(sensor_uart));

int main(void)
{
    if (!device_is_ready(sensor_uart)) {
        return 0; /* triage in references/debugging.md */
    }
    uart_poll_out(sensor_uart, 'A');
    return 0;
}
```

Rebuild and confirm:

```shell
west build -p auto -b nrf52840dk/nrf52840 <app-dir>
grep -n -A6 "uart@40028000" build/zephyr/zephyr.dts
# expect status = "okay" and psels integers that decode to your pins
```

## Required includes

The macros come from `<zephyr/dt-bindings/pinctrl/nrf-pinctrl.h>`. On Nordic targets the SoC common dtsi (`nrf_common.dtsi`) already includes this header, so overlays typically compile even without an explicit `#include`. Add it anyway:

```dts
#include <zephyr/dt-bindings/pinctrl/nrf-pinctrl.h>
```

It is include-guarded (harmless twice) and keeps the overlay self-describing. This is the Nordic instance of the general macro-cells-need-the-header rule (references/overlay-authoring.md); contrast with STM32 (phandle refs, no include at all) and NXP/ESP32/Silabs (a per-chip header is genuinely required).

## Defaults and per-peripheral conventions

- Two states everywhere: slot 0 `"default"`, slot 1 `"sleep"`. Nordic board files define a sleep state for every routed peripheral; the sleep node merges all signals into one group with `low-power-enable` (input, buffer disconnected). Follow that pattern: the sleep state is applied when device power management (`CONFIG_PM_DEVICE=y`) suspends the peripheral.
- Group properties, from the `nordic,nrf-pinctrl` binding:

| Property | Type | Notes |
| --- | --- | --- |
| `psels` | array | Required; `NRF_PSEL(...)` / `NRF_PSEL_DISCONNECTED(...)` entries |
| `bias-disable`, `bias-pull-up`, `bias-pull-down` | boolean | Mutually exclusive |
| `low-power-enable` | boolean | Input with buffer disconnected (sleep groups) |
| `nordic,drive-mode` | int | Default 0 (`NRF_DRIVE_S0S1`) |
| `nordic,invert` | boolean | PWM outputs only, inverts the output at the pin |

- Drive modes are `NRF_DRIVE_<x><y>` tokens from the same header (S standard, H high, D disconnect, one letter per logic level): `NRF_DRIVE_S0S1` (default), `NRF_DRIVE_H0H1` (high drive both levels, fast SPI), `NRF_DRIVE_S0D1` (open-drain style). Nordic's I2C drivers commonly configure an open-drain drive themselves; set `nordic,drive-mode` only to override, and double-check the driver behavior on your checkout.
- UART/UARTE: put RX in its own group with `bias-pull-up` (idle-high line, avoids garbage on an unconnected pin); TX plain. Route RTS/CTS only with hardware flow control.
- TWIM (I2C): `TWIM_SCL` + `TWIM_SDA` in one group; Nordic DKs rely on external pull-ups.
- SPIM: `SPIM_SCK`/`SPIM_MOSI`/`SPIM_MISO` psels; chip select is usually not a psel: Nordic SPI nodes take `cs-gpios` (software CS). A hardware `SPIM_CSN` token exists for newer families; double-check that your SoC's SPIM supports it before routing it.
- PWM: `PWM_OUT0` to `PWM_OUT3`; use `nordic,invert` in the group for an inverted output pin.
- Which signals a peripheral family requires (SCL+SDA pair, TX/RX rules, ...) is cross-vendor: references/pinctrl/model.md.

## Quirks

- Shared instance base addresses (nRF52 series): several serial peripherals are one hardware block exposed under multiple DT nodes. On the nRF52840, `i2c0` and `spi0` both sit at `0x40003000`, `i2c1` and `spi1` at `0x40004000`. Enabling both nodes of a pair breaks at build or at boot: enable one, leave the other disabled. Grep the SoC dtsi for the unit address to see which labels collide on your chip.
- NFC pins: on nRF52 SoCs, P0.09/P0.10 default to the NFCT function, not GPIO. Repurposing them requires UICR configuration; in recent Zephyr this is a devicetree property on the UICR node (`nfct-pins-as-gpios`), in older trees a Kconfig option (`CONFIG_NFCT_PINS_AS_GPIOS`). Double-check which form your version uses.
- The routing matrix is near-flat but not unlimited: some families restrict certain functions to specific pins or low-leakage/clock pin sets (notably nRF54L), and datasheets recommend dedicated pins for high-frequency signals. The devicetree compiles either way; a bad choice surfaces only at runtime. Double-check the SoC datasheet.
- Node label vs path: label `pinctrl`, path `/pin-controller` (same split as ESP32). Always match by label.
- `pinctrl-names` pairing matters more than usual here: Nordic drivers look up the `"sleep"` state by name for power management, so a `pinctrl-1` without the matching name entry silently loses the sleep behavior.

## Out of scope

- SAADC analog inputs: ADC `channel@<reg>` children select inputs through channel properties (`zephyr,input-positive` with `NRF_SAADC_AIN*` constants from a separate header), not through `psels`. Child-binding mechanics: references/bindings.md.
- Legacy pre-pinctrl properties (`tx-pin`, `rx-pin` integers on the uart node) from Zephyr 2.x-era trees: not covered; migrate to `psels` states.
- UICR-level pin repurposing details beyond the NFC note above (reset pin, regulator settings): named here, not decoded.
- nRF5340 dual-core pin handover (`nordic,nrf-gpio-forwarder`, which core owns a pin): not covered here.
