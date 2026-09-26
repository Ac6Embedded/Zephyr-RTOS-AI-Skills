# Symbols and dependencies

When to read this: you need the underlying model of what `depends on`, `select`,
`imply`, `default`, `range`, and `choice` actually do to a symbol, or a symbol
resists being set and you want to reason out why. For triaging a concrete build
warning, start at `references/debugging.md`; this file is the semantics behind it.

## The configuration tree

Kconfig is a tree of nodes, not a flat symbol list. Four node kinds exist:
menus, comments, symbols, and choices. Menus and comments carry no value; they
structure the tree and contribute conditions.

- A symbol can be defined in several places. Each definition site has its own
  prompt and help text; the menuconfig info screen aggregates prompts, helps,
  and definition locations across all sites.
- `menuconfig FOO` defines a normal symbol that is rendered as a menu whose
  children are its dependents. It is a display convention, not a new node kind.
- Tree order (the order menuconfig displays) is also the order symbols are
  written to `build/zephyr/.config`.

A real trimmed excerpt (`$ZEPHYR_BASE/subsys/logging/Kconfig`, Zephyr 4.4):

```kconfig
menu "Logging"

config LOG
	bool "Logging"
	select PRINTK if USERSPACE

if LOG

config LOG_CORE_INIT_PRIORITY
	int "Log Core Initialization Priority"
	range 0 99
	default 0

endif # LOG

endmenu
```

`if LOG` is not a node. It folds into each enclosed symbol: here
`LOG_CORE_INIT_PRIORITY` behaves exactly as if it carried `depends on LOG`.

## Types

Symbols are typed: `bool`, `int`, `hex`, or `string`. Kconfig in general also
has tristate, but Zephyr does not use it: every bool is `y` or `n`, and `m`
never appears. Practical type facts:

- `int` is base 10; `hex` is base 16 and written with `0x`.
- Strings are unquoted in memory; double quoting is a file convention with only
  `"` and `\` escaped.
- How each type is written in a `.conf` file (including the
  `# CONFIG_X is not set` form for bool n) is covered in
  `references/conf-syntax.md`.

## Three states per symbol

Every symbol has three distinct states. Keep them separate: almost every
"Kconfig will not do what I want" situation is exactly one of them failing.

1. Visibility: the symbol has a prompt AND the prompt condition holds. A
   prompt is (text, condition); a promptless symbol can never be shown or
   edited, it is only ever driven by defaults and selects. A `menu` can
   additionally be hidden by `visible if` (menus only).
2. Value: ALWAYS resolved, independent of visibility. An invisible symbol can
   still be `y`, through a `select` or a satisfied `default`.
3. Assignability: which values can be set right now.
   - Invisible: nothing is settable (this is why assigning a promptless symbol
     from a `.conf` file is a hard error; see `references/debugging.md`).
   - Visible choice member: only `y` can be assigned, never `n` directly.
   - Forced `y` by `select`: cannot be turned off.
   - Otherwise, a visible bool can be set to `n` or `y`.

Consequence: "why is CONFIG_X on when nothing sets it" is a value question
(default or select), while "why can't I change CONFIG_X" is a visibility or
assignability question. Answer them with different tools: grep the merged
`build/zephyr/.config` for values, the menuconfig info screen for the rest
(`references/finding-options.md`).

## Effective dependencies

The effective direct dependency of a symbol is the AND of:

- every `depends on` line at every definition site,
- every enclosing `if <expr>` block condition,
- every enclosing `menu` condition.

When a symbol will not turn on, the blockers are exactly the AND terms that
currently evaluate to `n`. The menuconfig info screen lists the dependency
split per AND term with each term's current value, and the configure-time
warning ("was assigned the value 'y' but got the value 'n'. Check these
unsatisfied dependencies: ...") lists the same false terms. That warning does
NOT abort the build; the feature silently stays off
(`references/debugging.md`).

Real example (`$ZEPHYR_BASE/subsys/shell/Kconfig`, Zephyr 4.4, trimmed):

```kconfig
menuconfig SHELL
	bool "Shell"
	imply LOG_RUNTIME_FILTERING

if SHELL

config SHELL_LOG_BACKEND
	bool "Shell log backend"
	depends on LOG && !LOG_MODE_MINIMAL
	select MPSC_PBUF
	select LOG_OUTPUT
	default y if LOG

endif # SHELL
```

Effective dependency of `SHELL_LOG_BACKEND`:
`SHELL && LOG && !LOG_MODE_MINIMAL`. The `depends on` line shows only two
terms; the third (`SHELL`) folds in from the enclosing `if SHELL` block. Grep
for enclosing blocks when reading Kconfig sources, or trust the info screen,
which always shows the full folded expression.

## depends on vs select vs imply

| Construct | Direction | Effect | User can override? |
| --- | --- | --- | --- |
| `depends on FOO` | this symbol needs FOO | gates visibility and assignability; symbol cannot be `y` while FOO is `n` | no (fix FOO instead) |
| `select FOO` | this symbol forces FOO | forces FOO to `y`, IGNORING FOO's own `depends on` | no (disable the selector) |
| `imply FOO` | this symbol suggests FOO | acts as a weak default of `y` for FOO | yes (`# CONFIG_FOO is not set`) |

Details that matter in practice:

- `select` ignoring the target's own dependencies is the reason Zephyr style
  guidance warns against it: a selector can force a symbol into a state its
  dependencies forbid. Reserve `select` for promptless helper symbols with no
  dependencies of their own (`references/kconfig-authoring.md`).
- A select-forced symbol cannot be turned off from any `.conf` file. Writing
  `# CONFIG_FOO is not set` produces a configure warning that FOO "was
  assigned the value 'n' but got the value 'y'" and the value stays `y`. Find
  the selector on the info screen ("selected by") and disable that instead.
- Selects can be conditional: `select PRINTK if USERSPACE` (real, on
  `CONFIG_LOG` above) only forces `PRINTK` when `USERSPACE` is `y`.
- Gap-fill (verified against the configure scripts of a Zephyr 4.4 checkout;
  re-verify on yours): when a `select` fires while the target's own
  dependencies are unmet, the configure stage emits a warning that the target
  "is currently being y-selected by" the selector (wording varies across
  versions), and Zephyr promotes this class of warning to a hard error
  ("Aborting due to Kconfig warnings"). So a dependency-violating select does
  not silently misconfigure; it stops the build.
- `imply` is soft in both directions: the user can still set the target to
  `n`, and, Gap-fill (verified against the configure scripts of a Zephyr 4.4
  checkout; re-verify on yours): an `imply` only takes effect when the
  target's own dependencies are met, unlike `select`. Example above:
  `SHELL` implies `LOG_RUNTIME_FILTERING`; enabling the shell turns runtime
  log filtering on by default, but `# CONFIG_LOG_RUNTIME_FILTERING is not set`
  in prj.conf still wins.

## Defaults

`default` lines are ordered (value, condition) pairs, accumulated across
definition sites. The FIRST default whose condition holds wins. A user
assignment (any `.conf` line, or a menuconfig edit) beats all defaults.
Unsetting (deleting the `.conf` line so no user value exists) restores the
default; that is different from assigning `n`.

Real example of ordered defaults (`$ZEPHYR_BASE/subsys/logging/Kconfig.mode`,
Zephyr 4.4):

```kconfig
choice LOG_MODE
	prompt "Mode"
	depends on !LOG_FRONTEND_ONLY
	default LOG_MODE_IMMEDIATE if ARCH_POSIX
	default LOG_MODE_MINIMAL if LOG_DEFAULT_MINIMAL
	default LOG_MODE_DEFERRED
```

On a POSIX (native simulator) build the first line fires and the mode is
immediate; on hardware with `LOG_DEFAULT_MINIMAL=y` the second fires; plain
hardware falls through to deferred. Only the first satisfied line counts:
order is priority.

DT-derived defaults: some defaults are computed from the resolved devicetree,
through `dt_*` preprocessor functions and the generated `DT_HAS_*` symbols
that gate drivers (pattern: `default y` plus `depends on
DT_HAS_<COMPAT>_ENABLED`). These values move with overlays and board files,
never with `.conf` edits. If a wanted driver symbol reports
`DT_HAS_..._ENABLED (=n)` as a blocker, fix the devicetree node (status
`okay`), not the Kconfig side: see the zephyr-devicetree skill, and
`references/debugging.md` for the triage flow.

## Ranges (int and hex only)

`range low high [if condition]` lines are conditional, like defaults: the
ACTIVE range is the first one whose condition currently holds, and it changes
live as other symbols change. Bounds may themselves be hex.

```kconfig
config APP_BUF_SIZE
	int "Working buffer size"
	range 64 256 if APP_SMALL_FOOTPRINT
	range 64 4096
	default 1024
```

Real single-range example: `CONFIG_LOG_BUFFER_SIZE` has `range 128 1048576`
(`$ZEPHYR_BASE/subsys/logging/Kconfig.processing`).

Enforcement differs by entry path:

- menuconfig rejects malformed or out-of-range input in the UI.
- A `.conf` value outside the active range is CLAMPED SILENTLY at merge time.
  Never rely on clamping: write in-range values, then verify the result in
  `build/zephyr/.config`.

```conf
# prj.conf
CONFIG_LOG_BUFFER_SIZE=64
# merges as 128 (clamped to the range low bound), with no warning
```

## Choices

A `choice` is a radio group of bool members. Rules:

- Exactly one member is `y` whenever the choice is active. A non-optional
  choice can never be all `n`.
- Selecting a member means setting it to `y`; the previous winner deselects
  automatically. You can never set a member directly to `n`.
- Across fragments, the LAST member set to `y` wins (consistent with the
  general last-assignment-wins rule, `references/merge-order.md`).
- The default selection of a non-optional choice is implicit: minimal configs
  do not write it, and neither should your fragments
  (`references/writing-fragments.md`).

To switch the logging mode from the example above:

```conf
# prj.conf: right way to pick a different radio member
CONFIG_LOG_MODE_IMMEDIATE=y
```

Do not write `# CONFIG_LOG_MODE_DEFERRED is not set` to push the old winner
out: members are not directly assignable to `n`, and a mis-selection (your
member did not win) only produces a configure warning
(`references/debugging.md`).

## Worked example: select across a dependency chain

Illustrative tree (three app symbols, `references/kconfig-authoring.md` for
where such a file lives):

```kconfig
config APP_CRYPTO
	bool "Crypto layer"
	select APP_ENTROPY

config APP_ENTROPY
	bool "Entropy pool"
	depends on APP_RNG

config APP_RNG
	bool "RNG driver"
```

Scenario 1, dependencies satisfied. prj.conf:

```conf
CONFIG_APP_RNG=y
CONFIG_APP_CRYPTO=y
```

Resulting `build/zephyr/.config` fragment:

```conf
CONFIG_APP_CRYPTO=y
CONFIG_APP_ENTROPY=y
CONFIG_APP_RNG=y
```

`APP_ENTROPY` is `y` although no file ever assigned it: the select forced it.
Adding `# CONFIG_APP_ENTROPY is not set` to a later fragment changes nothing:
the configure output warns it "was assigned the value 'n' but got the value
'y'", the build continues, and the value stays `y`. To turn it off, disable
the selector (`# CONFIG_APP_CRYPTO is not set`).

Scenario 2, the select violates the target's dependency. prj.conf sets only
`CONFIG_APP_CRYPTO=y`, leaving `APP_RNG` at `n`. The select tries to force
`APP_ENTROPY=y` while its `depends on APP_RNG` is false. Gap-fill (verified
against the configure scripts of a Zephyr 4.4 checkout; re-verify on yours):
the configure stage warns that `APP_ENTROPY` "is currently being y-selected
by" `APP_CRYPTO` (wording varies across versions) and aborts with "Aborting
due to Kconfig warnings". Fix by satisfying the dependency
(`CONFIG_APP_RNG=y`), making the select conditional
(`select APP_ENTROPY if APP_RNG`), or using `depends on` instead.

Scenario 3, same false term without a select. prj.conf sets only
`CONFIG_APP_ENTROPY=y`. That is a plain unmet-dependency assignment: the
configure output warns it "was assigned the value 'y' but got the value 'n'.
Check these unsatisfied dependencies: APP_RNG (=n)", the build SUCCEEDS, and
the feature stays off. Same blocker, different severity: always read the
configure output (`references/debugging.md`).

## Worked example: dissecting a real symbol

Take `CONFIG_SHELL_LOG_BACKEND` (source excerpt in the dependencies section
above) and read it the way the menuconfig info screen presents it
(`references/menuconfig-guiconfig.md` for reaching that screen):

- Prompt: "Shell log backend", one definition site,
  `$ZEPHYR_BASE/subsys/shell/Kconfig`.
- Direct dependencies, split per AND term with current values:
  `SHELL (=y) && LOG (=y) && !LOG_MODE_MINIMAL (=y)`. The `SHELL` term comes
  from the enclosing `if SHELL`, not from the `depends on` line.
- Defaults: `y if LOG`. With the shell and logging both enabled, this symbol
  is `y` in `.config` with no prj.conf line anywhere: the default fired.
- Selects (outgoing): forces `MPSC_PBUF` and `LOG_OUTPUT` to `y` whenever it
  is `y`; those two will appear enabled without any assignment.
- Selected by (incoming): empty here, so this symbol is NOT select-forced.
  That makes it assignable: `# CONFIG_SHELL_LOG_BACKEND is not set` in
  prj.conf is the correct, durable way to silence log output to the shell.
- If `CONFIG_LOG` were `n`, this symbol would be invisible and `n`; assigning
  it `=y` from a fragment would then produce the unmet-dependency warning with
  `LOG (=n)` as the blocker, and the build would continue without it.

The general recipe: effective dependency terms tell you what blocks it,
"selected by" tells you if you can turn it off, the default list tells you why
it is on without an assignment, and `build/zephyr/.config` confirms the final
value after a configure.
