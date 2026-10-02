# softticks_gcc

A GCC plugin that makes a program count its own progress in software ticks
(`include/rr_softticks.h`, ABI version 1). It places ticks by the same rules as the LLVM plugin
(`../llvm`).

## Usage

```sh
make -C tools/softticks gcc          # build/lib/rift/softticks_gcc.so
make -C tools/softticks check-gcc    # test/gcc/run.sh

gcc -fplugin=build/lib/rift/softticks_gcc.so -O2 -c foo.c
```

Pass the plugin to every compile and, with `-flto`, to the link as well (see below).

The plugin predefines `__RR_SOFTTICKS__` (the ABI version, `1`) in every translation unit
it is loaded into (C, C++, `-E`, and assembler-with-cpp `.S` files), so code it cannot
instrument (loops in inline assembly or hand-written assembly) can tick by hand only when
it is built for software ticks: `#ifdef __RR_SOFTTICKS__` … `RR_SOFTTICK()` (or the
`RR_SOFTTICKS_TICK_ASM` sequence in assembly) … `#endif`.

Options:

- `-fplugin-arg-softticks_gcc-stats` prints the number of tick sequences inserted per
  translation unit to stderr (`softticks_gcc: foo.c: 42 ticks`). (GCC takes a plugin
  argument's plugin name to end at the first `-`, which is why the file name has an
  underscore.)

## What is instrumented

The guarantee (`rr_softticks.h`): no instrumented instruction executes twice in the same
stack frame without a tick in between, so (ticks, registers) identifies a position. The
plugin places a tick

1. at the start of each function's first block (the single successor of the entry block),
   after its labels;
2. at the start of every block that is the target of a retreating edge of a depth-first
   walk of the CFG from the entry (`mark_dfs_back_edges`, computed afresh in the pass).
   Every cycle, reducible or not, contains a retreating edge, so every cycle contains a
   tick. A block gets at most one tick from 1 and 2: a first block that is also a loop
   header has one, which serves both;
3. right after every call to a `returns_twice` function (`ECF_RETURNS_TWICE`: `setjmp`,
   `_setjmp`, `sigsetjmp`, `vfork`, ...), where its first and every later return land.
   In GIMPLE such a call ends its block (it may return abnormally); the tick goes at the
   start of its normal successor if the call is that block's only predecessor (sharing
   a tick from 1 or 2 there), and on the edge (split by GCC) otherwise.

The DFS walks all edges: normal, abnormal (computed goto, setjmp, nonlocal goto) and EH. A
loop that goes round through a landing pad (an exception thrown and caught in a loop, a
retry loop) or through a computed goto (a threaded interpreter, whose computed gotos GCC
factors into one dispatch block) is a real cycle and must contain a tick; ignoring those
edges would leave such cycles unticked.

The one exception is GCC's abnormal dispatcher, an artificial block through which every call
that may longjmp or do a nonlocal goto has an abnormal edge to every setjmp call and nonlocal
label of the function. It holds no code, so when a retreating edge enters or leaves it the
cut goes where the jump lands at run time: right after the returns_twice call (rule 3
already puts a tick there; GCC models the later returns as an edge to the block that starts
with the call, so a tick at that block's start would not run) or at the start of a nonlocal
label's block (after a leading `__builtin_setjmp_receiver`).

Branches by themselves no longer tick (the plugins' earlier placement ticked every conditional
block). Recursion needs nothing more: each activation ticks at entry, and code that runs
again in the caller after a recursive call runs in another frame.

Each tick is the target's tick sequence (`RR_SOFTTICKS_{X64,X86,ARM64}_TICK_ASM`) as a
volatile `asm` with no operands, clobbering `"cc"` (x86) or `"x16", "x17"` (aarch64). A
translation unit with at least one tick also gets `RR_SOFTTICKS_*_INIT_ASM`, written to the
assembler output once at the end of the unit: the page-mapping initializer (on i386 it also
sets the disarmed word when it creates the page), its `.init_array` entry and the
`.note.rrsoftticks` note, all in the initializer's COMDAT group, so each linked module
(executable or shared library) has exactly one of each.

Targets: x86-64 (LP64), i386 (`-m32`) and aarch64 (LP64; untested, the plugin has not been
built for an aarch64 GCC). The target family comes from the configured target's headers the
plugin is built against; the ABI (`-m32`, `-mx32`, `-mabi=ilp32`) is checked per compile.
x32, aarch64 ILP32 and other targets get one warning and no instrumentation.

Skipped functions:

- `__attribute__((no_instrument_function))`;
- `__attribute__((naked))`;
- `__attribute__((disable_sanitizer_instrumentation))` (GCC 10 does not know it; the plugin
  registers it so that it is accepted);
- ifunc resolvers: the targets of the unit's `ifunc` attributes, which run during relocation
  before any initializer has mapped the page (functions they call are not skipped);
- functions in a section whose name starts with `.text.__rr_softticks`.

## Pass position

A GIMPLE pass right after `optimized` (the last GIMPLE pass, which runs at every
optimization level including `-O0`) and before RTL expansion, so the ticks are placed on the
optimized GIMPLE CFG: loops already rotated, unrolled or removed by the GIMPLE optimizers get
the ticks their final shape needs.

The RTL passes never delete a volatile asm and create no new cycles. They may duplicate a
tick (loop unrolling, tail duplication, block reordering, the duplication of a factored
computed goto) or merge copies (cross-jumping), but every execution of a ticking GIMPLE
block still executes exactly one copy of its tick, and every machine-level cycle still
passes one. Its dump is in `-fdump-tree-all` (`*.softticks`); `-fdump-tree-softticks` alone is
rejected because plugin passes are registered after the options are parsed.

## Tests

`test/gcc/run.sh` (`make check-gcc`): per-function tick counts in the assembly (`shapes.c`,
`shapes_eh.cc`, x86-64 and i386), GCC's IR checking (`-fchecking=2`) at every `-O` level,
exact execution counts at `-O0` (`counts.c`, `eh.cc`), the trap, the initializer and the
note per linked module (COMDAT), LTO, and the guarantee itself: `stepper.c` single-steps
`prop.c` and `prop_eh.cc` (loops, an irreducible cycle, recursion, a qsort callback,
setjmp/longjmp, a threaded interpreter, exceptions caught in loops) under ptrace and checks
that within each interval of constant countdown no (pc, sp) in the instrumented functions
repeats.

## LTO

With `-flto`, the compile step streams the IR before the plugin's pass, and the pass runs in
`lto1` during the link, which loads the plugin only when `-fplugin=` is given to the link
command too. Without it, slim LTO objects are not instrumented at all. Fat objects
(`-ffat-lto-objects`) hold instrumented machine code (for non-LTO links) and uninstrumented
IR, which an LTO link compiles and instruments once: nothing is instrumented twice. Each
LTRANS partition with ticks gets its own initializer, deduplicated by the linker.

## Limitations

- The plugin loads only into the GCC it was built against (the version check refuses any
  other); rebuild it for each compiler, with that compiler's plugin headers
  (`HOST_CC=/path/to/gcc`, Debian: `gcc-N-plugin-dev`).
- Debian's gcc-10-plugin-dev 10.2.1-6 lacks `common/config/i386/i386-cpuinfo.h`, which
  its `config/i386/i386.h` includes; `compat/` supplies the one type needed.
- Code that runs in a module before its initializer (ifunc resolvers' callees,
  `.preinit_array`, another module's constructors calling in early) faults outside rr if it
  is instrumented and no page is mapped yet.
- An ifunc resolver in another translation unit (or LTO partition) than its `ifunc` alias is
  not recognized.
- A `vfork` child shares its parent's memory, the countdown included: the tick after `vfork`
  (and any instrumented code the child runs before `exec`) counts against the parent's
  countdown.

