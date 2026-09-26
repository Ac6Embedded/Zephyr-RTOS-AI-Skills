---
name: zephyr-devicetree
description: Zephyr RTOS devicetree work. Use when editing or creating .overlay, .dts, or .dtsi files; enabling or configuring peripherals (uart, i2c, spi, adc, pwm, status "okay"); assigning pins via pinctrl/pinmux on STM32, NXP, ESP32, Silabs, or Nordic; working with /chosen, /aliases, or zephyr,user; reading or writing devicetree bindings YAML; reading devicetree from C code (DT_NODELABEL, DEVICE_DT_GET); configuring flash partitions; inspecting a build's resolved devicetree (zephyr.dts, build_info.yml, sysbuild); or diagnosing devicetree build errors and device-not-ready failures.
---

# Zephyr Devicetree

Help the developer configure hardware through Zephyr's devicetree. Ground every answer in the user's own files: the app overlay, the board DTS, the bindings, and (when a build exists) the resolved devicetree.

## Core model

- The final devicetree is the merge of the SoC `.dtsi` chain, the board `.dts`, and every overlay, in apply order. When two files set the same property on the same node, the later file wins.
- What is actually configured is only knowable after a build: read `build/zephyr/zephyr.dts`. Before a build you can only enumerate possibilities; label such conclusions as pre-build.
- Overlays never remove anything by omission. Removal is explicit: `/delete-property/` and `/delete-node/`.
- `status` and pinctrl are orthogonal: a disabled node is a valid pinctrl target, and the routing stays inert until `status = "okay"`.
- Devicetree alone does not enable software. The subsystem toggle (`CONFIG_I2C=y`, `CONFIG_SPI=y`, `CONFIG_ADC=y`, `CONFIG_PWM=y`, `CONFIG_SENSOR=y`, ...) must be set in `prj.conf`. This is the most common cause of "I enabled it in DT and nothing happened".
- An overlay edit is invisible until the application is rebuilt.

## First moves

1. Locate the build directory (commonly `<app>/build/`). If it contains `domains.yaml`, it is a sysbuild controller, not an application build: see `references/sysbuild.md`.
2. If the app is built: grep `build/zephyr/zephyr.dts` for the node or label in question. That file is the resolved truth.
3. If not built: reason from the board DTS and the bindings, flag conclusions as pre-build, and suggest `west build` when the answer requires the resolved tree.
4. Identify the vendor before touching pins: on Zephyr 4.0+ read the first `/`-segment of `cmake.board.qualifiers` in `build/build_info.yml`; without a build, use the board name. Then load the matching vendor page.

## Vendor identification

| SoC qualifier or board patterns | Vendor page |
| --- | --- |
| `stm32*`; boards `nucleo_*`, `disco_*`, `stm32*` | `references/pinctrl/stm32.md` |
| `mcxn*`, `mcxa*`, `mcxc*`, `mcxw*`, `mimxrt*`, Kinetis `mk*`, S32, LPC; boards `frdm_*`, `mcx*`, `mimxrt*` | `references/pinctrl/nxp.md` |
| `esp32*`; boards `esp32*`, `xiao_esp32*`, `m5stack_*`, `heltec_*` | `references/pinctrl/esp32.md` |
| `efr32*`, `efm32*`, `mgm*`, `bgm*` (qualifier is the full part number); boards `xg2*`, `slwrb*`, `sltb*`, `slstk*` | `references/pinctrl/silabs.md` |
| `nrf51*`, `nrf52*`, `nrf53*`, `nrf54*`, `nrf91*`; boards `nrf*dk*` | `references/pinctrl/nordic.md` |
| Any other vendor | `references/pinctrl/model.md` (fallback recipe: imitate the board's shipped `-pinctrl.dtsi`) |

## Routing

| Situation | Read |
| --- | --- |
| What a build dir contains; which overlays were applied; inspecting the resolved DT | `references/build-artifacts.md` |
| Writing or editing an overlay: syntax, value formats, deletions, includes, merging, which file to put it in | `references/overlay-authoring.md` |
| Adding a device on a bus (I2C sensor, SPI flash or display) | `references/buses-and-devices.md` |
| Understanding a binding YAML; reg and cell counts; interrupts; writing a minimal binding | `references/bindings.md` |
| `/chosen`, `/aliases`, `zephyr,user` | `references/chosen-aliases-user.md` |
| Assigning or moving pins; pinctrl states; which pins a peripheral family requires; pin conflicts | `references/pinctrl/model.md`, then the vendor page |
| Reading DT from C (`DT_NODELABEL`, `DEVICE_DT_GET`, `GPIO_DT_SPEC_GET`, ...) | `references/dt-from-c.md` |
| Flash partitions, MCUboot slots, `zephyr,code-partition` | `references/flash-partitions.md` |
| Checking an edit is complete and valid before or after building | `references/validation-rules.md` |
| Devicetree build errors; device not ready at boot; overlay and build disagree | `references/debugging.md` |
| Build dir has `domains.yaml` or multiple images | `references/sysbuild.md` |

## Non-negotiable rules

- `pinctrl-N` and `pinctrl-names` are parallel, position-indexed arrays; slot 0 is canonically `"default"`. Never write one without the other.
- Pinmux macros are cell values: `pinmux = <FC0_P0_PIO0_16>;`. Never phandles: `<&FC0_P0_PIO0_16>` is wrong.
- Multi-cell-group arrays keep their commas: `pinmux = <A>, <B>;` must not collapse to `<A B>` (that reads as one tuple).
- String arrays never use angle brackets: `pinctrl-names = "default", "sleep";`.
- Overlays whose cells use macros need the chip pinctrl `#include`; STM32-style direct label references need none.
- Required-property checks only matter when the node's effective status is okay; a disabled node's missing properties are not a boot problem.
- Never silently auto-resolve a pin conflict; report it and let the developer choose.
- When editing an existing overlay, re-open nodes by `&label`; duplicate blocks merge with last-wins, and same-name children merge silently (a renamed child does NOT replace the old one).
