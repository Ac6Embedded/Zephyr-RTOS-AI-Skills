# Interactive Configuration: menuconfig, guiconfig, hardenconfig

When to read this: you want to browse or experiment with configuration options
interactively, understand what the menuconfig UI is telling you, keep interactive
edits from disappearing, audit a configuration for security hardening, or find
where the ROM/RAM went.

## Running the interfaces

Both interfaces operate on an existing build directory. They run the CMake
configure stage first if it has not run yet, so the merged configuration they
show is always current:

```shell
west build -t menuconfig --build-dir <build-dir>    # terminal TUI
west build -t guiconfig --build-dir <build-dir>     # graphical, own window
```

To run only the configure stage (produce a fresh `<build>/zephyr/.config`
without compiling anything):

```shell
west build --cmake-only --build-dir <build-dir>
```

Both interfaces present the same Kconfig tree and the same rules; only the
front end differs. If `guiconfig` fails with an import error for `tkinter`,
install your distribution's Tk package (usually named `python3-tk` or
`python3-tkinter`); `tkinter` is standard library but often not installed by
default.

If the build directory contains `domains.yaml` it is a sysbuild controller and
has no `zephyr/.config` of its own; configure the right image instead, see
references/sysbuild.md.

## The critical rule: interactive edits are temporary

menuconfig and guiconfig edit the already-merged `<build>/zephyr/.config`
directly. They never touch prj.conf or any fragment. Consequences:

- Edits survive incremental builds: run `west build` again and your change is
  compiled in.
- Edits are lost on a pristine build (`west build -p`) and whenever any
  fragment or Kconfig source file changes, because the build re-merges
  `.config` from its inputs at that point.

Treat every interactive session as an experiment. When an edit proves right,
persist it in prj.conf or a registered fragment before it evaporates: see
references/writing-fragments.md for deriving the minimal fragment from a
session.

Hand-editing `<build>/zephyr/.config` in a text editor is legal but
dependency-unaware: an assignment with unsatisfied dependencies is silently
dropped the next time the configuration is reprocessed. The interfaces are
safer because they enforce dependencies live.

## Navigating menuconfig

- Arrow keys move; common Vim bindings also work.
- `Space` and `Enter` toggle values and enter menus (menus show `--->` next to
  them); `ESC` returns to the parent menu.
- `Y` and `N` set a boolean symbol directly.
- `?` opens the symbol information screen; `ESC` or `Q` returns from it.
- `/` opens the jump-to search dialog (also works in guiconfig).
- `D` saves a minimal configuration file (see Saving below).
- `Q` quits, offering a save dialog when there are unsaved changes; save to the
  default filename (`zephyr/.config`) so the next build uses your edits.

Value display:

- `[ ]` brackets: boolean symbols. `( )` brackets: int, hex, and string
  symbols.
- Entries shown as `- -` or `-*-` cannot be changed here: the symbol is either
  invisible in the current context or forced `y` by a `select`. Nothing you do
  in the UI will move it; change what drives it instead (see
  references/symbols-and-dependencies.md).
- Choices are radio groups: selecting one member sets it `y` and deselects the
  others. There is no way to set a choice member directly to `n`; pick the
  member you want `y` instead.
- Numeric input is validated: malformed or out-of-range int/hex input is
  rejected in the UI, and hex input gets `0x` prepended automatically. Contrast
  this with `.conf` files, where an out-of-range value is clamped silently at
  merge time (references/conf-syntax.md).
- A `(NEW)` marker means the symbol has never been assigned anywhere (no
  fragment, no previous session); its current value comes purely from its
  defaults.

In guiconfig, click the image next to a symbol or double-click its row to
change the value (double-click on a symbol with children opens the menu
instead). The bottom pane always shows the selected symbol's information, the
equivalent of `?` in menuconfig.

## Searching for symbols

Press `/` in either interface to open the jump-to dialog. Search semantics:

- The query is split on whitespace into tokens; each token is a regular
  expression, and ALL tokens must match (AND).
- Only the symbol NAME and its PROMPT text are searched, never the help text.
- Results are ordered: symbols first, then choices, menus, and comments,
  alphabetically within each group.

Jumping to a symbol that is not currently visible enables show-all mode, which
displays invisible symbols too. Turn it off with `A` in menuconfig or `Ctrl-A`
in guiconfig. In menuconfig, `Ctrl-F` inside the jump-to dialog shows the
selected result's help without leaving the dialog. For search strategy and a
grep-based alternative, see references/finding-options.md.

## The symbol information screen

Press `?` on any symbol (menuconfig) or read the bottom pane (guiconfig). This
is the primary provenance tool: it answers "why does this symbol have this
value" better than any grep. It shows:

- Prompt and help text per definition site (a symbol can be defined in several
  Kconfig files; each site contributes its own).
- Direct dependencies, split per AND term, with each term's current value. The
  terms currently evaluating to `n` are exactly the blockers you must fix to
  make the symbol assignable.
- Defaults as ordered (value, condition) pairs; the first satisfied one wins.
- Symbols this one selects or implies, and symbols that select or imply it
  ("selected by" is how you find who is forcing a `y` you cannot turn off).
- Definition locations with the Kconfig include chain and the menu path.

For turning an info-screen reading into a fix, follow the decision tree in
references/debugging.md.

## Saving

- `Q` then `Y` writes the full configuration to `zephyr/.config` (the file the
  build consumes).
- `D` writes a minimal configuration file: only symbols that differ from their
  default value in the current context. This is the best starting point for a
  permanent fragment because it already applies most of the fragment-authoring
  rules from references/writing-fragments.md.

## Persisting a session (the workflow)

1. Experiment in menuconfig until the behavior is right; save and build.
2. Press `D` and save the minimal configuration to a scratch file, for example
   `<build>/minimal.conf`.
3. Copy the relevant lines into `prj.conf` (always-on) or a named fragment
   (profile-specific), pruning per references/writing-fragments.md.
4. Force a clean re-merge and confirm the setting survives without your
   interactive edit:

```shell
west build -p -b <board> <app-dir>
grep "CONFIG_<SYM>" <build-dir>/zephyr/.config
```

If the grep result changed after the pristine build, the line did not make it
into a merged fragment; check registration per references/merge-order.md.

## hardenconfig: security audit

```shell
west build -t hardenconfig --build-dir <build-dir>
```

hardenconfig is a read-only audit; it never modifies `.config`. It compares
the current `<build>/zephyr/.config` against the Zephyr Security Working
Group's recommendations (`$ZEPHYR_BASE/scripts/kconfig/hardened.csv`) and
additionally flags enabled symbols that select `EXPERIMENTAL`, `DEPRECATED`,
or `NOT_SECURE`. It prints a table with Name, Current, Recommended, and
Check-result columns (requires the Python `tabulate` package).

Remediation is manual: put the recommended values in `prj.conf` (or a
dedicated fragment such as `harden.conf` registered via `EXTRA_CONF_FILE`),
then rebuild and re-run the target until the table is clean:

```conf
# Example remediations (both are real hardened.csv entries recommended y);
# always drive the list from what hardenconfig reports for YOUR target
CONFIG_STACK_SENTINEL=y
CONFIG_HW_STACK_PROTECTION=y
```

The set of rows the table shows depends on your current configuration and
architecture; read the recommendations for your exact tree in
`$ZEPHYR_BASE/scripts/kconfig/hardened.csv`.

## Footprint reports and size levers

After any configuration change, rebuild before measuring:

```shell
west build -t rom_report --build-dir <build-dir>    # flash footprint by symbol
west build -t ram_report --build-dir <build-dir>    # RAM footprint by symbol
```

Optimization-level symbols (a choice; set exactly one):

- `CONFIG_SIZE_OPTIMIZATIONS` (-Os, the usual default)
- `CONFIG_SIZE_OPTIMIZATIONS_AGGRESSIVE` (-Oz, Zephyr 3.4 and newer)
- `CONFIG_SPEED_OPTIMIZATIONS`, `CONFIG_DEBUG_OPTIMIZATIONS`,
  `CONFIG_NO_OPTIMIZATIONS`

Candidate reduction levers, to be treated as starting points you MEASURE with
`rom_report` (not guaranteed wins): `CONFIG_ASSERT=n`, `CONFIG_LOG=n` or
`CONFIG_LOG_MODE_MINIMAL=y`, `CONFIG_CBPRINTF_NANO=y`,
`CONFIG_BOOT_BANNER=n`, `CONFIG_THREAD_NAME=n`, smaller
`CONFIG_MAIN_STACK_SIZE` and `CONFIG_HEAP_MEM_POOL_SIZE`,
`CONFIG_MINIMAL_LIBC=y`.

## Worked example: from session to permanent fragment

Goal: enable logging at debug level, first interactively, then permanently.

```shell
west build -b <board> <app-dir>
west build -t menuconfig --build-dir build
```

In menuconfig: press `/`, type `LOG_DEFAULT`, jump to
`LOG_DEFAULT_LEVEL`. It is not settable yet (`- -`), so press `?`: the
dependency line shows `LOG (=n)` as the blocker. Jump to `LOG`, press `Y`,
return, set `LOG_DEFAULT_LEVEL` to 4. Press `Q`, save to `zephyr/.config`,
rebuild, and confirm the behavior on target.

Now persist it. Press `D` in a new menuconfig session and save the minimal
configuration; among its lines you will find:

```conf
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=4
```

Copy exactly those two lines into `prj.conf` (they represent your intent; the
cascade of logging sub-symbols follows from defaults and stays out of the
file). Then prove the setting no longer depends on the interactive edit:

```shell
west build -p -b <board> <app-dir>
grep -E "CONFIG_LOG(=|_DEFAULT_LEVEL)" build/zephyr/.config
```

Expected: `CONFIG_LOG=y` and `CONFIG_LOG_DEFAULT_LEVEL=4` are present after a
pristine build. Without the prj.conf lines, the pristine build would have
silently reverted both to their defaults.
