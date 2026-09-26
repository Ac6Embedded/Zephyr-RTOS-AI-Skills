# .conf File Syntax

When to read this: you are writing or reading any line of `prj.conf`, a `.conf`
fragment, or a generated `build/zephyr/.config` (value formats, comments, the
`is not set` form, duplicates, ranges). For which files merge and in what order,
read `references/merge-order.md`; for what belongs in which file, read
`references/writing-fragments.md`.

## Line grammar

A `.conf` file has exactly three kinds of lines:

```conf
CONFIG_MAIN_STACK_SIZE=2048
# CONFIG_ASSERT is not set
# a free-form comment (blank lines are fine too)
```

1. An assignment: `CONFIG_<NAME>=<value>`, one per line.
2. An assignment of `n` in comment form: `# CONFIG_<NAME> is not set`.
3. A comment (`#` first) or a blank line.

Rules:

- The `CONFIG_` prefix is mandatory. `LOG=y` is not an assignment.
- Everything between `=` and the end of line is the value. Never append a
  trailing comment to an assignment line; put comments on their own line.
- The `is not set` form must match its shape exactly (starts with `# CONFIG_`,
  single spaces, ends with ` is not set`). Anything else is an ordinary
  comment and silently does nothing.
- Gap-fill (verified against Zephyr 4.4.1 and 4.4.99; double-check on other
  versions): parsing is strict. Do not indent assignments and do not put
  spaces around `=`. A line that is none of the three kinds, or a value that
  does not fit the symbol's type, fails the configure step ("ignoring
  malformed line", "is invalid for"; wording varies across versions).

## Value forms by type

Zephyr has no tristate: every bool is `y` or `n`, `m` never appears. Write the
form that matches the symbol's type (find the type via
`references/finding-options.md`):

| Type | Form | Example |
| --- | --- | --- |
| bool, on | `=y` | `CONFIG_LOG=y` |
| bool, off | comment form | `# CONFIG_ASSERT is not set` |
| int | decimal, unquoted | `CONFIG_MAIN_STACK_SIZE=2048` |
| hex | `0x` prefix, unquoted | `CONFIG_FLASH_LOAD_OFFSET=0x20000` |
| string | double-quoted | `CONFIG_BT_DEVICE_NAME="Zephyr demo"` |

Notes:

- `CONFIG_X=n` parses as `n`, but generated files never emit it. Always write
  the comment form so your fragments match `.config` and grep one shape.
- Strings support exactly two escapes: `\"` and `\\`. There is no `\n` and no
  Unicode escape. An unquoted string value fails the configure.
- Gap-fill (verified against Zephyr 4.4.1 and 4.4.99; double-check on other
  versions): a hex symbol's value is read base 16 even without the prefix, so
  `CONFIG_X=10` on a hex symbol means 16 decimal. Always write `0x`.

## Off is not the same as unset

For a bool there are three distinct situations, not two:

| Line in your fragment | Meaning |
| --- | --- |
| `CONFIG_X=y` | user assignment of `y` |
| `# CONFIG_X is not set` | user assignment of `n` (a real assignment, not a comment) |
| no line at all | no user value: the symbol's default decides |

To let a symbol fall back to its default, delete its line entirely. Forcing
`n` and falling back are different whenever the default is `y` or is computed
(for example from the devicetree). How defaults resolve is covered in
`references/symbols-and-dependencies.md`.

## Duplicates: last assignment wins

Assignments are order-sensitive. The last one wins, within a single file and
across fragments in merge order (`references/merge-order.md`). The comment
form participates like any other assignment:

```conf
CONFIG_ASSERT=y
CONFIG_LOG_DEFAULT_LEVEL=4
# CONFIG_ASSERT is not set
CONFIG_LOG_DEFAULT_LEVEL=3
```

Result: `CONFIG_ASSERT` is `n` and `CONFIG_LOG_DEFAULT_LEVEL` is `3`.
Duplicate and overriding assignments are silent by design (that is how
fragments override the board defconfig), so nothing warns you about the lines
that lost. Keep each symbol on exactly one line per file.

## Ranges (int and hex)

Int and hex symbols may carry `range` constraints in their Kconfig definition:

```kconfig
config LOG_DEFAULT_LEVEL
	int "Default log level"
	default 3
	range 0 4
```

- Ranges are conditional: the ACTIVE range is the first
  `range <low> <high> if <cond>` line whose condition currently holds, so it
  can change as other symbols change. Bounds may themselves be hex.
- An out-of-range value in a `.conf` never takes effect as written. Never rely
  on the build clamping it for you: look up the active range first (the
  symbol info screen shows it, see `references/finding-options.md`) and write
  an in-range value.
- Gap-fill (verified against Zephyr 4.4.1 and 4.4.99; double-check on other
  versions): the configure step reports an out-of-range assignment as
  "ignored due to being outside the active range" and treats it as a Kconfig
  warning, which aborts the build rather than clamping your value.

## What you cannot write

Message texts below were verified on Zephyr 4.4.x; wording varies across
versions. Severity differs by case, so always read the configure output:

- An unknown or misspelled name is a HARD ERROR: the output mentions an
  "undefined symbol" and ends with "Aborting due to Kconfig warnings".
  Causes: a typo, a symbol that only exists in a newer Zephyr, or a module
  missing from the manifest.
- A promptless symbol is a HARD ERROR: the output says the symbol is
  "not directly user-configurable (has no prompt)". Set the prompted symbol
  (or devicetree node) that drives it instead; see
  `references/symbols-and-dependencies.md`.
- A choice member can never be set to `n`. Do not write
  `# CONFIG_<member> is not set`: set the member you want to `y` and the
  others deselect automatically (`references/symbols-and-dependencies.md`).
- A symbol forced `y` by `select` cannot be turned off from any `.conf` file.
  The line parses but has no effect; find and disable the selector
  (`references/debugging.md`).
- Contrast: assigning a symbol whose dependencies are unmet only WARNS
  ("was assigned the value 'y' but got the value 'n'") and the build
  continues without the feature. Triage in `references/debugging.md`.

## Reading a generated .config

`build/zephyr/.config` is the merged output, written in a fixed layout:

- Symbols appear in Kconfig tree order, each exactly once.
- Entering a visible menu emits a three-line banner (`#`, `# <menu prompt>`,
  `#`); leaving it emits `# end of <menu prompt>`.
- Bool `n` is written as `# CONFIG_X is not set`, never `=n`.

```conf
#
# General Kernel Options
#
CONFIG_MAIN_STACK_SIZE=1024
CONFIG_ISR_STACK_SIZE=2048
# CONFIG_DYNAMIC_THREAD is not set
# end of General Kernel Options
```

Grep for a symbol matching both the set and not-set shapes in one pass:

```shell
grep -E "^(# )?CONFIG_ASSERT[= ]" <build>/zephyr/.config
```

Never persist a change by editing this file: it is regenerated (and your edit
discarded) on any pristine build or fragment/Kconfig change, and hand edits
are dependency-unaware. See `references/build-artifacts.md` for lifecycle and
`references/menuconfig-guiconfig.md` for safe interactive editing.

## Worked example: an annotated prj.conf

Every symbol below exists in Zephyr 4.4.x. Bluetooth lines assume a board
with BT support; `CONFIG_FLASH_LOAD_OFFSET` is only assignable when the code
partition does not come from the devicetree.

```conf
# Logging: bool on, plus an int with range 0 4
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=4

# int: decimal, unquoted
CONFIG_MAIN_STACK_SIZE=2048

# bool off: exact comment shape, never =n
# CONFIG_ASSERT is not set

# string: double quotes mandatory; \" and \\ are the only escapes
CONFIG_BT=y
CONFIG_BT_DEVICE_NAME="Zephyr \"demo\" node"

# hex: keep the 0x prefix
CONFIG_FLASH_LOAD_OFFSET=0x20000

# duplicate: the LAST assignment wins, so the effective level is 3
CONFIG_LOG_DEFAULT_LEVEL=3
```

Build, then confirm what actually took effect:

```shell
west build -b <board> <app-dir>
grep -E "^(# )?CONFIG_(LOG_DEFAULT_LEVEL|ASSERT)[= ]" <build>/zephyr/.config
```

Expected output:

```conf
# CONFIG_ASSERT is not set
CONFIG_LOG_DEFAULT_LEVEL=3
```

Delete the duplicate line once you have seen the effect: one line per symbol
per file. Whether a line belongs in `prj.conf` at all (versus a fragment, or
not being written because it restates a default) is decided in
`references/writing-fragments.md`.
