# Overlay Authoring

When to read this: you are writing or editing a `.overlay` file (syntax, value formats,
deletions, includes, merging into an existing overlay), or you need to decide which file
a devicetree change belongs in and how Zephyr picks overlays at build time.

## Block forms

An overlay is DTS syntax. It changes the final tree by re-opening nodes and setting,
adding, or deleting properties and children. There are two idiomatic block heads:

```dts
&usart1 {                 /* re-open an existing node by its label (preferred) */
    status = "okay";
};

/ {                       /* re-open the root */
    aliases {
        my-uart = &usart1;
    };
};
```

- `&{/soc/serial@40013800} { ... };` (path reference) is valid dtc syntax but rare;
  use it only when the target node has no label.
- A node without a label of its own is addressed by nesting under its nearest labelled
  ancestor, or under root:

```dts
&adc1 {
    channel@3 {           /* label-less child, reached through the labelled parent */
        zephyr,gain = "ADC_GAIN_1";
    };
};
```

- Root nesting is the canonical form for `/chosen`, `/aliases`, and `/zephyr,user`
  (see references/chosen-aliases-user.md):

```dts
/ {
    chosen {
        zephyr,console = &usart1;
    };
    zephyr,user {
        signal-gpios = <&gpioa 5 GPIO_ACTIVE_HIGH>;
    };
};
```

- Node head forms you may encounter when reading: `name {`, `label: name {`, and
  `label_a: label_b: name {` (multiple labels). The unit name is the last
  colon-delimited token.

## Value syntax by binding type

Write the DTS form that matches the property's binding `type:` (find the type in the
binding YAML; see references/bindings.md).

| Binding type | DTS form | Example |
| --- | --- | --- |
| `string` | quoted | `status = "okay";` |
| `int` | one cell | `clock-frequency = <400000>;` |
| `boolean` | presence shorthand | `hw-flow-control;` (absent means false) |
| `array` / `int-array` | cells in one group | `reg = <0x40011000 0x400>;` |
| `uint8-array` | bytestring, square brackets | `local-mac-address = [00 04 9f 05 22 71];` |
| `string-array` | comma-separated strings, NO angle brackets | `pinctrl-names = "default", "sleep";` |
| `phandle` | one label reference in cells | `interrupt-parent = <&nvic>;` |
| `phandles` | one group of references | `pinctrl-0 = <&usart1_tx_pa9 &usart1_rx_pa10>;` |
| `phandle-array` | one group per controller, comma-separated | `cs-gpios = <&gpioa 4 GPIO_ACTIVE_LOW>, <&gpiob 9 GPIO_ACTIVE_LOW>;` |
| `path` | `= &label;` or `= "/full/path";` | `zephyr,code-partition = &slot0_partition;` |
| `compound` | verbatim, copy the source shape | `interrupt-map = < ... >;` |

The two `path` shapes are mutually exclusive per assignment: a bare label reference
(no angle brackets) or a quoted absolute node path.

## Formatting rules that prevent silent breakage

- String arrays never use angle brackets:

```dts
pinctrl-names = "default", "sleep";     /* correct   */
pinctrl-names = <"default", "sleep">;   /* invalid   */
```

- Multi-cell-group arrays keep their commas. Each `< >` group is one logical tuple;
  collapsing groups changes the meaning:

```dts
pinmux = <FC0_P0_PIO0_16>, <FC0_P1_PIO0_17>;   /* two entries          */
pinmux = <FC0_P0_PIO0_16 FC0_P1_PIO0_17>;      /* ONE two-cell tuple   */
```

- Pinmux macros are cell values, never phandles: `pinmux = <FC0_P0_PIO0_16>;` is
  correct, `<&FC0_P0_PIO0_16>` is wrong (see references/pinctrl/model.md).
- Strings support only two escapes: `\\` and `\"`. DTS has no `\n` or Unicode escapes.
- Hex and decimal are equivalent after resolution: `< 13 2 >` and `< 0xd 0x2 >` are the
  same value. Match the radix convention of the surrounding file.
- Never write decimal literals with leading zeros: `<010>` parses as octal 8
  (C-style literal rules), a classic silent bug.
- Boolean shorthand means true. To turn a boolean off you must delete the property
  (see Deletions below); writing nothing does not override an earlier `flag;`.

## Deletions

Overlays never remove anything by omission. Removal is an explicit verb:

```dts
&i2c1 {
    /delete-property/ clock-frequency;   /* inside the owning node's block  */
    /delete-node/ eeprom@50;             /* inside the PARENT's block,      */
};                                       /* named by label or unit name     */
```

Rules:

- `/delete-property/ <name>;` goes inside the block of the node that owns the property.
- `/delete-node/ <label-or-unit-name>;` goes inside the block of the node's parent.
- Delete verbs act on whatever the merge has defined SO FAR: the SoC dtsi, the board
  DTS, and overlays applied earlier than yours. They cannot delete something a later
  file defines; if a later overlay re-adds the node, the re-add wins.
- To remove something you declared in the same overlay file, do not add a delete verb:
  just edit the declaration text out. Delete verbs are for overriding earlier files.
- After a build you cannot distinguish "deleted" from "never existed" in
  `build/zephyr/zephyr.dts`; keep the overlay as the record of intent.

## Includes

Overlays whose cells use macros must `#include` the header that defines them, at the
top of the file:

```dts
#include <zephyr/dt-bindings/pinctrl/esp32s3-pinctrl.h>

&pinctrl {
    /* groups using UART1_TX_GPIO17 etc. */
};
```

- Derive the exact header path from the vendor page (references/pinctrl/<vendor>.md).
  On Zephyr 4.0 or newer you can also match the header against the roots listed in
  `build_info.yml` under `cmake.devicetree.include-dirs` (see
  references/build-artifacts.md): the include path is the header's path relative to
  the longest matching root, exactly what `cpp` will resolve.
- STM32-style direct label references (`pinctrl-0 = <&usart1_tx_pa9>;`) need no
  include: they reference DT labels, not macros.
- Gap-fill: GPIO flag macros (`GPIO_ACTIVE_LOW`, ...) come from
  `<zephyr/dt-bindings/gpio/gpio.h>`; include it when your overlay uses them.
- When adding an include to an existing overlay, insert it after the last existing
  `#include` and never duplicate a line already present.

## Editing an existing overlay: merge behavior

- Re-open an existing labelled node with `&label { ... };`. Do not re-declare the node
  with its label under its parent again (`&pinctrl { grp: grp { ... }; };` when `grp`
  already exists elsewhere): that risks duplicate labels.
- If one file contains multiple blocks for the same label, they merge and the last
  occurrence wins for conflicting properties. Prefer editing the existing block in
  place over appending a duplicate: last-wins works but hides the effective value.
- Same-name children merge silently. Renaming a child does NOT replace the old one;
  it creates a sibling. The name must match exactly, including the unit-address
  spelling:

```dts
/* Board DTS defines channel@3. This creates a SECOND node at the same address: */
&adc1 {
    channel@0x3 { };    /* wrong: "channel@0x3" != "channel@3", dtc warns        */
};                      /* unique_unit_address; see references/debugging.md      */
```

- Everything you do not mention is preserved. An overlay block only touches the
  properties and children it names.
- An overlay edit is invisible until the application is rebuilt. To check what a build
  actually used, inspect `build/zephyr/zephyr.dts` (references/build-artifacts.md).

## Where to put the overlay

### Automatic overlay selection

Gap-fill (verified against docs.zephyrproject.org "Set devicetree overlays" and
`$ZEPHYR_BASE/cmake/modules/dts.cmake`, current as of Zephyr main in 2026; the exact
steps have changed across releases, so double-check `dts.cmake` in the user's checkout
on older trees). When `DTC_OVERLAY_FILE` is NOT set, the build system picks overlays
from the application source directory in this order:

1. If `socs/<SOC>_<BOARD_QUALIFIERS>.overlay` exists, use it.
2. If `boards/<BOARD>.overlay` exists, use it in addition.
3. If the board has revisions and `boards/<BOARD>_<revision>.overlay` exists, use it
   in addition.
4. If any file was found in steps 1 to 3, stop here.
5. Otherwise, if `<BOARD>.overlay` exists at the app root (legacy location), use it
   and stop.
6. Otherwise, if `app.overlay` exists, use it.

Filename composition for step 2: take the full board target minus the revision and
replace every `/` with `_`. Board target `nrf5340dk/nrf5340/cpuapp` gives
`boards/nrf5340dk_nrf5340_cpuapp.overlay`; a plain target like `nucleo_h563zi` gives
`boards/nucleo_h563zi.overlay`. The revision-less name is always among the candidates.

- Gap-fill (verified against the Zephyr application docs, "File Suffixes"): when
  `FILE_SUFFIX` is set, the build looks for suffixed variants such as
  `boards/<BOARD>_<suffix>.overlay` and falls back to the unsuffixed file when the
  suffixed one does not exist.
- Gap-fill: shields and snippets selected for the build add their own overlay files to
  the list; the docs state this for shields explicitly. You do not manage those files
  from the application.
- Under sysbuild, each image resolves overlays from its OWN source directory; see
  references/sysbuild.md.

### Overriding selection on the command line

Gap-fill (verified against docs.zephyrproject.org and the `list(APPEND dts_files ...)`
order in `cmake/modules/dts.cmake`): the two variables behave differently.

- `DTC_OVERLAY_FILE` REPLACES the automatic selection: only the files you list apply.
- `EXTRA_DTC_OVERLAY_FILE` APPENDS to whatever was selected (automatic or explicit),
  and appended files come last, so they win under later-wins merging.

```shell
# Add one overlay on top of the normal selection (usually what you want):
west build -b <board> <app-dir> -- -DEXTRA_DTC_OVERLAY_FILE=<path/to/extra.overlay>

# Replace the selection entirely (semicolon- or space-separated, later files win):
west build -b <board> <app-dir> -- -DDTC_OVERLAY_FILE="<first.overlay>;<second.overlay>"
```

Prefer `EXTRA_DTC_OVERLAY_FILE` for experiments: it keeps `app.overlay` and the board
overlay in effect. To confirm which overlays a finished build consumed, see
references/build-artifacts.md.

### Choosing a home for the change

| The change is | Put it in |
| --- | --- |
| Board-specific wiring for this app (pins, bus devices, enables) | `boards/<board>[_<qualifiers>].overlay` |
| Board-independent app config (`/aliases`, `/chosen`, `zephyr,user`) | `app.overlay` |
| A one-off experiment you may throw away | a file passed via `EXTRA_DTC_OVERLAY_FILE` |
| Reusable plug-in hardware shared across apps | a shield (out of scope here) |
| A feature toggle applied across boards and apps | a snippet (out of scope here) |

Never edit shield, snippet, or module-shipped files to fix one application: those files
apply to every build that uses them. Author an app overlay that overrides them instead
(your overlay applies later and wins). Authoring shields and snippets themselves is out
of scope for this skill (planned `zephyr-build-west` skill).
