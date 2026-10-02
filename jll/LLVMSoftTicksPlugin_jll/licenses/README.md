# softticks_llvm: software ticks for clang/LLVM 19 (and 17)

A new-pass-manager plugin that makes a program count its own ticks
(`../include/rr_softticks.h`; the ABI version is its `RR_SOFTTICKS_ABI_VERSION`).

## Build and test

    make -C tools/softticks llvm         # build/lib/rift/softticks_llvm.so
    make -C tools/softticks check-llvm   # test/llvm/run.sh

`LLVM_VERSION` (default 19) or `LLVM_CONFIG` select the LLVM. The plugin is built with the host
g++ and takes LLVM's symbols from the process that loads it, so it only loads into the LLVM major
version it was built against, and only into a clang/opt/lld linked against a shared libLLVM (or
otherwise exporting LLVM's symbols).

The source also builds against LLVM 17 (BinaryBuilder2's Clang_jll 17, which ygglet's
`LLVMSoftTicksPlugin` recipe builds it for), and the tests pass there except `exec/lto-fat`,
which is skipped: clang 17 has no `-ffat-lto-objects`. With an LLVM tree that has no `FileCheck`
(Clang_jll), run `test/llvm/run.sh` with a `LLVM_BINDIR` that adds another LLVM's `FileCheck`.

## Usage

    clang -fpass-plugin=build/lib/rift/softticks_llvm.so -O2 -c foo.c
    opt -load-pass-plugin=build/lib/rift/softticks_llvm.so -passes=softticks foo.ll

A pass plugin runs after preprocessing, so unlike the GCC plugin it cannot predefine
`__RR_SOFTTICKS__` (for code that must tick by hand: see `rr_softticks.h`). Pass
`-D__RR_SOFTTICKS__=1` along with `-fpass-plugin`.

LTO: the plugin must also be loaded by the linker, the clang driver does not pass it on:

    clang -flto=thin -fpass-plugin=$SO -c foo.c
    clang -flto=thin -fuse-ld=lld -Wl,--load-pass-plugin=$SO foo.o

## What is instrumented

The guarantee: no instrumented instruction executes twice in the same stack frame without a tick
in between. (A pc can repeat without a tick only in different live activations, i.e. returning
through recursion, and the stack pointer tells those apart.) A tick is the
`RR_SOFTTICKS_*_TICK_ASM` sequence as `asm sideeffect`, with the flags clobber on x86 and x16/x17
on aarch64, no memory clobber, `nounwind`. In every instrumented function:

1. **Entry**: one tick in the entry block, after its leading `alloca`s (so that they stay static
   allocas in the prologue). Every call ticks at least once, so a function called again (also
   recursively, or as a callback from uninstrumented code such as libc's `qsort`) has ticked in
   between.
2. **Cycles**: one tick at the first insertion point (after phis and EH pads) of every block that
   is the target of a retreating edge of a depth-first walk of the CFG from the entry
   (`llvm::FindFunctionBackedges`). Every cycle reachable from the entry contains such an edge:
   take the cycle's block that the walk reaches first; its predecessor on the cycle is a
   descendant, so the edge into it retreats. This covers irreducible cycles, `indirectbr` (computed
   goto) and `callbr` (asm goto) cycles, and cycles through unwind edges. A block that is the
   target of several retreating edges ticks once. Unreachable blocks do not tick. (A
   `catchswitch` block has no insertion point; its successors tick instead. Linux targets do not
   use it.)
3. **returns_twice**: a tick right after every call to a `returns_twice` function (call-site or
   callee attribute: `setjmp`, `_setjmp`, `sigsetjmp`, `vfork`), in addition to 1 and 2, because a
   second return re-runs the code after the call without going around any CFG cycle. For an
   `invoke`, the tick goes at the start of the normal destination; if that block has other
   predecessors the edge is split first, so the tick runs only after the invoke. A `musttail` call
   is skipped (nothing may follow it).

Conditional blocks that are not on a cycle do not tick (this replaced the ABI v1 rule of one tick
per conditional block). At -O0 a counted `for` loop costs n + 1 ticks (its `for.cond` header,
the exiting test included) plus the function's entry tick; `test/llvm/exec/*.c` have the worked
counts.

On sqlite3.c (3.49.1, x86-64) the static tick sites went from 15624 (v1) to 4080 at -O0, and from
27126 to 4357 at -O2; the text grows by 4.4% (-O0) and 4.5% (-O2) instead of 17% and 28%.

A module with at least one tick also gets the initializer (`RR_SOFTTICKS_*_INIT_ASM`) as module
inline asm, once; it also carries the `.note.rrsoftticks` note in the initializer's COMDAT group,
so each linked module has one. A module whose functions are all skipped has no ticks, no
initializer and no note. Every processed module is marked with the module flag `rr.softticks` = ABI
version (behaviour `Max`); a marked module is never instrumented again.

Targets: ELF x86-64 (not x32), i386/i686, aarch64, from the module's triple. For anything else
the plugin warns once and leaves the module unchanged.

Skipped functions:

- declarations and `available_externally` functions;
- `naked` functions;
- `disable_sanitizer_instrumentation` functions (`__attribute__((disable_sanitizer_instrumentation))`;
  `ST_NOINSTR` in the tests). clang 19 leaves no trace of `no_instrument_function` in the IR
  unless `-finstrument-functions` is on, so that attribute is **not** honoured;
- IFUNC resolvers (they run during relocation, before the initializer);
- functions in a section starting with `.text.__rr_softticks`.

## Pipeline position

As late as the IR pipeline allows, so the count is a property of the optimized IR:

- normal compiles, all levels including -O0: the OptimizerLast extension point;
- full LTO: at the end of the link-time pipeline (FullLinkTimeOptimizationLast), -O0 included;
- ThinLTO: OptimizerLast of each post-link backend;
- LTO pre-link compiles (`-flto`, `-flto=thin`) are not instrumented;
- `-ffat-lto-objects`: the native code is instrumented, the embedded bitcode is not.

LLVM 19's OptimizerLast callback does not say which LTO phase it serves. The plugin tells them
apart as follows: the ThinLTO post-link pipeline is the only one with OptimizerLast that has no
PipelineStart; of the others, a module is a pre-link module if clang put the `EnableSplitLTOUnit`
or `ThinLTO` module flag on it (it does so before running the pre-link pipeline). Pre-link
pipelines run by `opt` on modules without those flags are therefore instrumented.

## Options

`-softticks-stats`: print the number of ticks (tick sites) and instrumented functions per module
to stderr.
With opt: `opt -load-pass-plugin=$SO -softticks-stats ...`. With clang the plugin must be loaded
before option parsing: `-fpass-plugin=$SO -Xclang -load -Xclang $SO -mllvm -softticks-stats`.

## Limitations

- ThinLTO backends at -O0 (`--lto-O0`, or linking with `-O0`) run no pipeline extension points
  in LLVM 19, so nothing is instrumented. Full LTO at -O0 works.
- lld does not deduplicate COMDAT groups in the objects produced by ThinLTO backends (the IR
  symbol table does not see groups defined in module asm), so a ThinLTO-linked module gets one
  `.init_array` entry per backend object that has ticks. All point at the same
  `__rr_softticks_init`, which is harmless to run again (EEXIST), but it is not "exactly one".
  The same applies when an LTO link mixes in natively compiled instrumented objects.
- Functions called by IFUNC resolvers, and any other code that runs before the module's
  initializer (e.g. called from another module's earlier constructor), are instrumented and fault
  outside rr if the page is not mapped yet.
- Code generation after the IR pipeline (block placement, tail duplication, branch folding) may
  change the machine CFG, but not the number of executed ticks: each tick is a side-effecting
  asm statement, executed exactly as often as its IR block (or its call site), and code
  generation creates no cycle that avoids it. `test/llvm/prop` checks the guarantee on the
  machine code by single-stepping (`stepper.c`): within each tick interval, no (pc, sp) in an
  instrumented function repeats, for x86-64 and i386 at -O0 and -O2.
- With LTO, lld does not deduplicate the note's COMDAT group in ThinLTO backend objects either,
  so a ThinLTO-linked module can carry several identical notes.

