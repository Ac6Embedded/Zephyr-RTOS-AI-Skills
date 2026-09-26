# Pinctrl: NXP (MCX, Kinetis, S32, LPC, i.MX)

When to read this: you are assigning or moving pins on an NXP SoC (MCX, Kinetis, S32, LPC, i.MX, i.MX RT). Read references/pinctrl/model.md first for the vendor-independent rules (states, group mechanics, pin-set requirements, conflicts); this page covers NXP file locations, macro formats, and conventions.

## Identification

The authoritative silicon selector is the first `/`-segment of `cmake.board.qualifiers` in `<build-dir>/build_info.yml` (requires Zephyr 4.0 or newer); board-name prefixes are only a pre-build fallback. See references/build-artifacts.md.

| Family | SoC qualifier examples | Board name patterns |
| --- | --- | --- |
| MCX (MCXN/MCXA/MCXC/MCXW) | `mcxn947` (from `mcxn947/cpu0/qspi`) | `frdm_mcx*`, `mcx*` |
| Kinetis (K, MK, MKW, MKE, MKV, K32) | `mk64f12`, `k32l2a41a` | `frdm_k*` |
| S32 (S32K1xx, S32K3xx, S32Z, S32M) | `s32k344` | `s32*` |
| LPC (LPC51, LPC54, LPC55: IOCON parts) | `lpc55s69` | `lpcxpresso*` |
| i.MX 6/7/8/9 and i.MX RT | `mimxrt1062` (from `mimxrt1060_evk/mimxrt1062/qspi`) | `mimxrt*`, `imx*` |

Gap-fill: the Kinetis, S32, LPC, and i.MX board-name patterns above are the common upstream naming; double-check against `$ZEPHYR_BASE/boards/nxp/` in the user's checkout. The MCX rows and both qualifier examples are exact.

## Where pin data lives

| Family | Pin data file | Macro |
| --- | --- | --- |
| MCX | `modules/hal/nxp/dts/nxp/mcx/<CHIP>-pinctrl.h` | `N9X_MUX(port_char, pin, mux)` |
| Kinetis | `modules/hal/nxp/dts/nxp/kinetis/<CHIP>-pinctrl.h` | `KINETIS_MUX(port_letter, pin, mux)` |
| S32 | `modules/hal/nxp/dts/nxp/s32/<CHIP>-pinctrl.h` | `KINETIS_MUX` (identical layout) |
| LPC | `modules/hal/nxp/dts/nxp/lpc/<CHIP>-pinctrl.h` | `IOCON_MUX(offset, type, mux)` |
| i.MX 6/7/8/9 | `modules/hal/nxp/dts/nxp/nxp_imx/<chip>-pinctrl.dtsi` | 5-int IOMUXC tuple (no macro) |
| i.MX RT 10xx/11xx | `modules/hal/nxp/dts/nxp/nxp_imx/rt/<chip>-pinctrl.dtsi` | 5-int IOMUXC tuple (no macro) |

- `modules/hal/nxp` sits relative to the west workspace topdir; the exact prefix varies with `west.yml`. On a built app, do not guess: `cmake.devicetree.include-dirs` in `build_info.yml` (Zephyr 4.0 or newer) lists every HAL DTS root the build used. See references/build-artifacts.md.
- The board's shipped pinctrl shell, `$ZEPHYR_BASE/boards/nxp/<board>/<board>-pinctrl.dtsi`, includes the chip header (or chip dtsi for i.MX) and defines the board's pin groups. It is the best template to imitate.
- Gap-fill: header file stems are full orderable part numbers including the package suffix (`MCXN236VDF-pinctrl.h`, not `MCXN236-pinctrl.h`). List the family directory to find the right one:

```shell
ls <west-topdir>/modules/hal/nxp/dts/nxp/mcx/ | grep -i <part-number-prefix>
```

## Authoring format and macro

MCX, Kinetis, S32, and LPC all use the group style (see references/pinctrl/model.md): `pinctrl-N` points at a pin group node you write under `&pinctrl`, whose `groupN` children carry `pinmux` cells. Each cell is one packed 32-bit macro. i.MX uses the per-signal-node style instead (subsection below).

Every macro header line has the same shape:

```c
#define FC0_P0_PIO0_16 N9X_MUX('0',16,2) /* PT0_16 */
```

- Symbol shape: `<PERIPHERAL>_<ROLE>_PIO<port>_<pin>` (here: Flexcomm 0, role P0, pad PIO0_16). A bare `PIO0_4` symbol is the plain-GPIO mux for that pad.
- The trailing comment carries the datasheet/silkscreen pad name (`PT0_16`). Prefer it when talking about physical pins; it is the name the schematic uses.

Bit layouts (what the macro packs into the cell):

| Macro | Layout | Notes |
| --- | --- | --- |
| `N9X_MUX(port, pin, mux)` | `((port - '0') & 0xF) << 28 \| (pin & 0x3F) << 22 \| (mux & 0xF) << 8` | `port` is a hex-digit char `'0'`..`'F'` |
| `KINETIS_MUX(port, pin, mux)` | `((port - 'A') & 0xF) << 28 \| (pin & 0x3F) << 22 \| (mux & 0x7) << 8` | `port` is a letter `'A'`..; mux is 3-bit (MCX is 4-bit) |
| `IOCON_MUX(offset, type, mux)` | `(offset & 0xFFF) << 20 \| (type & 0x3) << 18 \| (mux & 0xF)` | `IOCON_TYPE_D/I/A` = 0/1/2 |

Worked value: `N9X_MUX('0',16,2)` = `(0 << 28) | (16 << 22) | (2 << 8)` = `0x04000200`. That integer, not the macro name, is what appears in the compiled `build/zephyr/zephyr.dts`.

The macro is a cell value, never a phandle, and cells keep their commas:

```dts
group0 { pinmux = <FC0_P0_PIO0_16>, <FC0_P1_PIO0_17>; };   /* correct */
group0 { pinmux = <&FC0_P0_PIO0_16>; };                    /* WRONG: a macro is not a label */
group0 { pinmux = <FC0_P0_PIO0_16 FC0_P1_PIO0_17>; };      /* WRONG: reads as one tuple */
```

### i.MX: per-signal nodes, no macro decode needed

The i.MX chip dtsi ships one labelled node per (pad, peripheral, mux mode) combination, each carrying a 5-int tuple of IOMUXC register values:

```dts
/omit-if-no-ref/ iomuxc_gpio_ad_b0_00_lpi2c1_scls: IOMUXC_GPIO_AD_B0_00_LPI2C1_SCLS {
    pinmux = <0x401f80bc 5 0x0 0 0x401f82ac>;
};
```

Never write or decode the tuple by hand. Reference the existing labels. Label shape: `iomuxc_<pad>_<peripheral>_<signal>`, where the pad segment ends in a numeric token (`gpio_ad_b0_00` is the pad; `lpi2c1_scls` is the function). To find a label, grep the chip dtsi:

```shell
grep -i 'lpuart1' <west-topdir>/modules/hal/nxp/dts/nxp/nxp_imx/rt/<chip>-pinctrl.dtsi | grep -i tx
```

## Node shape example

Complete MCX example (matches the shipped `frdm_mcxn236` files): pin group plus the peripheral that consumes it. Both blocks can live in one app overlay.

```dts
#include <nxp/mcx/MCXN236VDF-pinctrl.h>

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

&flexcomm0_lpuart0 {
    pinctrl-0 = <&pinmux_flexcomm0_lpuart>;
    pinctrl-names = "default";
    current-speed = <115200>;
    status = "okay";
};
```

- Group label convention: `pinmux_<peripheral>`. NXP's shipped board files add a role suffix (`_lpuart`, `_lpi2c`) because one Flexcomm can serve several roles; the bare `pinmux_<peripheral>` form is also valid and complete.
- The default (and usually only) child is `group0`. Add `group1`, `group2`, ... only when different signals need different electrical properties (see references/pinctrl/model.md).
- Open the pin controller by label (`&pinctrl`), never by path.

i.MX example (matches the shipped `mimxrt1060_evk` files): the group's `pinmux` cells are phandle references to the shipped per-signal labels.

```dts
&pinctrl {
    pinmux_lpuart1: pinmux_lpuart1 {
        group0 {
            pinmux = <&iomuxc_gpio_ad_b0_13_lpuart1_rx>,
                     <&iomuxc_gpio_ad_b0_12_lpuart1_tx>;
            drive-strength = "r0-6";
            slew-rate = "slow";
            nxp,speed = "100-mhz";
        };
    };
};

&lpuart1 {
    pinctrl-0 = <&pinmux_lpuart1>;
    pinctrl-names = "default";
    status = "okay";
};
```

After a build, confirm the result in the resolved tree. Macro names are gone; only packed integers remain:

```shell
grep -n -A6 'pinmux_flexcomm0_lpuart' <build-dir>/zephyr/zephyr.dts
```

## Required includes

- MCX, Kinetis, S32, LPC: any overlay whose `pinmux` cells use macros must include the chip header:

```dts
#include <nxp/<family>/<CHIP>-pinctrl.h>
```

  where `<family>` is `mcx`, `kinetis`, `s32`, or `lpc`. The include is resolved against the HAL DTS roots recorded in `cmake.devicetree.include-dirs` (`build_info.yml`, Zephyr 4.0 or newer); copy the exact line from the board's shipped `-pinctrl.dtsi` when unsure. See references/overlay-authoring.md for include placement and deduplication.
- i.MX: no include is needed in an app overlay that only references existing `iomuxc_*` labels. The cells are phandle references, and the labels are already in the merged tree because the board's `-pinctrl.dtsi` includes the chip dtsi (`#include <nxp/nxp_imx/rt/<chip>-pinctrl.dtsi>` or the non-RT equivalent).

## Defaults and per-peripheral conventions

Group-style families (MCX, Kinetis, S32, LPC):

- Baseline electrical properties for a new group: `drive-strength = "low"` and `slew-rate = "fast"`. This is what NXP's own board files use for ordinary digital signals.
- `input-enable` goes on bidirectional or input-carrying peripherals only. NXP's shipped dtsi sets it for LPUART, LPI2C, LPSPI, CAN, I3C, FlexIO, and SAI groups, and skips it for output-only peripherals: PWM, ADC, COMP/CMP, DAC.

```dts
/* bidirectional: input-enable present */
group0 { pinmux = <FC2_P2_PIO4_2>, <FC2_P3_PIO4_3>; slew-rate = "fast"; drive-strength = "low"; input-enable; };
/* PWM output: no input-enable */
group0 { pinmux = <PWM0_A0_PIO1_2>; slew-rate = "fast"; drive-strength = "low"; };
```

- Extra properties are added per need, in the same group when all pins share them (the shipped `frdm_mcxn236` LPUART2 group adds `bias-pull-up`), or in a separate `groupN` when only some pins need them.
- Gap-fill: the `PWM0_A0_PIO1_2` symbol above is illustrative; look up real symbols in the chip header.

i.MX conventions (Gap-fill: taken from shipped Zephyr 4.x board files, `mimxrt1060_evk`; double-check property values against the user's board files): `drive-strength = "r0-6"`, `slew-rate = "slow"`, `nxp,speed = "100-mhz"` on ordinary signals. The value vocabularies are i.MX-specific (resistor-ratio drive strengths, speed grades); always copy the nearest analogous group from the board's shipped `-pinctrl.dtsi` rather than inventing values.

Pin-set requirements (which signals a UART, I2C, SPI, or CAN needs) are cross-vendor: see references/pinctrl/model.md. NXP naming there: UART is `lpuart`, SPI is `lpspi`, CAN is `flexcan`/`mcan`.

## Quirks

- LPC `IOCON_MUX` first argument is an IOCON register offset, NOT a per-port pin number. The pad identity comes from the trailing comment (`/* PIO0_0 */`); never compute a pad name from the offset.
- Kinetis/S32 mux field is 3 bits wide versus MCX's 4 bits; the port/pin fields are identical. Gap-fill: Kinetis/S32 symbols name the pad as `PT<port><pin>` at the end (`TSI0_CH1_PTA0`) instead of MCX/LPC's `PIO<port>_<pin>` suffix; verify against the chip header in the user's checkout.
- Gap-fill (verified against a Zephyr 4.x tree, double-check labels in the user's board dts): on MCX, one LP Flexcomm exposes its functions as separate DT nodes (`flexcomm0_lpuart0`, `flexcomm2_lpi2c2`). Put `pinctrl-0` and `status` on the function node you enable, not on the `flexcomm0` wrapper.
- The `nxp,port-pinctrl` binding nests two `child-binding:` levels: the `groupN` leaf is what declares `pinmux`, `drive-strength`, `slew-rate`, and `nxp,passive-filter`. Consequence: electrical properties belong on `groupN`, never on the `pinmux_<peripheral>` node itself.
- The pin controller's path is `/pinctrl` but tooling and overlays should match it by its label `pinctrl` (paths vary across vendors; see references/pinctrl/model.md).
- After a build, macro cells exist only as packed integers in `zephyr.dts`. Comparing overlay text against `zephyr.dts` by string always mismatches on macro cells; compare resolved values (see references/debugging.md).

## Out of scope

- RW610/RW612 `IO_MUX_*` macros (`modules/hal/nxp/dts/nxp/rw/RW61x-pinctrl.h`): multi-line composite macros with a different encoding; bit-level decode is not covered here. Authoring still works by copying entries from the board's shipped `-pinctrl.dtsi`.
- i.MX RT macro headers under `modules/hal/nxp/dts/nxp/nxp_imx/rt/*.h` (RT500/RT600 class parts): a distinct macro format; decode not covered. RT10xx/11xx authoring via the shipped dtsi labels IS covered above.
- Decoding the i.MX 5-int IOMUXC tuple: treat it as opaque; reference labels.
- For any of these, fall back to the generic recipe in references/pinctrl/model.md: imitate the board's shipped pinctrl file.
