# Contributing

This file holds the authoring conventions for the skills in this repository: how the collection is split, how each skill is structured, the vendor page template, and the verification policy. Read it before adding or changing a skill.

## Collection map

Skills are named `zephyr-<domain>`. One topic lives in exactly one skill; skills cross-reference each other by name only.

| Skill | Status | Scope |
| --- | --- | --- |
| `zephyr-devicetree` | Available | Overlay/DTS/DTSI authoring, bindings, pinctrl (common model plus per-vendor pages), /chosen and /aliases, reading DT from C, flash partitions, build-artifact inspection, sysbuild, DT debugging |
| `zephyr-kconfig` | Available | Symbol semantics (types, prompts, depends on/select/imply, choices), .conf syntax, prj.conf and fragment creation with merge order, finding the right option, DT_HAS_* gating, menuconfig/guiconfig/hardenconfig, "symbol won't set" and provenance triage, Kconfig authoring for apps and modules, CONFIG_ in C, sysbuild Kconfig (SB_CONFIG, per-image config) |
| `zephyr-app-dev` | Planned | Application and source development: device model, driver usage and writing, threads and work queues, logging, CMakeLists |
| `zephyr-build-west` | Planned | West workspaces and manifests, build/flash/debug flows, shields and snippets authoring, twister, board porting |
| `zephyr-subsystems` | Planned | Settings, shell, sensors, networking, BLE, USB, filesystems |
| `zephyr-usermode` | Planned | User mode, syscalls, memory domains and partitions, MPU constraints |

## Boundaries and handoffs

Topics deliberately excluded from the available skills and where they will live:

From `zephyr-devicetree`:

- Kconfig interplay beyond one-line pointers (DT_HAS_* gating details, symbol dependency triage): `zephyr-kconfig`
- Device model internals, driver writing, the full C API: `zephyr-app-dev` (the devicetree skill keeps a survival kit for reading DT from C)
- Shields and snippets authoring, board porting (board.yml, defconfig): `zephyr-build-west`
- Per-subsystem devicetree conventions (sensors, display, USB, Bluetooth) beyond the generic bus-device pattern: `zephyr-subsystems`

From `zephyr-kconfig`:

- CMakeLists.txt authoring and the full C macro utility layer beyond the CONFIG survival kit: `zephyr-app-dev`
- Board porting defconfigs and board Kconfig files: `zephyr-build-west`
- Subsystem-specific option catalogs (which BT/networking/USB options to combine): `zephyr-subsystems`
- Devicetree-side fixes for DT-gated symbols (overlays, status okay): `zephyr-devicetree`

## Skill anatomy conventions

- `SKILL.md` is a lean router (under 300 lines): core model, first moves, and a routing table into `references/`.
- Reference files are 100 to 400 lines each, split by "when you need it", and each opens with a short "When to read this" preamble.
- Skills ship documentation only: no bundled scripts.
- Writing style: imperative and concrete; every rule is shown with a DTS or shell example; commands are copy-pasteable with `<angle-bracket>` placeholders; no em-dashes (use parentheses, colons, or commas).

## Vendor page template (pinctrl)

Vendor pinctrl pages live in `skills/zephyr-devicetree/references/pinctrl/` and follow this fixed section order, so future vendors (TI, Renesas, Raspberry Pi, Microchip, Infineon, ...) slot in consistently:

1. Identification (SoC qualifier prefixes, board name patterns)
2. Where pin data lives (exact paths, generated-by notes)
3. Authoring format and macro (style, macro signature, bit layout where known)
4. Node shape example (complete DTS: the pinctrl definition plus the peripheral's pinctrl-0/pinctrl-names)
5. Required includes
6. Defaults and per-peripheral conventions
7. Quirks
8. Out of scope (known different encodings, named explicitly so nothing gets guessed)

## Sources and verification policy

- The backbone knowledge was mined from a production devicetree tool and verified against its implementation.
- Content added from general Zephyr knowledge is flagged inline with `Gap-fill:` and should be double-checked against the user's Zephyr checkout when precision matters.
- Version-sensitive facts state their bound inline (for example: `build_info.yml` requires Zephyr 4.0+). Version-varying vocabularies (chosen keys, domains.yaml schema) are always read from the user's own checkout, never assumed.
