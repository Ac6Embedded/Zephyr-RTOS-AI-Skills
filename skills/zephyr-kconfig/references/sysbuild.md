# Sysbuild Kconfig: SB_CONFIG and Per-Image CONFIG

When to read this: the build directory contains `domains.yaml`, the project builds
more than one firmware image (application plus MCUboot, multi-core AMP images), or
you need to set an `SB_CONFIG_` symbol or configure exactly one image's `CONFIG_`
options. For per-image devicetree overlays, use the zephyr-devicetree skill.

## Two namespaces, never confused

Sysbuild (System build) lets one `west build` invocation configure and build
several Zephyr applications, called images. Each image Zephyr builds under
sysbuild is a "domain". Two completely separate Kconfig namespaces exist:

- `SB_CONFIG_*` configures sysbuild itself: which images exist, whether MCUboot
  is built, image-level wiring. Sysbuild's Kconfig tree is rooted at
  `$ZEPHYR_BASE/share/sysbuild/Kconfig`.
- `CONFIG_*` configures exactly one image. Every rule from the rest of this
  skill (merge order, fragment authoring, error severity) applies unchanged
  inside each image, independently of the others.

No `CONFIG_*` line can add or remove an image: which images exist is decided
only by `SB_CONFIG_*`. The classic mistake:

```conf
# WRONG: in <app>/prj.conf. This configures only the application image;
# it does not add an MCUboot image to the build.
CONFIG_BOOTLOADER_MCUBOOT=y
```

```conf
# RIGHT: in <app>/sysbuild.conf. This is a sysbuild decision.
SB_CONFIG_BOOTLOADER_MCUBOOT=y
```

`SB_CONFIG_BOOTLOADER_MCUBOOT` is a member of a bootloader choice in sysbuild's
Kconfig (`share/sysbuild/images/bootloader/Kconfig`), so normal choice semantics
apply: set the member you want to `y`, never set members to `n`
(references/symbols-and-dependencies.md).

## Where the configs live

Building with sysbuild produces a two-level tree. The top-level dir is a
controller; each image lives in its own complete build dir directly under it:

```shell
build/                    # controller: domains.yaml, its own build_info.yml
build/<app-name>/         # main application image: a normal Zephyr build dir
build/mcuboot/            # bootloader image (when enabled): another normal build dir
```

- The controller has NO `zephyr/.config`. If you grep a controller dir for
  `zephyr/.config` and find nothing, you are one level too high.
- Each image dir is a full, self-contained Zephyr build tree with its own
  `zephyr/.config` and its own `build_info.yml` (Zephyr 4.0+). `build_info.yml`
  exists at the sysbuild root AND per domain; only the per-image one carries
  that image's `cmake.kconfig.*` keys (references/build-artifacts.md).
- A directory is a sysbuild controller when `domains.yaml` exists in it, or its
  `build_info.yml` has a named `cmake.images[]` entry. Resolve an image's build
  dir as `<controller-dir>/<image-name>`; that composition holds by
  construction, while the absolute `build_dir` paths recorded in `domains.yaml`
  go stale when the tree is moved.

```shell
# The effective config of each image, independently:
grep "CONFIG_LOG[=\" ]" <build>/<app-name>/zephyr/.config
grep "CONFIG_LOG[=\" ]" <build>/mcuboot/zephyr/.config
```

Build with the `--sysbuild` flag:

```shell
west build -b <board> --sysbuild <app-dir>
```

Gap-fill (verify against your checkout's doc/build/sysbuild pages):
`west config build.sysbuild True` makes `--sysbuild` the default and
`west build --no-sysbuild` opts out for one invocation.

## Setting sysbuild symbols (SB_CONFIG)

Four ways, from durable to one-off:

```shell
# 1. <app>/sysbuild.conf: the sysbuild counterpart of prj.conf, picked up
#    automatically. FILE_SUFFIX applies to it too (sysbuild_<suffix>.conf).
# 2. Replace the file list explicitly:
west build -b <board> --sysbuild <app-dir> -- -DSB_CONF_FILE=<file>
# 3. Add extra fragments on top (SB_OVERLAY_CONFIG is the deprecated alias):
west build -b <board> --sysbuild <app-dir> -- -DSB_EXTRA_CONF_FILE=<file>
# 4. One-off CLI assignment:
west build -b <board> --sysbuild <app-dir> -- -DSB_CONFIG_BOOTLOADER_MCUBOOT=y
```

`sysbuild.conf` uses the same line grammar as any `.conf` file
(references/conf-syntax.md), just with the `SB_CONFIG_` prefix.

An application can extend sysbuild's Kconfig tree by providing
`<app-dir>/Kconfig.sysbuild`. That file becomes the sysbuild Kconfig root and
MUST source the stock tree or every standard `SB_CONFIG_` symbol disappears:

```kconfig
# <app-dir>/Kconfig.sysbuild
source "sysbuild/Kconfig"

config SB_MY_EXTRA_IMAGE_OPTION
	bool "Build the companion image"
```

## Configuring one image (CONFIG_)

Each image resolves its configuration from its own source tree, with the full
merge order of references/merge-order.md applied unchanged. That is why MCUboot
picks up configuration from the MCUboot source tree, not from your app, and why
editing `<app>/prj.conf` never changes the bootloader's `.config`.

To adjust another image without forking its source tree, use the `sysbuild/`
folder of the main application:

```shell
<app-dir>/sysbuild/mcuboot.conf          # Kconfig fragment merged into the mcuboot image
<app-dir>/sysbuild/<image-name>.conf     # same pattern for any image
<app-dir>/sysbuild/<image-name>/         # full replacement config dir for that image:
                                         # REPLACES the image's config dir, does not add
```

The single-file form adds a fragment on top of the image's own configuration.
The directory form replaces the image's application configuration directory
entirely: the image's own prj.conf and boards/ fragments stop applying, so
prefer the single-file form unless you really want a clean slate.

From the command line, prefix a variable with `<image-name>_` to aim it at one
image:

```shell
west build -b <board> --sysbuild <app-dir> -- -Dmcuboot_CONFIG_LOG=y
west build -b <board> --sysbuild <app-dir> -- -Dmcuboot_EXTRA_CONF_FILE=<file>
```

What belongs in such a fragment follows the normal authoring rules
(references/writing-fragments.md); error severity inside each image is also
unchanged, so an unknown symbol still aborts and an unmet dependency still only
warns (references/debugging.md).

## The three -D namespaces

One command line can carry all three; keep them apart:

| Form | Configures | Example |
| --- | --- | --- |
| `-DSB_CONFIG_<SYM>=y` | sysbuild itself (image set, bootloader) | `-DSB_CONFIG_BOOTLOADER_MCUBOOT=y` |
| `-DCONFIG_<SYM>=y` | the default (main) application image | `-DCONFIG_LOG=y` |
| `-D<image>_CONFIG_<SYM>=y` | exactly that image | `-Dmcuboot_CONFIG_LOG=y` |

## Interactive configuration

```shell
west build -t sysbuild_menuconfig --build-dir <controller-dir>
```

opens menuconfig on sysbuild's own tree (the `SB_CONFIG_` symbols). All UI
rules from references/menuconfig-guiconfig.md apply, including that edits are
temporary experiments to be persisted into `sysbuild.conf`.

For one image, run the target inside that image's build dir, which works
because an image dir is a normal Zephyr build dir:

```shell
west build -d <controller-dir>/<image-name> -t menuconfig
```

Recent Zephyr also creates image-prefixed targets runnable from the controller
build dir: each image gets `<image-name>_menuconfig` (for example
`mcuboot_menuconfig`). If the prefixed target is not present in the user's
version, the image-build-dir form above is always available.

## Worked example: enable and tune MCUboot

1. Enable the bootloader image, durably:

```conf
# <app-dir>/sysbuild.conf
SB_CONFIG_BOOTLOADER_MCUBOOT=y
```

2. Set one MCUboot option without touching the MCUboot source tree:

```conf
# <app-dir>/sysbuild/mcuboot.conf
CONFIG_LOG=y
```

3. Build and verify each image's effective config separately:

```shell
west build -b <board> --sysbuild <app-dir>
grep "CONFIG_LOG[=\" ]" build/mcuboot/zephyr/.config      # expect CONFIG_LOG=y
grep "CONFIG_LOG[=\" ]" build/<app-name>/zephyr/.config   # unaffected by step 2
```

4. A one-off equivalent of step 2, without the fragment file:

```shell
west build -b <board> --sysbuild <app-dir> -- -Dmcuboot_CONFIG_LOG=y
```

When changing which fragments feed a build, force a clean regeneration with
`west build -p` before trusting the result.

## Pitfalls

- `CONFIG_BOOTLOADER_MCUBOOT=y` in `prj.conf` when you meant to add MCUboot to
  the build: image existence is `SB_CONFIG_BOOTLOADER_MCUBOOT=y` in
  `sysbuild.conf`.
- Looking for `.config` at the controller root: it only exists per image, at
  `<controller-dir>/<image-name>/zephyr/.config`.
- Editing `<app>/prj.conf` and expecting another image's `.config` to change:
  per-image configs never leak. Use `sysbuild/<image>.conf` or
  `-D<image>_CONFIG_<SYM>=`.
- Using `<app>/sysbuild/<image-name>/` (the directory form) as if it added
  fragments: it replaces the image's whole config dir. For an additive tweak,
  use the single `<image>.conf` file.
- Writing `Kconfig.sysbuild` without `source "sysbuild/Kconfig"`: the standard
  `SB_CONFIG_` symbols vanish and every `sysbuild.conf` line becomes an
  unknown-symbol error.

## Out of scope

- Per-image devicetree overlays (`sysbuild/<image>.overlay`,
  `-D<image>_EXTRA_DTC_OVERLAY_FILE=`): the zephyr-devicetree skill.
- MCUboot signing, key management, and slot semantics: not Kconfig topics.
- Adding extra images to a project (`ExternalZephyrProject_Add`) and sysbuild
  CMake authoring: build-system topics.
