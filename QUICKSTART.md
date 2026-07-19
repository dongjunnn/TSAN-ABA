# TSan-ABA Quickstart

Fork: `github.com/dongjunnn/llvm-project`, branch `tsan-aba`
(based on upstream `8947e494f92e`, clang 23.0.0git).

## 1. Get the code

```bash
git clone git@github.com:dongjunnn/llvm-project.git
cd llvm-project
git checkout tsan-aba
```

## 2. Configure + first build (slow, one-off)

```bash
cmake -G Ninja -S llvm -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS=clang \
  -DLLVM_ENABLE_RUNTIMES=compiler-rt \
  -DLLVM_TARGETS_TO_BUILD=X86
ninja -C build
```

## 3. Rebuild after touching the TSan runtime (fast)

```bash
ninja -C build/runtimes/runtimes-bins tsan
```

Then copy the fresh archives into clang's resource dir — `-fsanitize=thread`
links from there, NOT from the build tree. Skipping this silently uses the
old runtime (see BUILD_AND_RUN.md §3):

```bash
RES=build/lib/clang/23/lib/x86_64-unknown-linux-gnu
SRC=build/runtimes/runtimes-bins/compiler-rt/lib/linux
cp $SRC/libclang_rt.tsan-x86_64.a     $RES/libclang_rt.tsan.a
cp $SRC/libclang_rt.tsan_cxx-x86_64.a $RES/libclang_rt.tsan_cxx.a
```

## 4. Run the tests

```bash
CLANG=build/bin/clang
T=compiler-rt/test/tsan
for t in aba_heap_reuse aba_hazard_pointer aba_epoch_reclamation; do
  echo "=== $t ==="
  $CLANG -fsanitize=thread -O1 -g $T/$t.c -o /tmp/$t && /tmp/$t
done
```

Expected:

- `aba_heap_reuse` (true positive) prints
  `WARNING: ThreadSanitizer: ABA detected (heap object identity change)`
- `aba_hazard_pointer` and `aba_epoch_reclamation` (true negatives) print
  only `done` — no warning.

## Known limitations (by design)

- 8-bit epoch cycles 1..255: a reuse after a multiple of 255 allocations is
  missed. Collisions are false negatives, never false positives.
- The per-thread cache keeps only the latest load per atomic address: a
  reload of the atomic between the original load and the CAS overwrites the
  cached epoch and blinds detection. The canonical load→CAS pattern
  (e.g. Treiber pop) is unaffected.
- Cache eviction between load and CAS → missed detection, never a false
  positive.
- Heap-reuse ABA only manifests if the allocator actually reuses the freed
  address; TSan's allocator does so reliably for same-size alloc/free.
