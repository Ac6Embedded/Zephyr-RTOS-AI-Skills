# Flash Partitions

> Gap-fill: this entire file is authored from upstream Zephyr knowledge rather than the verified backbone. Node placement and property shapes were checked against `$ZEPHYR_BASE/dts/bindings/mtd/fixed-partitions.yaml`, and the C macro spellings against `$ZEPHYR_BASE/include/zephyr/storage/flash_map.h` (note: Zephyr 4.4 renamed `FIXED_PARTITION_*` to `PARTITION_*`, keeping the old names as deprecated aliases). Double-check version-sensitive details against the user's checkout.

**When to read this.** The task involves flash partition layout: resizing or adding partitions, MCUboot slot layout, `zephyr,code-partition`, storage for NVS/LittleFS/FCB, or reading a partition from C via the flash map API. For general overlay syntax and deletion semantics, read `references/overlay-authoring.md` first.

## Where fixed-partitions lives

The `fixed-partitions` node is a child of the **flash memory node** (the node with a size, typically compatible `soc-nv-flash` or a SPI NOR chip), NOT the flash controller that contains it. On most SoCs the memory node carries the label `flash0`:

```dts
/* Typical shipped shape inside the SoC/board dts (do not copy blindly) */
flash-controller@40023c00 {
    flash0: flash@8000000 {
        compatible = "soc-nv-flash";
        reg = <0x08000000 DT_SIZE_K(1024)>;

        partitions {
            compatible = "fixed-partitions";
            #address-cells = <1>;
            #size-cells = <1>;

            boot_partition: partition@0 {
                label = "mcuboot";
                reg = <0x00000000 0x00010000>;
                read-only;
            };
        };
    };
};
```

Rules for the container node:

- Compatible is exactly `"fixed-partitions"` (no vendor prefix).
- Always declare `#address-cells = <1>; #size-cells = <1>;` explicitly on the container you write. The binding types them as plain ints and does not pin the values; `<1>` is what internal-flash layouts use. If the shipped layout you are replacing uses different values, match them.
- The container's conventional node name is `partitions`, but verify against the user's board files before deleting by name:

```shell
grep -rn "fixed-partitions" $ZEPHYR_BASE/boards/<vendor>/<board>/
```

## Rules for partition children

Each partition is a child named `partition@<offset>`:

```dts
slot0_partition: partition@10000 {
    label = "image-0";
    reg = <0x00010000 0x00070000>;
};
```

- `reg = <offset size>;` is required. The offset is **relative to the flash memory node's base address** (so `partition@0` on a flash mapped at `0x08000000` starts at `0x08000000`), and the size is in bytes.
- The unit address (`@10000`) must mirror the first `reg` cell, written in hex without the `0x` prefix.
- `label` is an optional human-readable string. The C macros key on the **node label** (`slot0_partition:`), not on this string.
- `read-only;` is an optional boolean (presence means true); use it on bootloader partitions.
- Partitions must not overlap and must fit inside the flash. **The build does not check this**: the only related diagnostic is dtc's `unique_unit_address_if_enabled` warning, which fires only when two *enabled* children share the exact same unit address and only if dtc is installed at all (Zephyr suppresses the plain `unique_unit_address` check, and dtc is an optional lint step). A partial overlap or an out-of-bounds partition compiles silently. Do the arithmetic yourself (offset + size of each partition against the next offset and the flash size).
- Same-name children merge silently across the overlay chain. Redeclaring `partition@10000` without deleting first merges properties into the shipped node; redeclaring under a *new* unit address without deleting leaves the old partition in place next to yours. See `references/validation-rules.md`.
- Align partition boundaries to the flash erase-block size (check the flash node's `erase-block-size` property in `build/zephyr/zephyr.dts`); `flash_area_erase` operates on erase blocks.

## Standard partition labels (MCUboot and storage)

Zephyr and MCUboot find partitions by these **node labels**. Keep the names exactly; renaming them breaks MCUboot and storage-subsystem defaults.

| Node label | Conventional `label` string | Role |
| --- | --- | --- |
| `boot_partition` | `"mcuboot"` | The MCUboot bootloader itself |
| `slot0_partition` | `"image-0"` | Primary slot: the application runs from here |
| `slot1_partition` | `"image-1"` | Secondary slot: upgrade candidate |
| `scratch_partition` | `"image-scratch"` | Swap scratch (only for swap-using-scratch mode) |
| `storage_partition` | `"storage"` | Default backend for NVS, LittleFS, FCB, and settings |

Boards shipping MCUboot-ready layouts define all or most of these. An app that does not use MCUboot may reclaim `slot1_partition` and `scratch_partition` space entirely; the worked example below instead keeps the MCUboot slots, enlarges them, and reclaims the storage partition space.

## zephyr,code-partition

The `zephyr,code-partition` chosen property selects which partition the application image is linked into (link address and maximum size):

```dts
/ {
    chosen {
        zephyr,code-partition = &slot0_partition;
    };
};
```

- It only takes effect when `CONFIG_USE_DT_CODE_PARTITION=y` is set in `prj.conf`. Without that symbol the image links at the flash base and the chosen entry is inert.
- MCUboot-chainloaded applications (`CONFIG_BOOTLOADER_MCUBOOT=y`) get this wired up for you: that symbol arranges linking into `slot0_partition` including the MCUboot image-header offset. Double-check the exact Kconfig plumbing against the user's checkout; details beyond this pointer belong to the Kconfig side.
- Chosen syntax and the rest of the chosen catalog: `references/chosen-aliases-user.md`.

## Replacing a shipped layout (worked example)

Scenario: the board ships a four-partition MCUboot layout on a 1 MB `&flash0`, and the app needs bigger slots with storage moved off-chip. You cannot shrink or remove shipped partitions by omission; overlays only add or override. Delete, then redeclare.

Put the overlay in `<app>/boards/<board>.overlay` (placement rules: `references/overlay-authoring.md`):

```dts
/* boards/<board>.overlay : replace the shipped partition layout */

&flash0 {
    /delete-node/ partitions;

    partitions {
        compatible = "fixed-partitions";
        #address-cells = <1>;
        #size-cells = <1>;

        boot_partition: partition@0 {
            label = "mcuboot";
            reg = <0x00000000 0x00010000>;
            read-only;
        };

        slot0_partition: partition@10000 {
            label = "image-0";
            reg = <0x00010000 0x00078000>;
        };

        slot1_partition: partition@88000 {
            label = "image-1";
            reg = <0x00088000 0x00078000>;
        };
    };
};

/ {
    chosen {
        zephyr,code-partition = &slot0_partition;
    };
};
```

Why this shape:

- `/delete-node/ partitions;` sits **inside the parent block** (`&flash0`) and names the child to remove. It erases the whole shipped container, including any unlabelled partitions you might otherwise miss. Deleting then redeclaring in the same block is valid: delete verbs act on everything defined earlier in the merge order.
- The redeclared container re-lists every partition the app still needs, with the arithmetic closed: `0x10000 + 0x78000 + 0x78000 = 0x100000` (exactly 1 MB, no gaps, no overlap).
- The chosen entry re-points `zephyr,code-partition` at the new `slot0_partition` phandle. If the shipped board dts already chose it, your overlay's assignment wins (later file wins).

Surgical alternative: to remove or resize only some partitions, delete by label reference at the top level of the overlay, then redeclare just what changed:

```dts
/delete-node/ &storage_partition;
/delete-node/ &slot1_partition;

&flash0 {
    partitions {
        slot1_partition: partition@88000 {
            label = "image-1";
            reg = <0x00088000 0x00078000>;
        };
    };
};
```

Caveats:

- A delete verb cannot remove nodes defined in overlays applied **after** yours; check the apply order if a partition survives (`references/build-artifacts.md` shows how to list applied overlays).
- Under sysbuild, MCUboot and the application are separate images with separate overlay sets; both must see the same layout, so mirror the overlay into each image's source dir. See `references/sysbuild.md`.
- On Nordic's NCS distribution, the Partition Manager (`pm.yml` / `pm_static.yml`) can override devicetree partitions entirely in multi-image builds; if partition edits seem ignored there, check for Partition Manager output before debugging the DT.

## Partitions on external flash

`fixed-partitions` works on any flash-like memory node, including SPI NOR chips (compatible `jedec,spi-nor`). Reference the chip by its node label:

```dts
&mx25r64 {
    partitions {
        compatible = "fixed-partitions";
        #address-cells = <1>;
        #size-cells = <1>;

        lfs_partition: partition@0 {
            label = "littlefs-storage";
            reg = <0x00000000 0x00200000>;
        };
    };
};
```

Adding the SPI flash chip itself (as a bus child with `reg` and `spi-max-frequency`) is covered in `references/buses-and-devices.md`.

## Inspecting the resolved layout

After a build, the merged layout is in `build/zephyr/zephyr.dts`:

```shell
grep -n -A 8 "fixed-partitions" <app>/build/zephyr/zephyr.dts
grep -n "partition@" <app>/build/zephyr/zephyr.dts
```

Verify: every expected partition present exactly once, offsets/sizes as intended, and no leftover shipped partition your delete missed. Partition edits are invisible until rebuilt (`references/debugging.md`).

## Reading partitions from C

Compile-time macros from `<zephyr/storage/flash_map.h>` key on the partition **node label**. Zephyr 4.4 renamed them: `FIXED_PARTITION_*` became deprecated aliases of new `PARTITION_*` macros (the old spellings still compile on 4.4+ but emit deprecation warnings and break `-Werror` builds). Pick the spelling matching the user's checkout:

| Zephyr 4.4+ | Before 4.4 | Yields |
| --- | --- | --- |
| `PARTITION_EXISTS(label)` | `FIXED_PARTITION_EXISTS(label)` | 1 if the partition node exists (the 4.4+ macro additionally requires the node to be **enabled**, not merely present) |
| `PARTITION_ID(label)` | `FIXED_PARTITION_ID(label)` | Numeric flash-area ID for `flash_area_open()` |
| `PARTITION_OFFSET(label)` | `FIXED_PARTITION_OFFSET(label)` | Offset within the flash device (first `reg` cell) |
| `PARTITION_SIZE(label)` | `FIXED_PARTITION_SIZE(label)` | Size in bytes (second `reg` cell) |
| `PARTITION_DEVICE(label)` | `FIXED_PARTITION_DEVICE(label)` | `const struct device *` of the backing flash device |

When unsure which generation the checkout expects, grep its header: `grep -n "define PARTITION_ID\|define FIXED_PARTITION_ID" $ZEPHYR_BASE/include/zephyr/storage/flash_map.h`.

Runtime access goes through the flash map API. Kconfig one-liner: `CONFIG_FLASH=y` and `CONFIG_FLASH_MAP=y` in `prj.conf`.

```c
#include <zephyr/storage/flash_map.h>

/* Zephyr 4.4+ spellings; on older trees use FIXED_PARTITION_EXISTS / FIXED_PARTITION_ID */
#if !PARTITION_EXISTS(storage_partition)
#error "storage_partition is missing (or disabled) in the devicetree"
#endif

static int read_storage(uint8_t *buf, size_t len)
{
    const struct flash_area *fa;
    int rc = flash_area_open(PARTITION_ID(storage_partition), &fa);

    if (rc != 0) {
        return rc;
    }
    rc = flash_area_read(fa, 0, buf, len); /* offset is partition-relative */
    flash_area_close(fa);
    return rc;
}
```

- `flash_area_open(uint8_t id, const struct flash_area **fa)` resolves the ID to a `struct flash_area` (fields include `fa_off`, `fa_size`, `fa_dev`); `flash_area_read/write/erase` then take offsets relative to the partition start.
- The general DT-from-C toolbox (node identifiers, `DEVICE_DT_GET`, readiness checks) is in `references/dt-from-c.md`.
- Macro history, oldest first: `FLASH_AREA_*` (keyed on the `label` string, deprecated in 2022 and later removed), then `FIXED_PARTITION_*` (keyed on the node label), then `PARTITION_*` since Zephyr 4.4. Check which generation the user's `flash_map.h` expects before advising.

## Out of scope

- MCUboot configuration, image signing, and upgrade modes: planned `zephyr-build-west` / MCUboot documentation.
- Mounting LittleFS/NVS/FCB on a partition and the storage APIs beyond `flash_area_*`: planned `zephyr-subsystems`.
- Nordic Partition Manager authoring (`pm.yml` semantics): NCS-specific, not covered here.
