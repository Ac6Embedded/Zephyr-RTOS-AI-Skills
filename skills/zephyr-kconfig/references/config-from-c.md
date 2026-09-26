# Using CONFIG_ Symbols from C

When to read this: you are writing or fixing C code that reacts to Kconfig
settings (`#ifdef`, `IS_ENABLED`, CONFIG_ values in expressions), or a symbol
you set "does not show up" in the code. For choosing and setting the symbol
itself, see `references/finding-options.md` and `references/writing-fragments.md`.

## How symbols reach C

No Kconfig processing happens at runtime. After the configure stage merges
everything into `<build>/zephyr/.config`, the build generates one preprocessor
macro per configured symbol into:

```
<build>/zephyr/include/generated/zephyr/autoconf.h
```

(Zephyr 3.7+; older trees generate it without the inner `zephyr/`, at
`<build>/zephyr/include/generated/autoconf.h`.)

This header is force-included into every C, C++, and assembly compile through
the compiler's `-imacros` mechanism. Never `#include` it manually: it is
already visible in every translation unit. To see exactly what was generated:

```shell
grep "CONFIG_<NAME>" <build>/zephyr/include/generated/zephyr/autoconf.h
```

The header reflects the merged `.config`, not your prj.conf alone; when a value
surprises you, debug the merge first (`references/debugging.md`), then the C side.

## Macro forms per type

| Kconfig type and value | In autoconf.h |
| --- | --- |
| bool `y` | `#define CONFIG_X 1` |
| bool `n` | absent: no `#define` at all (not defined to 0) |
| int | `#define CONFIG_X 4096` (decimal literal) |
| hex | `#define CONFIG_X 0x8000` (literal, keeps `0x`) |
| string | `#define CONFIG_X "value"` (quoted C string literal) |

The bool `n` row drives everything else on this page: a disabled bool is not
`0`, it is undefined. More generally, any symbol absent from `.config` (for
example because its dependencies are unmet) has no macro at all; symbols that
appear in `.config` are defined with the literal shown above.

## Testing bool symbols: two correct tools

### #ifdef / #if defined(): exclude code from compilation

Use when the guarded code must not compile at all when the option is off, for
example because it calls an API or references a field that only exists when
the option is enabled:

```c
#ifdef CONFIG_MYAPP_USE_SENSOR
static struct sensor_state state;   /* only exists when the feature is on */
#endif
```

`#if defined()` combines conditions:

```c
#if defined(CONFIG_MYAPP_USE_SENSOR) && !defined(CONFIG_MYAPP_SIMULATION)
```

### IS_ENABLED(): keep both branches compiling

`IS_ENABLED(CONFIG_X)` expands to `1` if the macro is defined to `1`, and `0`
otherwise, including when the macro is not defined at all. That makes it safe
on disabled bools (which are absent) and usable in ordinary C expressions:

```c
if (IS_ENABLED(CONFIG_MYAPP_USE_SENSOR)) {
    start_sensor();
}
```

The advantage over `#ifdef`: the compiler still parses and type-checks the
disabled branch, so it cannot silently rot; the optimizer then removes the
dead branch. Prefer this form whenever both branches can compile.

Upstream guidance (from the macro's own documentation in
`$ZEPHYR_BASE/include/zephyr/sys/util_macro.h`): using `IS_ENABLED` inside
`#if` is discouraged, it gives no benefit over plain `#if defined()`. Use
`IS_ENABLED` in C expressions, `defined()` in the preprocessor. The macro is
defined in `<zephyr/sys/util_macro.h>` and also available through
`<zephyr/sys/util.h>`.

## The bare CONFIG_X pitfall

Because a disabled bool is absent, using the macro bare goes wrong two ways:

- In C code, `if (CONFIG_MYAPP_USE_SENSOR)` fails to compile when the option
  is off: the macro does not exist, so the compiler sees an undeclared
  identifier. This is exactly the case `IS_ENABLED` exists for.
- In the preprocessor, `#if CONFIG_MYAPP_USE_SENSOR` appears to work because
  the preprocessor evaluates an unknown name as 0, but that same rule makes a
  typo in the symbol name behave identically to the option being off, with no
  diagnostic. Use `#ifdef` or `#if defined()` instead.

## Code-emission helpers (one-liners)

For cases where a declaration or statement itself must appear or disappear,
`<zephyr/sys/util_macro.h>` provides `IF_ENABLED(CONFIG_X, (code))` (emit code
when enabled) and `COND_CODE_1(CONFIG_X, (if_code), (else_code))` (pick one of
two code fragments); the code arguments must be in parentheses. The full macro
layer belongs to the zephyr-app-dev skill (planned); on this page it is enough
to know they exist.

## int, hex, and string symbols: use directly

Non-bool symbols present in `.config` expand to literals, usable in
initializers and expressions:

```c
int stack = CONFIG_MAIN_STACK_SIZE;          /* int: decimal literal */
uintptr_t off = CONFIG_FLASH_LOAD_OFFSET;    /* hex: 0x literal */
const char *name = CONFIG_BT_DEVICE_NAME;    /* string: C literal */
```

int and hex values also work in preprocessor arithmetic:

```c
#if CONFIG_LOG_DEFAULT_LEVEL > 2
#warning "verbose logging enabled"
#endif
```

Strings cannot be compared in `#if`; compare them at runtime or wrap the logic
in a bool symbol instead.

## CMake side: conditional sources

Gap-fill (verify against `$ZEPHYR_BASE/cmake/modules/extensions.cmake` in your
checkout): after `find_package(Zephyr ...)`, every CONFIG_ symbol is also a
CMake variable, and Zephyr ships `_ifdef` helpers to compile a file only when
a symbol is enabled:

```cmake
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(myapp)

target_sources(app PRIVATE src/main.c)
target_sources_ifdef(CONFIG_MYAPP_USE_SENSOR app PRIVATE src/sensor.c)
```

`zephyr_library_sources_ifdef(CONFIG_X file.c)` is the equivalent inside a
Zephyr library. Full CMakeLists authoring belongs to the zephyr-app-dev skill
(planned).

## Rebuild before you trust the header

Stale generated headers are the top cause of "I set CONFIG_X=y but the #ifdef
is still false". `autoconf.h` only changes when the configure stage re-merges
`.config`, and object files only pick it up when they recompile. After any
change to prj.conf, a fragment, or a Kconfig file, run a build (`west build`),
and when changing which config files are registered, force a pristine build
(`west build -p`). Then verify both layers:

```shell
grep "CONFIG_MYAPP_USE_SENSOR" <build>/zephyr/.config
grep "CONFIG_MYAPP_USE_SENSOR" <build>/zephyr/include/generated/zephyr/autoconf.h
```

If `.config` is right but `autoconf.h` is not, the configure/generation step
has not re-run; see `references/build-artifacts.md` for the staleness rules.

## Worked example: gate one feature three ways

The symbol, defined in the application's own Kconfig file (full authoring
rules in `references/kconfig-authoring.md`):

```kconfig
config MYAPP_USE_SENSOR
	bool "Enable the sensor feature"
```

Enabled in prj.conf:

```conf
CONFIG_MYAPP_USE_SENSOR=y
```

Way 1, exclude code that cannot compile when off:

```c
#ifdef CONFIG_MYAPP_USE_SENSOR
static struct sensor_ring_buffer rb;

void sensor_feed(void) { ring_push(&rb); }
#endif
```

Way 2, runtime branch that always compiles (preferred when possible):

```c
int main(void)
{
    if (IS_ENABLED(CONFIG_MYAPP_USE_SENSOR)) {
        sensor_start();
    }
    return 0;
}
```

Way 3, drop the whole file from the build (CMakeLists.txt):

```cmake
target_sources_ifdef(CONFIG_MYAPP_USE_SENSOR app PRIVATE src/sensor.c)
```

With way 3, `sensor_start()` must be declared unconditionally and either
guarded at call sites (way 1 or 2) or given a stub, otherwise disabling the
option turns the call into a link error.

Verify the chain end to end:

```shell
west build -b <board> <app-dir>
grep "MYAPP_USE_SENSOR" <build>/zephyr/.config
grep "MYAPP_USE_SENSOR" <build>/zephyr/include/generated/zephyr/autoconf.h
```

## Out of scope here

- The full utility-macro layer (`IF_ENABLED`, `COND_CODE_1`, listified
  iteration) and CMakeLists authoring: the zephyr-app-dev skill (planned).
- Reading devicetree data from C (`DT_*` macros, `DEVICE_DT_GET`): the
  zephyr-devicetree skill.
- Where `autoconf.h` sits among the other build artifacts and when it is
  regenerated: `references/build-artifacts.md`.
