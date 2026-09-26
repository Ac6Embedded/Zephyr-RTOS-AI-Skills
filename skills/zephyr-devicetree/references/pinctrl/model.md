# Pinctrl: The Vendor-Independent Model

When to read this: you are assigning, moving, or auditing pins for any Zephyr peripheral, or investigating a suspected pin conflict. Read this first for the common model and rules, then continue to the matching vendor page (references/pinctrl/stm32.md, nxp.md, esp32.md, silabs.md, nordic.md) for macro formats and exact file paths.

## Vocabulary: pads and signals

- A pad is the physical I/O cell on the package: `PA2` (STM32), `PT0_14` (NXP), `GPIO18` (ESP32), `PC1` (Silabs).
- A signal is one alternate function that pad can carry: USART1 TX, I2C0 SCL, a PWM channel output.
- Pin muxing selects which signal a pad carries. A pad can physically support many signals; the devicetree records the one you picked. Before a build you can only enumerate possibilities; the built `build/zephyr/zephyr.dts` is what tells you what is actually wired.

## A pin assignment is a devicetree edit

Zephyr has no separate pin-assignment mechanism. Assigning a pin means editing the `pinctrl-N` property on the peripheral's node (or on a designated child, see "Wrapper peripherals" below) and, on group-style vendors, defining or editing the pin group node it points at.

```dts
&usart1 {
    pinctrl-0 = <&usart1_tx_pa9 &usart1_rx_pa10>;
    pinctrl-names = "default";
    status = "okay";
};
```

## States: pinctrl-N plus pinctrl-names

`pinctrl-0`, `pinctrl-1`, ... are states. `pinctrl-names` is a parallel string array indexed by position: entry N names state `pinctrl-N`. Zephyr's parser resolves states by position, so the pairing is structural, not cosmetic.

- Slot 0 is canonically `"default"`; the common second state is `"sleep"`.
- Gap-fill: drivers apply the `"default"` state during initialization; the `"sleep"` state is applied by device power management when enabled (`CONFIG_PM_DEVICE`). Double-check the exact trigger against your checkout's pinctrl documentation (`$ZEPHYR_BASE/doc/hardware/pinctrl/`) if it matters.
- A `pinctrl-N` without `pinctrl-names` is misconfigured: the driver cannot look the state up by name.
- Writing slot N greater than 0 requires names for every earlier slot as well.
- Removing the last `pinctrl-N` must also remove `pinctrl-names`: dangling names are invalid.

```dts
&lpuart1 {
    pinctrl-0 = <&pinmux_lpuart1>;
    pinctrl-1 = <&pinmux_lpuart1_sleep>;
    pinctrl-names = "default", "sleep";
};
```

## Does this peripheral take pinctrl at all?

A node supports pinctrl when either of these holds:

1. It (or its designated child) already carries a `pinctrl-N` property somewhere in the merged tree.
2. Its binding declares the pinctrl properties, normally via `include: pinctrl-device.yaml`.

```shell
# locate the binding for a compatible, then check it declares pinctrl
grep -rln '<vendor>,<device>' "$ZEPHYR_BASE/dts/bindings/"
grep -n 'pinctrl' "$ZEPHYR_BASE/dts/bindings/<category>/<vendor>,<device>.yaml"
```

If the binding never declares pinctrl in any form, the driver does not read pinctrl, and adding `pinctrl-0` to that node changes nothing. Example: `st,stm32-lptim` does not declare it in some Zephyr versions.

## Status is orthogonal to pinctrl

A `status = "disabled"` node is a valid pinctrl target. The routing is real devicetree, but it stays inert until the effective status is okay.

- Assigning pins does not enable the peripheral: also set `status = "okay"` (and the matching `CONFIG_<SUBSYS>=y` in `prj.conf`).
- Enabling does not assign pins: many SoC nodes ship both disabled and pin-less; the board DTS or your overlay must supply the pins.
- When auditing pin usage, include disabled nodes: their routings are still in the tree, and flipping status later activates whatever was already routed.

## The three authoring styles

Every covered vendor uses one of three styles. Identify the style, then read the vendor page for macros, paths, and conventions.

| Style | Used by | `pinctrl-N` points at | Pin data lives in |
| --- | --- | --- | --- |
| direct-ref | STM32 | per-signal labelled nodes shipped in the chip pinctrl dtsi | one node per signal, referenced as phandles |
| group | NXP MCX/Kinetis/S32/LPC, ESP32, Silabs Series 2 | a pin group node you write, holding `groupN` children | macro cell lists inside the `groupN` children |
| per-signal-node | NXP i.MX (including i.MX RT) | labelled nodes shipped in the chip dtsi, one per pad-function pair | a 5-int register tuple per node |

Gap-fill: Nordic nRF also uses the group style, with cell property `psels`; see references/pinctrl/nordic.md (that page is itself gap-filled and should be verified against the user's checkout).

### direct-ref example (STM32)

```dts
&usart1 {
    pinctrl-0 = <&usart1_tx_pa9 &usart1_rx_pa10>;
    pinctrl-names = "default";
};
```

The labels (`usart1_tx_pa9`) are defined by the chip's shipped pinctrl dtsi under `modules/hal/stm32/dts/st/<series>/`. No `#include` is needed: the cells are phandles to DT labels, not macros.

### group example (NXP shape)

```dts
#include <nxp/<family>/<CHIP>-pinctrl.h>

&pinctrl {
    pinmux_flexcomm0_lpuart: pinmux_flexcomm0_lpuart {
        group0 {
            pinmux = <FC0_P0_PIO0_16>, <FC0_P1_PIO0_17>;
            slew-rate = "fast";
            drive-strength = "low";
            input-enable;
        };
    };
};
```

The peripheral node then references the group: `pinctrl-0 = <&pinmux_flexcomm0_lpuart>;`. Group label conventions vary per vendor (NXP `pinmux_<peripheral>`, ESP32 `pinmux_<peripheral>_default`, Silabs `<peripheral>_default`): follow the vendor page, or imitate the board's shipped files.

### per-signal-node (NXP i.MX)

The chip dtsi (under `modules/hal/nxp/dts/nxp/nxp_imx/`) ships one labelled node per pad-function combination, each carrying a 5-int tuple of IOMUXC register values. Never write or decode the tuple by hand; reference the existing labels. Details in references/pinctrl/nxp.md.

## Macros are cell values, never phandles

A pinmux macro expands to a packed integer. It goes inside the angle brackets as a value:

```dts
group0 { pinmux = <FC0_P0_PIO0_16>, <FC0_P1_PIO0_17>; };   /* correct */
group0 { pinmux = <&FC0_P0_PIO0_16>; };                    /* WRONG: a macro is not a label */
group0 { pinmux = <FC0_P0_PIO0_16 FC0_P1_PIO0_17>; };      /* WRONG: commas removed, reads as one tuple */
```

Overlays whose cells use macros must `#include` the chip pinctrl header (see references/overlay-authoring.md for deriving the include path). STM32-style phandle references need no include.

## What the built zephyr.dts shows

Pinmux macros are C preprocessor symbols; after the build only the packed integer remains in `build/zephyr/zephyr.dts`.

- Group and per-signal-node vendors: grep the cell property name and read numeric values; the macro names are gone.
- STM32 direct-ref: node labels are preserved, so grep the label.

```shell
grep -n 'pinmux' <build-dir>/zephyr/zephyr.dts          # group vendors: numeric cells
grep -n 'usart1_tx_pa9' <build-dir>/zephyr/zephyr.dts   # STM32: labels survive
```

Consequence: comparing overlay text against `zephyr.dts` by string always mismatches on macro cells (`<FC0_P0_PIO0_16>` versus `<0x...>`). Compare resolved values instead; see references/debugging.md.

## Group mechanics

A pin group node contains one or more `groupN` children. Multiple children exist so different signals can carry different electrical properties (a push-pull TX versus an RX with a pull-up). Standard group-child properties:

`bias-pull-up`, `drive-open-drain`, `drive-push-pull`, `input-enable`, `output-high`, `output-low`, `slew-rate`, `drive-strength`

Vendor-specific extras exist (`nxp,passive-filter`, `silabs,input-filter`); see the vendor pages.

```dts
eusart1_default: eusart1_default {
    group0 { pins = <EUSART1_TX_PC1>; drive-push-pull; output-high; };
    group1 { pins = <EUSART1_RX_PC2>; input-enable; silabs,input-filter; };
};
```

The cell property name inside `groupN` varies: `pinmux` (NXP, ESP32), `pins` (Silabs Series 2), and (Gap-fill) `psels` (Nordic).

## The pinctrl parent

The pin controller's DT label is consistently `pinctrl`, but its path varies per vendor (`/pinctrl` on NXP, `/pin-controller` on ESP32). Always open it by label, never by path:

```dts
&pinctrl {
    my_uart_default: my_uart_default { /* groups here */ };
};
```

## Wrapper peripherals: pinctrl on a child

Two generic mismatches to expect on some vendors:

1. The signal token differs from the node label: STM32 timer signals are named `tim<N>_...` but the node label is `timers<N>`.
2. The labelled wrapper carries no `pinctrl-N`; it lives on a child: STM32 timers and Silabs timers/letimers both put pinctrl on the `pwm` child.

Convention: the pin group node keeps the wrapper peripheral's name (`timer0_default`, never `pwm_default`); only the `pinctrl-0` reference sits on the child.

```dts
&timer0 {
    status = "okay";
    pwm {
        pinctrl-0 = <&timer0_default>;
        pinctrl-names = "default";
        status = "okay";
    };
};
```

Exact node shapes are on references/pinctrl/stm32.md and references/pinctrl/silabs.md.

## Cross-vendor pin-set requirements

These hold for the peripheral family itself, on every vendor. Node label prefixes identify the family.

| Family (label prefixes) | Required pin set |
| --- | --- |
| `i2c`, `i3c` | SCL and SDA, always as a pair |
| `uart`, `usart`, `lpuart`, `eusart` | TX always required. RX required, unless `single-wire` is set. When `single-wire` is set, RX must NOT be pinned (TX and RX are internally connected on-chip). When `hw-flow-control` is set, CTS and RTS are both required. |
| `can`, `fdcan`, `twai`, `flexcan`, `mcan` | TX and RX, both required |
| `spi`, `lpspi` | SCK required if MOSI or MISO is present; SCK alone is legal (clock-only configuration) |

Property provenance: `single-wire` is defined by the `st,stm32-uart-base` and `nxp,lpuart` bindings; `hw-flow-control` is defined on the base `uart-controller` binding, so it applies to virtually every UART vendor. A node whose binding lacks the property never triggers that property's gated rule.

ADC channel children are named `channel@<reg>` across vendors: every Zephyr ADC binding (`st,stm32-adc`, `nxp,lpc-lpadc`, `nordic,nrf-saadc`, ...) enforces it.

```dts
&adc1 {
    channel@3 {
        reg = <3>;
        /* fill the channel binding's required properties: see references/bindings.md */
    };
};
```

Validation timing: these checks matter only when the node's effective status is okay; see references/validation-rules.md.

## Family naming variants

- UART goes by `uart`/`usart`/`lpuart` on STM32, `lpuart` on NXP, `uart` on ESP32, `usart` and `eusart` on Silabs.
- CAN is `twai` on ESP32, `can`/`fdcan` on STM32, `flexcan`/`mcan` on NXP.
- ESP32's SPI clock signal is `SCLK`, not `SCK`. Every other covered vendor uses `SCK`.

## Pin conflicts

Two kinds:

1. Duplicate role: the same peripheral signal routed to two pads (USART1 TX on both PA9 and PB6).
2. Pad over-subscription: one pad carrying signals from two peripherals at once.

The devicetree genuinely permits both, and the build will not error. Never resolve a conflict silently: report it and let the developer choose which routing to keep (a duplicate is usually a misconfiguration, occasionally a mid-refactor leftover). When a peripheral misbehaves electrically (silent output, garbage input), review the pinctrl groups of every enabled peripheral that can touch the suspect pad.

```shell
# group vendors: dump all cell lines, then look for a repeated packed value
grep -nE 'pinmux|pins|psels' <build-dir>/zephyr/zephyr.dts | sort | uniq -c | sort -rn | head
# STM32 direct-ref: find every reference to a pad's signal nodes by label
grep -n '_pa9' <build-dir>/zephyr/zephyr.dts
```

## Fallback recipe for any other vendor

For a vendor without a dedicated page (TI, Renesas, Raspberry Pi, Microchip, ...):

1. Open the board's shipped pinctrl file: `$ZEPHYR_BASE/boards/<vendor>/<board>/<board>-pinctrl.dtsi`.
2. Identify the style: phandle lists to per-signal labels (direct-ref), group nodes with `groupN` children holding macro cells (group), or labelled nodes with multi-int register tuples (per-signal-node).
3. Copy an existing entry for the most similar peripheral and modify it: swap the macro or label, keep the electrical properties of the nearest analogous signal (an output stays push-pull, an input keeps its pull and filter settings).
4. If macros are involved, copy the `#include` line from the top of that dtsi into your overlay.
5. Check the pin-set table above, rebuild, and confirm the result in `<build-dir>/zephyr/zephyr.dts`.

## Vendor identification and pages

The authoritative silicon selector is the first `/`-segment of `cmake.board.qualifiers` in `<build-dir>/build_info.yml` (requires Zephyr 4.0 or newer); it survives custom board names. Board-name prefixes are only a pre-build fallback.

| SoC qualifier or board patterns | Page |
| --- | --- |
| `stm32*`; boards `nucleo_*`, `disco_*`, `stm32*` | references/pinctrl/stm32.md |
| `mcxn*`, `mcxa*`, `mcxc*`, `mcxw*`, `mimxrt*`, Kinetis `mk*`, S32, LPC; boards `frdm_*`, `mcx*`, `mimxrt*` | references/pinctrl/nxp.md |
| `esp32*`; boards `esp32*`, `xiao_esp32*`, `m5stack_*`, `heltec_*` | references/pinctrl/esp32.md |
| `efr32*`, `efm32*`, `mgm*`, `bgm*` (the qualifier is the full part number, e.g. `efr32mg24b210f1536im48`); boards `xg2*`, `slwrb*`, `sltb*`, `slstk*` | references/pinctrl/silabs.md |
| `nrf51*`, `nrf52*`, `nrf53*`, `nrf54*`, `nrf91*`; boards `nrf*dk*` | references/pinctrl/nordic.md |
| Anything else | the fallback recipe above |

Out of scope for this page: the runtime pinctrl C API (`PINCTRL_DT_DEFINE`, switching states from code) and writing new pin controller drivers; vendor macro bit layouts live on the vendor pages.
