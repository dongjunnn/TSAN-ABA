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

Globs every `aba_*.c` test in the directory, so new tests run automatically —
nothing to edit here when a test file gets added.

```bash
CLANG=build/bin/clang
T=compiler-rt/test/tsan
for f in $T/aba_*.c; do
  t=$(basename "$f" .c)
  echo "=== $t ==="
  $CLANG -fsanitize=thread -O1 -g "$f" -o /tmp/$t && /tmp/$t
done
```

Expected (true positives fire, true negatives print only `done`):

- `aba_heap_reuse` (TP, heap) — `WARNING: ThreadSanitizer: ABA detected ...`
- `aba_hazard_pointer` (TN, heap) — `done`
- `aba_epoch_reclamation` (TN, heap) — `done`
- `aba_pool_reuse` (TP, custom-allocator annotation) — `WARNING: ThreadSanitizer: ABA detected ...`
- `aba_pool_hazard` (TN, custom-allocator annotation) — `done`

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
