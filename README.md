# hipcc-parallel

A drop-in `hipcc` wrapper that compiles one source file for several GPU architectures
**in parallel** instead of one after another.

It handles both offloading drivers: it detects from `hipcc … -###` which pipeline the
compiler is running and splits that one. See *How it works*.

## The problem

`hipcc` walks its `--offload-arch` list sequentially. For a translation unit built for N
architectures the wall time is the sum over architectures, and the process sits at ~100%
CPU — one core — for the whole time.

Measured on a large C++ translation unit (heavy template instantiation, 4 architectures,
`-fgpu-rdc -Ofast`, 30-core EPYC 9534):

```
hipcc            1063.4 s     99% CPU
hipcc-parallel    288.0 s    387% CPU     3.7x
```

The per-architecture device compilations are independent, so this is pure serialization,
not work.

With `-fgpu-rdc` the compile only emits LLVM bitcode; the AMDGPU backend runs later, inside
`lld` at link time. The cost this wrapper removes is therefore the *frontend* — parsing and
instantiating the same headers once per architecture.

## How it works

It drives the same pipeline the clang driver uses internally, but concurrently. Which
pipeline that is depends on the driver, and the wrapper decides by reading back
`hipcc … -###` — which prints the commands the driver *would* run without running them.

### Old offloading driver (default through ROCm 10.0)

1. one `--offload-host-only` compile → host object (**once**, not once per architecture)
2. N `--offload-device-only --offload-arch=X` compiles → per-architecture bitcode, in parallel
3. `clang-offload-bundler -type=o` → the final object

All N+1 jobs are independent, so the wall time is `max(host, device)`.

The bundler binary, the `-targets` list and the compilation-unit id are **not guessed** —
they are read back from `-###`. This matters: the `clang-offload-bundler` on `PATH` may
belong to a different LLVM than the one `hipcc` uses, and the target spelling
(`hip-amdgcn-amd-amdhsa-unknown-gfx1100`, host entry last) is toolchain-specific.

### New offloading driver (`--offload-new-driver`; default from ROCm 10.1)

1. N `--offload-device-only --offload-arch=X` compiles → per-architecture bitcode, in parallel
2. `llvm-offload-binary` → one package holding all N device images
3. one `--offload-host-only` compile with that package embedded → the final object

The `--image=file=…,triple=…,arch=…,kind=hip` specs handed to the packager are copied
verbatim from `-###` except for the `file=` path. The triple, arch and kind spellings are
toolchain-specific (`gfx90a:sramecc+` target IDs included) and are not reconstructed.

`--offload-jobs=` / `-parallel-jobs=` are dropped from the sub-compiles. They only affect
the link, and on a `-c` compile they are either ignored or diagnosed as unused, once per
job.

## Correctness

Output is **byte-for-byte identical** to plain `hipcc`, for both pipelines — verified on an
82 MB object and on small test files, and re-verified after every change to the jobserver
logic:

```
hipcc -c -fgpu-rdc                      $ARCHS …   vs wrapper   → identical
hipcc -c -fgpu-rdc --offload-new-driver $ARCHS …   vs wrapper   → identical
```

One caveat, because it is easy to misread a `cmp`. Clang derives the HIP compilation-unit
id (`__hip_cuid_…`) by hashing the file path *and the whole command line*, and it **hashes
the `-cuid=` argument** rather than using it literally. Two compiles that differ only in
`-o` therefore get different cuids, and the objects differ in ~0.4% of their bytes —
symbol-table string offsets repack around the different hash string. Passing `-###` also
perturbs it, since `-###` is part of the hashed command line.

So the wrapper's cuid is internally consistent and unique per translation unit — which is
all the cuid must be — but is not the same *value* plain `hipcc` would pick. To compare
byte-for-byte, pin the same `-cuid=` on both sides and use the same `-o`:

```bash
hipcc          -c -fgpu-rdc -cuid=pinned $ARCHS x.hip -o same.o && mv same.o A.o
hipcc-parallel -c -fgpu-rdc -cuid=pinned $ARCHS x.hip -o same.o && mv same.o B.o
cmp A.o B.o      # identical
```

Getting the cuid wrong is not cosmetic: it is what ties device-side static variables to the
host code that registers them.

## Usage

It takes exactly the arguments `hipcc` takes:

```bash
hipcc-parallel -c -fgpu-rdc --offload-arch=gfx90a:sramecc+ --offload-arch=gfx942:sramecc+ \
               --offload-arch=gfx1100 --offload-arch=gfx1102 x.hip -o x.o
```

As a CMake compiler launcher:

```bash
cmake -DCMAKE_HIP_COMPILER_LAUNCHER=/path/to/hipcc-parallel/hipcc …
```

Build systems that refer to the compiler by absolute path will not pick it up from `PATH`;
point the relevant variable at it directly.

### Under GNU Make

Point the compile rule at the wrapper and mark the recipe recursive:

```make
-  $(CC_HIP) -c $(HIPFLAGS) -o $@ $<
+  +$(HIPCC_PARALLEL) -c $(HIPFLAGS) -o $@ $<
```

If the device link is also to run in parallel, add to the flags used by both the compile
and the link:

```
--offload-new-driver --offload-jobs=jobserver
```

and, if the build extracts device code out of objects itself, note that the new driver
stores it in a `.llvm.offloading` section rather than `__CLANG_OFFLOAD_BUNDLE*`:

```make
-  objcopy -j '__CLANG_OFFLOAD_BUNDLE*' $^ $@
+  objcopy -j '.llvm.offloading*' $^ $@
```

**The leading `+` on the compile recipe is not optional.** GNU Make only passes the
jobserver descriptors to recipes it considers recursive. Without it the wrapper cannot see
the jobserver and falls back (see below), and — once the driver itself does the
parallelising in 10.1 — LLVM's own jobserver client fails *open*, to every core on the
machine.

## GNU Make jobserver

Under a jobserver the wrapper is a cooperative client: it owns one implicit slot (the one
make already accounted for when it started the recipe) and acquires a token for each
*additional* concurrent job, returning them all on exit. A `-jN` build therefore stays
within `N`, instead of becoming `N × architectures` processes.

Both protocols are supported: the named pipe used by GNU Make ≥ 4.4
(`--jobserver-auth=fifo:PATH`) and the inherited fd pair used by older makes.

### Tokens are re-checked while the compile runs

The wrapper asks for tokens at startup, and then again every `HIPCC_PARALLEL_JOBSERVER_RETRY`
seconds (default 10), for as long as there is still an architecture waiting for a slot.

Since `wait` in bash has no timeout, the retry is implemented by putting a `sleep` in the
same job pool, so `wait -n` returns either when a compile finishes or when the interval
elapses, whichever comes first. Setting the interval to `0` restores the old
decide-once-at-startup behaviour.

Observed behaviour (GNU Make 4.3, 4 architectures → 5 jobs):

| context | result |
|---|---|
| standalone (no `MAKEFLAGS`) | 5 concurrent |
| `make -j4`, recipe marked `+` | 3 tokens → 4 concurrent |
| `make -j4`, ordinary recipe | fds not inherited → degrades to the `-j4` cap |
| `make -j1` | capped to 1 |
| `make -j2`, another recipe holding the only spare slot | starts at 1 concurrent, picks up `+1` within 10 s of that recipe finishing (185 s → 116 s; 39 s if started unconstrained) |

Three details worth knowing:

- **GNU Make only passes the jobserver fds to recipes it considers recursive** (prefixed
  `+`, or mentioning `$(MAKE)`). For an ordinary recipe it still advertises
  `--jobserver-auth=3,4` in `MAKEFLAGS` but closes the descriptors. A client that trusts
  `MAKEFLAGS` blocks forever; this wrapper validates the descriptors and falls back.
- **`make -j1` starts no jobserver at all.** Without a fallback the wrapper would run every
  architecture at once, the opposite of what `-j1` asks for, so when there is no usable
  jobserver it honours a numeric `-jN` from `MAKEFLAGS`. A bare `-j` means unlimited.
- **Tokens acquired late are still released on every exit path**, including signals: they
  are handed back only after the jobs they covered have actually died. Re-checking widens
  the window in which the wrapper holds tokens, so this matters more than it used to —
  the regression test below is the one that catches it.

Tokens are single bytes, and losing one poisons the build for every other process, so the
read consumes *exactly* one byte (`dd bs=1 count=1`, not a shell `read`, which may buffer
ahead) and each token is kept in its own file (a token byte may be NUL or unprintable).
The regression test for this is that a **follow-up** `make -j4` still completes — leaked
tokens hang the *next* target, not the current one.

## Signals and interruption

`^C` on the build must stop the compiles it started. Two things make that harder than it
looks, and both were getting in the way:

- **bash sets `SIGINT` and `SIGQUIT` to `SIG_IGN` in every `&` child of a non-interactive
  shell**, and an ignored disposition survives `exec`. The `^C` that the terminal delivers
  to make's foreground process group therefore reached make and the wrapper, but never the
  `hipcc` jobs: make printed `Error 130`, the shell prompt came back, and N device compiles
  kept running in the background, invisible and still burning cores.
- **Signalling the direct child is not enough**: `hipcc` is a driver that forks `clang`,
  which forks `cc1`. Killing the process we started leaves the grandchildren behind.

So the wrapper enables job control (`set -m`), which gives every job its own process group
and stops bash from ignoring `SIGINT` on its behalf. `INT`, `TERM`, `HUP` and `QUIT` are
forwarded to each job's *whole* group, the wrapper waits for them to actually exit (with a
`SIGKILL` fallback for anything that will not), and only then hands the jobserver tokens
back and removes the temporary directory — releasing a token while its job is still running
would over-subscribe the build, and removing the directory would pull the ground out from
under a live compile. Finally the wrapper re-raises the signal on itself, so make reports
`Interrupt` and stops the build rather than recording a recipe that merely failed.

Because the jobs are no longer in the terminal's foreground process group, `^Z` no longer
reaches them either, so `SIGTSTP`/`SIGCONT` are forwarded as well.

## Environment variables

| variable | effect |
|---|---|
| `HIPCC_REAL` | path to the real `hipcc` (default: the next `hipcc` in `PATH` that is not this script) |
| `HIPCC_PARALLEL_JOBS` | cap on concurrent device compiles (default: all, or as many as the jobserver grants) |
| `HIPCC_PARALLEL_JOBSERVER` | `0` to ignore the jobserver entirely |
| `HIPCC_PARALLEL_JOBSERVER_RETRY` | seconds between attempts to acquire another jobserver token while an architecture is still queued (default `10`; `0` decides once at startup) |
| `HIPCC_PARALLEL_DEDUP` | `0` to print every job's diagnostics verbatim instead of de-duplicating |
| `HIPCC_PARALLEL_DEBUG` | `1` to trace what the wrapper decides, and keep the temp directory |
| `HIPCC_PARALLEL_OFF` | `1` to disable the wrapper (straight passthrough) |

## When it does nothing

It delegates to plain `hipcc`, with the original arguments untouched, whenever it cannot
guarantee the same result:

- not an object compile (`-E`, `-S`, `-M`, `-MM`, `-fsyntax-only`)
- no `-c` (linking)
- no explicit `-o`
- fewer than two `--offload-arch` values
- the caller already passed `--offload-{host,device}-only`
- anything unexpected in the `-###` output: no usable bundler *and* no usable packager, a
  `-targets` list or `--image=` set that does not match the requested architectures, or a
  bundler/packager that fails at the end

## Notes and limitations

- **Diagnostics** from the host compile are printed once; each device job's output is
  filtered to the lines the host job did not already produce, prefixed with its
  architecture, so a warning does not appear N times. `HIPCC_PARALLEL_DEDUP=0` disables
  this.
- **`--offload-compress`** is handled, on the old-driver path, by mirroring whatever the
  driver passes to its own bundler. On ROCm 7.14 that is *nothing*: compression applies to
  the final fat binary at link time, and the compile-time object is byte-identical with and
  without the flag. Hardcoding `-compress` would produce an object the driver never would.
  On the new-driver path nothing is mirrored to `llvm-offload-binary` at all, for the same
  reason — `-###` passes it no compression arguments in RDC mode. If a future ROCm starts
  compressing at compile time, this is the first thing that will need revisiting.
- **The architecture list given to the host compile is inert** — the host object is
  byte-identical whatever it is, or even with none. It is passed only to stop the driver
  shelling out to `rocm_agent_enumerator` to autodetect a local GPU.
- The wrapper addresses **compilation only**. Where the device link dominates the build,
  parallelising the compile will not move it — and on a large RDC build it usually does
  dominate. Measured on a large multi-package build (4 architectures, `-j16`, 62 cores):
  one device link took 4619 s of a 5805 s build, 80% of the total, running single-threaded
  at the end with 61 cores idle. Parallelising every compile in that build was worth 1.13x;
  switching the *link* to the new driver with `--offload-jobs=jobserver` took it to 4.53x.
  The two compose — use both.
- **Peak memory goes up** once the link is parallel too: 12.2 GB against 5.1 GB on that
  build, all of it in the device link running four LTO codegens of the same module at once.
  It scales with architectures, not with `-jN`.

## Related work

[StreamHPC/phc](https://github.com/StreamHPC/phc) splits multi-architecture HIP compiles in
the same spirit, for the whole-program (non-RDC) model, and adds sccache and Ninja-trace
integration. It does not support `-fgpu-rdc`: given it, it emits a whole-program object with
an embedded `.hip_fatbin` rather than the device code an RDC link needs —
`__CLANG_OFFLOAD_BUNDLE__*` sections under the old driver, a `.llvm.offloading` section
under the new one.
