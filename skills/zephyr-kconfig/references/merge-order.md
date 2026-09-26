# Kconfig Fragment Merge Order

When to read this: you need to know which files feed `build/zephyr/.config`, in what
order, how to register a new fragment or a build profile, or why a symbol has a value
you did not write. For the syntax inside a fragment, read `references/conf-syntax.md`.

## The merge model

The configure stage builds `.config` from an ordered list of fragment files. The first
file is the base; each subsequent file merges on top of it. Two rules follow:

- Later files win. If two fragments assign the same symbol, the assignment from the
  file merged later is the effective one. This holds for `# CONFIG_X is not set` too,
  because that line is an assignment of `n`, not a comment.
- Fragments are sparse deltas. A symbol mentioned in no fragment keeps its Kconfig
  default (see `references/symbols-and-dependencies.md`; some defaults are computed
  from the devicetree, which the zephyr-devicetree skill covers).

```conf
# merged earlier (board defconfig)
CONFIG_GPIO=y

# merged later (prj.conf): this assignment wins
# CONFIG_GPIO is not set
```

Duplicate assignments across fragments are silent by design: overriding an earlier
fragment is the normal mechanism, not an error. If a value surprises you, suspect a
later fragment first (`references/debugging.md`).

## The full order

Verified identical in Zephyr 4.4.1 and the 4.5 development tree
(`$ZEPHYR_BASE/cmake/modules/kconfig.cmake`). Later entries win:

1. Board defconfig: `$ZEPHYR_BASE/boards/<vendor>/<board>/<board>_defconfig`, plus
   qualifier and revision defconfig variants when the board target has them.
2. Legacy board-revision `.conf` files (deprecated in the 4.5 development tree;
   treat as legacy only).
3. Board extension conf files (out-of-tree board extensions).
4. The CONF_FILE slot (prj.conf and friends; details below).
5. Shield conf files (each shield selected with `-DSHIELD=<name>` brings its own).
6. The EXTRA_CONF_FILE slot (details below).
7. CLI `-DCONFIG_<X>=<value>` assignments, collected into a generated options file
   merged here (details below).
8. A glob of `*.conf` files sitting at the top of the build directory (not in
   `build/zephyr/`), sorted by name and merged last.

Practical reading of the order: the board sets hardware-driven baselines, the
application (CONF_FILE slot) overrides the board, shields and extra fragments
override the application, and command-line assignments override everything except
stray `.conf` files left in the build directory.

## Inside the CONF_FILE slot

When you do not set `CONF_FILE` yourself, the slot expands automatically, in this
internal order (later wins), all picked up from the application configuration
directory:

1. `prj.conf`
2. `socs/<SOC>_<QUALIFIERS>.conf` (when it exists)
3. `boards/<BOARD>.conf`, optionally with `_<qualifiers>` and `_<revision>` parts
   in the name (when it exists)

This is why a `boards/<BOARD>.conf` fragment overrides `prj.conf` for that board
only, with no registration step: the name alone activates it. Choosing between these
homes is covered in `references/writing-fragments.md`.

Setting `CONF_FILE` manually REPLACES `prj.conf` and disables the automatic
`socs/` and `boards/` pickup entirely:

```shell
west build -b <board> <app-dir> -- -DCONF_FILE=prj_release.conf
```

`CONF_FILE` accepts a semicolon-separated list; the listed files merge in list order,
later entries winning.

`FILE_SUFFIX=<s>` is the profile mechanism that keeps the automatic pickup: it swaps
`prj.conf` for `prj_<s>.conf` when that file exists (falling back to `prj.conf`
otherwise):

```shell
west build -b <board> <app-dir> -- -DFILE_SUFFIX=release
# uses prj_release.conf instead of prj.conf, keeps socs/ and boards/ pickup
```

## CONF_FILE vs EXTRA_CONF_FILE (the classic confusion)

- `CONF_FILE` REPLACES `prj.conf` (and kills the `boards/`/`socs/` auto-pickup).
- `EXTRA_CONF_FILE` ADDS fragments on top of the normal set; `prj.conf` still applies.

If you meant "prj.conf plus my debug settings" and used `-DCONF_FILE=debug.conf`,
everything in `prj.conf` silently reverted to defaults. Almost always you want:

```shell
west build -b <board> <app-dir> -- -DEXTRA_CONF_FILE=debug.conf
```

## Inside the EXTRA_CONF_FILE slot

`EXTRA_CONF_FILE` entries come from several sources. Their internal order (later
wins):

1. Entries set in the application's CMakeLists.txt with `set(EXTRA_CONF_FILE ...)`
   (Gap-fill: place the `set()` before `find_package(Zephyr)`; verify against your
   checkout's application development docs).
2. Entries from the `EXTRA_CONF_FILE` environment variable.
3. `.conf` files contributed by active snippets (Gap-fill: snippets are activated
   with `west build -S <snippet>` or `-DSNIPPET=<snippet>`; verify the flag against
   your checkout). Snippet fragments therefore merge BEFORE fragments you pass on
   the command line, so your `-DEXTRA_CONF_FILE` entries can override a snippet.
4. Entries passed on the command line with `-DEXTRA_CONF_FILE=...`.

Multiple files are accepted, semicolon or space separated, merged in list order:

```shell
west build -b <board> <app-dir> -- -DEXTRA_CONF_FILE="debug.conf;trace.conf"
```

`OVERLAY_CONFIG` is the deprecated alias for `EXTRA_CONF_FILE`; treat them as the
same slot when you meet it in older projects.

The value is cached: once passed, it sticks across incremental rebuilds of the same
build directory. When you add, remove, or rename config sources, force a clean
regeneration:

```shell
west build -p -b <board> <app-dir> -- -DEXTRA_CONF_FILE=debug.conf
```

Creating a fragment file changes nothing until it is registered here (or matches an
auto-picked name from the CONF_FILE slot). See `references/writing-fragments.md` for
what to put inside.

## CLI -DCONFIG_ assignments

Single symbols can be assigned directly on the west or CMake command line:

```shell
west build -b <board> <app-dir> -- -DCONFIG_ASSERT=y -DCONFIG_MAIN_STACK_SIZE=4096
```

These are collected into a generated fragment merged AFTER the EXTRA_CONF_FILE slot,
so they beat every file-based fragment. This mechanism is officially experimental but
stable in practice. The values persist in the CMake cache across rebuilds of that
build directory: an assignment you passed once keeps applying until a pristine build,
which is a common source of "where is this value coming from" surprises.

## Path resolution and typos

Relative fragment paths (in `CONF_FILE`, `EXTRA_CONF_FILE`, and the auto-picked
names) resolve against the application configuration directory, which is not
necessarily the application source directory (Gap-fill: it defaults to the
application source directory unless `APPLICATION_CONFIG_DIR` redirects it; verify
against your checkout). Absolute paths are used as-is.

A listed file that does not exist is a hard CMake error at configure time, with a
message containing `File not found:` and the resolved path. Typos in fragment names
fail loudly; typos in symbol names inside a fragment are a different failure class
(`references/debugging.md`).

## Seeing what actually merged

On Zephyr 4.0 or newer, read `<build>/build_info.yml`:

```yaml
cmake:
  kconfig:
    files:              # the full ordered merge list, board defconfig first
      - /path/to/boards/<vendor>/<board>/<board>_defconfig
      - /path/to/app/prj.conf
      - /path/to/app/debug.conf
    user-files:         # what filled the CONF_FILE slot
      - /path/to/app/prj.conf
    extra-user-files:   # what filled the EXTRA_CONF_FILE slot
      - /path/to/app/debug.conf
```

```shell
grep -A 20 "kconfig:" <build>/build_info.yml
```

Caveat: `build_info.yml` is only rewritten when `.config` is actually regenerated,
so after a no-op configure it reflects the last real regeneration (see
`references/build-artifacts.md` for staleness rules).

Fallback on older trees: the configure log prints one line per merged fragment;
look for the lines containing `Merged configuration` (wording varies across
versions).

## Sysbuild builds

Under sysbuild each image runs its own merge with its own fragment list; the
controller build directory has no `zephyr/.config` at its top level. If the build
directory contains `domains.yaml`, read `references/sysbuild.md` before applying
anything above.

## Worked example: one symbol through three files

Trace `CONFIG_GPIO` through a build where three files mention it (values
hypothetical):

```conf
# 1. boards/<vendor>/<board>/<board>_defconfig  (merged first)
CONFIG_GPIO=y

# 2. prj.conf  (CONF_FILE slot, merges over the defconfig)
# CONFIG_GPIO is not set

# 3. lowpower.conf  (passed via -DEXTRA_CONF_FILE, merges last)
CONFIG_GPIO=y
```

```shell
west build -b <board> <app-dir> -- -DEXTRA_CONF_FILE=lowpower.conf
grep -E "CONFIG_GPIO(=| is not set)" <build>/zephyr/.config
# CONFIG_GPIO=y        <- lowpower.conf won, because it merged last
```

To find WHICH file produced the effective value, grep the merge list in reverse
order and stop at the first hit:

```shell
# paste the cmake.kconfig.files list from build_info.yml, deepest-priority last;
# walk it backwards:
for f in <file3> <file2> <file1>; do
  grep -Hn -E "CONFIG_GPIO(=| is not set)" "$f" && break
done
```

The first file that matches (searching backwards) holds the winning assignment.
If none matches, no fragment set the symbol: its value comes from a Kconfig
default or a `select`, which the symbol info screen explains
(`references/menuconfig-guiconfig.md`, `references/debugging.md`).
