# Sysbuild: Controller vs Image Build Dirs

When to read this: the build directory contains `domains.yaml`, or its `build_info.yml` lists `cmake.images[]`, or the project builds more than one firmware image (application plus MCUboot, multi-core AMP images). This page tells you how to find the real application build dir inside a sysbuild tree and how to aim a devicetree overlay at exactly one image.

## What sysbuild produces

Sysbuild (System build) lets one `west build` invocation configure and build several Zephyr applications, called images. Each image Zephyr builds under sysbuild is a "domain".

Gap-fill (verified against docs.zephyrproject.org/latest/build/sysbuild/): build with the `--sysbuild` flag, or make it the default:

```shell
west build -b <board> --sysbuild <app-dir>
west config build.sysbuild True   # make --sysbuild the default
west build --no-sysbuild ...      # opt out for one invocation
west build -b <board> --sysbuild <app-dir> -- -DSB_CONFIG_BOOTLOADER_MCUBOOT=y
```

The result is a two-level tree. The top-level dir is a controller; each image lives in its own complete build dir directly under it:

```shell
build/                    # controller: domains.yaml, its own build_info.yml
build/<app-name>/         # main application image: a normal Zephyr build dir
build/mcuboot/            # bootloader image (when enabled): another normal build dir
```

## The controller is not an application build

The controller's `build_info.yml` (present on Zephyr 4.0 or newer) describes the sysbuild CMake project itself, not your application:

- `cmake.application.source-dir` points at `$ZEPHYR_BASE/share/sysbuild`.
- There is no `cmake.zephyr.zephyr-base` key and no `cmake.devicetree` section.
- There is no `<controller>/zephyr/zephyr.dts`. The resolved devicetree only exists per image.

If you grep a controller dir for `zephyr/zephyr.dts` and find nothing, you are one level too high: descend into the image dir first.

## Detecting a sysbuild controller

A directory is a sysbuild controller when either marker holds:

1. `<dir>/domains.yaml` exists, or
2. `<dir>/build_info.yml` has a `cmake.images[]` list with a named entry.

```shell
ls <build-dir>/domains.yaml
grep -A6 '^  images:' <build-dir>/build_info.yml
```

Never key detection on a `cmake.sysbuild` value: whether that key appears (and on which level) varies across Zephyr versions. The two markers above are controller-level only, which also guarantees the nesting depth is exactly one: an image dir is never itself a controller, so you never need to recurse.

## domains.yaml

The controller's `domains.yaml` names the images:

```yaml
default: <app-name>
domains:
  - name: mcuboot
    build_dir: /abs/path/to/build/mcuboot
  - name: <app-name>
    build_dir: /abs/path/to/build/<app-name>
```

Rules for reading it:

- All fields are effectively optional and the schema varies across Zephyr versions: read the file from the user's own build, do not assume this exact shape.
- `build_dir` is an absolute path recorded at configure time. After the build tree is moved or copied it is stale. Resolve the image dir as `<controller-dir>/<image-name>` instead; that composition holds by construction.

```shell
# Correct, move-proof resolution of an image's build dir:
ls <controller-dir>/<image-name>/zephyr/zephyr.dts
```

## Picking the image to inspect

When a question arrives about "the build" and the dir is a controller, select the image in this order:

1. The domain the user explicitly named (the same name `west flash --domain <name>` takes).
2. The `default:` entry in `domains.yaml`.
3. The `cmake.images[]` entry whose `type` is `MAIN` (compare trimmed and case-insensitive).
4. If there is exactly one image, that image.

If none of these resolve, list the image names and ask which one the user means; do not guess between an app and its bootloader.

## Each image is a complete build dir

Every `<controller-dir>/<image-name>/` is a full, self-contained Zephyr build dir with its own:

- `build_info.yml` (Zephyr 4.0 or newer), including its own `cmake.devicetree.user-files` (the resolved overlay list for that image only)
- `zephyr/zephyr.dts` (that image's resolved devicetree)
- `CMakeCache.txt` and application source dir

Consequence: every inspection recipe in references/build-artifacts.md applies unchanged inside the image dir. For example, to check what the main app actually wired:

```shell
grep -n -A6 'status = "okay"' <controller-dir>/<app-name>/zephyr/zephyr.dts | head -40
```

Each image also resolves its own overlay set from its own source dir. On a multi-core board, the two images consume different files (for example `boards/<board>_<soc>_m7.overlay` in one app and `boards/<board>_<soc>_m4.overlay` in the other): an overlay placed for one image never leaks into another.

## Providing per-image overlays

Gap-fill (verified against docs.zephyrproject.org/latest/build/sysbuild/): three mechanisms exist. Double-check the exact precedence between them against the user's Zephyr checkout; after a build, the authoritative record is each image's own `build_info.yml` `cmake.devicetree.user-files` (Zephyr 4.0 or newer) or the `-- Found devicetree overlay:` lines in that image's section of the build log.

1. The image's own source dir. Each image follows the normal placement rules from references/overlay-authoring.md inside its own application tree (`boards/<board>.overlay`, `app.overlay`, ...). This is why MCUboot picks up configuration from the MCUboot source tree, not from your app.

2. The `sysbuild/` folder of the main application. Create a devicetree overlay or Kconfig fragment named after the target image:

```shell
<app-dir>/sysbuild/mcuboot.overlay       # overlay applied to the mcuboot image
<app-dir>/sysbuild/mcuboot.conf          # Kconfig fragment for the mcuboot image
<app-dir>/sysbuild/<image-name>/         # full replacement config dir: used as
                                         # APPLICATION_CONFIG_DIR for that image
```

   This keeps bootloader tweaks versioned next to your app without forking the bootloader. Note the directory form replaces the image's config dir rather than adding to it.

3. Namespaced CMake variables. Sysbuild prefixes a variable with `<image-name>_` to route it to one image:

```shell
west build -b <board> --sysbuild <app-dir> -- \
  -D<image-name>_EXTRA_DTC_OVERLAY_FILE=<abs-path-to>/my.overlay
```

   The general rule is `-D<image-name>_<VAR>=<value>` for CMake variables and `-D<image-name>_CONFIG_<SYMBOL>=<value>` for one image's Kconfig. As in a plain build, `EXTRA_DTC_OVERLAY_FILE` appends after the auto-detected overlays (so it wins on conflicts) while `DTC_OVERLAY_FILE` replaces the whole set; see references/overlay-authoring.md.

Do not confuse the namespaces: `-DSB_CONFIG_<SYMBOL>` sets sysbuild's own Kconfig (image selection, MCUboot on/off), not any image's application Kconfig.

## Building, flashing, and debugging one image

Gap-fill (verified against docs.zephyrproject.org/latest/build/sysbuild/): domain-aware west commands:

```shell
west flash --domain <image-name>
west debug --domain <image-name>
west build -d <controller-dir>/<image-name> -t <target>   # run one image's build target
```

The last form works because the image dir is a normal build dir; anything you would do to a single-app build dir, do it there.

## Pitfalls

- Inspecting `<controller>/zephyr/zephyr.dts`: it does not exist. Resolve the image first, then inspect `<controller>/<image>/zephyr/zephyr.dts`.
- Trusting `domains.yaml` `build_dir` after the tree moved: stale absolute path. Use `<controller-dir>/<image-name>`.
- Keying sysbuild detection on `cmake.sysbuild`: version-dependent. Use `domains.yaml` presence or a named `cmake.images[]` entry.
- Editing the main app's `boards/<board>.overlay` and expecting MCUboot's devicetree to change: per-image overlays only apply to their own image. Use `sysbuild/mcuboot.overlay` or `-Dmcuboot_EXTRA_DTC_OVERLAY_FILE=...`.
- Diagnosing "device not ready" against the wrong image's `zephyr.dts`: confirm which domain the failing code runs in before applying references/debugging.md.

## Out of scope

- Sysbuild Kconfig (`SB_CONFIG_*`) beyond the namespace pointer above, and per-image Kconfig triage: a Kconfig-focused skill topic.
- MCUboot signing, key management, and slot semantics beyond devicetree partitions (partition layout itself is covered in references/flash-partitions.md).
- Adding extra images to a project (`ExternalZephyrProject_Add`) and sysbuild CMake module authoring: build-system topics, not devicetree.
