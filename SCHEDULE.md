# Phase 2: GIL-Free Threaded Image Operations

**The Validation Task**:
> To validate Phase 2, the developer must construct a hybrid Python-C++ module. Using Modern CMake, they will compile a Pybind11 shared object library (.so). The C++ library must implement a memory-safe RAII class that accepts a multi-dimensional NumPy tensor from Python. The C++ code must leverage multi-threading (via std::thread) to concurrently manipulate different quadrants of the image array in-place, bypassing the Python Global Interpreter Lock (GIL) entirely. The Python script will import this compiled library, pass a massive image payload, and verify that the transformation occurred instantaneously without any memory reallocation or copying overhead.

This phase proves that the developer can bridge the Python-C++ boundary at zero cost — no copies, no GIL contention, no safety violations — and exploit hardware parallelism directly from a Python caller.
**Goal**: Demonstrate mastery of native extension architecture: CMake-driven builds, RAII ownership semantics over foreign memory, GIL-free concurrency on shared mutable buffers, and empirical proof of zero-copy data flow.

---

## Day 0: Project Skeleton and Build Toolchain
**Focus**: Modern CMake, FetchContent, Pybind11 discovery, Python virtual environment, project layout
**Load**: Level 2

- **Tasks**:
    - [ ] Create the top-level directory layout: `src/` for C++ sources, `include/` for headers, `python/` for the driver script and tests, and `build/` (gitignored) for CMake artifacts.
    - [ ] Initialize a Python virtual environment via `uv venv` and install dependencies with `uv pip install numpy pytest pytest-benchmark`.
        - Note: Pin the NumPy version explicitly (e.g. `numpy>=1.24,<3`) so the ABI stays stable across rebuilds. Use `uv` for all Python package operations — not pip directly.
    - [ ] Author the root `CMakeLists.txt` — set `cmake_minimum_required` to 3.16+, set `CMAKE_CXX_STANDARD` to 17, and use `FetchContent` to pull `pybind11` from its GitHub release tag.
        - Note: Prefer `FetchContent_Declare` + `FetchContent_MakeAvailable` over git submodules — it keeps the repo clone shallow and avoids recursive-init footguns.
    - [ ] Add a minimal `src/module.cpp` containing a single `pybind11_add_module` target that exposes a no-op function returning a string literal.
    - [ ] Verify the full build-import cycle: run `cmake -B build && cmake --build build`, then `uv run python -c "import <module>; print(<module>.noop())"` and confirm the string prints.
    - [ ] Add `build/`, `*.so`, `__pycache__/`, and `.venv/` to `.gitignore`.

---

## Day 1: RAII Wrapper and Zero-Copy NumPy Buffer Binding
**Focus**: `py::array_t` buffer protocol, RAII resource semantics, NumPy stride arithmetic, `py::buffer_info`
**Load**: Level 3

- **Objectives**:
    1. A C++ RAII class owns a non-copying, mutable view into a caller-supplied NumPy array and validates its shape on construction.
    2. The class exposes raw pointer access and dimension metadata through a clean public interface — no Python types leak past the constructor.
    3. Python can instantiate the class, pass an `(H, W, C)` uint8 array, and read back the dimensions without any data duplication.

- **Tasks**:

    **RAII Class (C++)**
    - [ ] Define `ImageBuffer` in `include/image_buffer.hpp` — constructor accepts `py::array_t<uint8_t, py::array::c_style | py::array::forcecast>` and extracts `py::buffer_info` from it.
        - Note: The `py::array::c_style` flag guarantees contiguous row-major layout. Without it, a Fortran-order array silently produces garbage strides.
    - [ ] Store the raw `uint8_t*` data pointer plus `height`, `width`, and `channels` as private members. Compute and store `row_stride` (bytes per row) from `buffer_info.strides[0]`.
        - Note: Do **not** store `py::array_t` or `py::buffer_info` as members — the GIL must not be required to read these fields later during threaded work.
    - [ ] Assert in the constructor that `buffer_info.ndim == 3`, that `buffer_info.shape[2]` is 3 or 4 (RGB/RGBA), and that the buffer is writable (`buffer_info.readonly == false`). Throw `std::invalid_argument` on violation.
    - [ ] Implement a destructor that is explicitly defaulted (`= default`) — the class does **not** own the buffer's memory (NumPy does), so no deallocation occurs.
        - Note: This is the RAII discipline under test: the class manages *access lifetime*, not *allocation lifetime*. The destructor's job is to guarantee that no dangling work (threads) outlives the buffer view.

    **Pybind11 Binding**
    - [ ] In `src/module.cpp`, bind `ImageBuffer` via `py::class_<ImageBuffer>` with an `__init__` that accepts `py::array_t<uint8_t>`.
    - [ ] Expose read-only properties: `.height`, `.width`, `.channels`.
    - [ ] Bind a `data_ptr()` method that returns the raw pointer as `uintptr_t` — this is the diagnostic hook Python will use to prove zero-copy.

    **Verification Script**
    - [ ] Write `python/test_zerocopy.py`: allocate a NumPy array with `np.zeros((4096, 4096, 3), dtype=np.uint8)`, pass it to `ImageBuffer`, and assert that `buf.data_ptr() == arr.ctypes.data` — same integer address means zero copies.
    - [ ] Assert `buf.height == 4096`, `buf.width == 4096`, `buf.channels == 3`.

- **Acceptance Criteria**:
    - `uv run python python/test_zerocopy.py` exits 0 and prints no assertion errors.
    - The `data_ptr()` returned by the C++ side is byte-identical to `ndarray.ctypes.data` on the Python side — proving no intermediate copy was allocated.
    - Passing a 2D array (`np.zeros((100, 100))`) raises a Python `ValueError` originating from the C++ `std::invalid_argument`.

- **Resources**:
    - [NumPy — pybind11 documentation](https://pybind11.readthedocs.io/en/stable/advanced/pycpp/numpy.html) — Official guide to `py::array_t`, buffer protocol, and stride access. Read the **Buffer protocol** and **Arrays** sections for how `py::buffer_info` maps to NumPy internals; the **Vectorizing functions** section is irrelevant here.
    - [Object-oriented code — pybind11 documentation](https://pybind11.readthedocs.io/en/stable/classes.html) — Binding C++ classes to Python. Focus on **Creating bindings for a custom type** and **Instance and static fields** — these cover `py::class_`, `py::init`, and property definitions.
    - [The N-dimensional array (ndarray) — NumPy Manual](https://numpy.org/doc/stable/reference/arrays.ndarray.html) — NumPy's memory model. Read **Internal memory layout of an ndarray** for the stride formula and contiguity guarantees; skim **Array attributes** for `.ctypes`, `.strides`, `.flags`.
    - [RAII — cppreference.com](https://en.cppreference.com/w/cpp/language/raii.html) — Canonical definition of RAII. Read the full page — it's short. Pay attention to the **Standard library** section listing which STL types follow RAII, and the bad-vs-good mutex example.

---

## Day 2: Multi-Threaded Quadrant Transforms with GIL Release
**Focus**: `std::thread`, `py::gil_scoped_release`, quadrant partitioning, thread join safety, data races
**Load**: Level 4

- **Objectives**:
    1. A free function partitions the image into four quadrants and dispatches four `std::thread` instances — one per quadrant — that each apply an in-place pixel transformation concurrently.
    2. The GIL is released **before** threads are spawned and reacquired **after** all threads are joined — C++ threads never touch the Python API.
    3. Thread lifetime is fully RAII-managed: if any thread throws, all threads are joined before the exception propagates.

- **Tasks**:

    **Quadrant Logic (C++)**
    - [ ] Implement a standalone function `transform_quadrant(uint8_t* base, int row_start, int row_end, int width, int channels, int row_stride)` in `src/transforms.cpp` that applies a per-pixel operation (e.g., bitwise invert `pixel = 255 - pixel`) to every byte in the given row range.
        - Note: Keep the transform trivial. The point is proving concurrent memory access, not image processing. Inversion is ideal because it's visually verifiable and commutative (applying it twice restores the original).
    - [ ] Implement `process_quadrants(ImageBuffer& buf)` that computes the four row-ranges `[0, H/4)`, `[H/4, H/2)`, `[H/2, 3H/4)`, `[3H/4, H)`, constructs four `std::thread` objects each calling `transform_quadrant` with the appropriate slice, and joins all four.
        - Note: Use integer division. If height is not divisible by 4, the last quadrant absorbs the remainder rows — off-by-one here is a classic bug.
    - [ ] Wrap all four `std::thread` objects in a local `std::vector<std::thread>` and join them in a scope-exit loop (or use a small RAII thread-joiner wrapper). This guarantees no thread is left detached if an earlier join throws.

    **GIL Release (Binding)**
    - [ ] Bind `process_quadrants` in `src/module.cpp` using `py::call_guard<py::gil_scoped_release>()` so the GIL is released for the entire duration of the C++ call.
        - Note: `call_guard` is cleaner than manually scoping `py::gil_scoped_release release;` inside the function body. It applies RAII at the binding layer rather than polluting C++ logic.
    - [ ] Alternatively, if the function needs pre- or post-GIL work, use an explicit `py::gil_scoped_release` block within the bound lambda. Document which approach was chosen and why.

    **Thread Safety Audit**
    - [ ] Verify that no two threads write to overlapping row ranges — the quadrant boundaries must be non-overlapping and cover `[0, H)` exactly.
    - [ ] Confirm that `ImageBuffer` members read during threading (`data_ptr`, `width`, `channels`, `row_stride`) are `const` or effectively immutable after construction — no synchronization needed.
    - [ ] Verify that no thread calls any `py::` API — all parameters are raw C++ types (`uint8_t*`, `int`).

- **Acceptance Criteria**:
    - After calling `process_quadrants` from Python on a white `(4096, 4096, 3)` image (`np.full(..., 255)`), every pixel in the result array is `0` (bitwise inversion).
    - Calling `process_quadrants` twice restores the original array exactly (idempotency proof of inversion).
    - Running under `ThreadSanitizer` (`cmake -DCMAKE_CXX_FLAGS="-fsanitize=thread" ...`) reports zero data races.
    - The `data_ptr()` value is unchanged before and after `process_quadrants` — the buffer was mutated in-place, never reallocated.

- **Resources**:
    - [Miscellaneous — pybind11 documentation](https://pybind11.readthedocs.io/en/stable/advanced/misc.html) — GIL management in pybind11. Read the **Global Interpreter Lock (GIL)** section — it covers `gil_scoped_release`, `gil_scoped_acquire`, and `call_guard`. This is the single most critical section for this day's work.
    - [std::thread — cppreference.com](https://en.cppreference.com/w/cpp/thread/thread.html) — Full reference for `std::thread`. Read **Member functions** (constructor, `join`, `detach`) and note the precondition: destroying a joinable thread calls `std::terminate`.
    - [Build systems — pybind11 documentation](https://pybind11.readthedocs.io/en/stable/compiling.html) — CMake integration details. Read the **FetchContent** and **pybind11_add_module** sections for linking additional source files into the module target.
    - [pybind/cmake_example — GitHub](https://github.com/pybind/cmake_example) — Reference implementation of a CMake-based pybind11 project. Study the `CMakeLists.txt` for how `pybind11_add_module` is invoked and how source files are listed.

---

## Day 3: Performance Benchmarking and Validation Proof
**Focus**: Wall-clock timing, memory profiling, pytest-benchmark, scaling analysis, acceptance harness
**Load**: Level 3

- **Objectives**:
    1. A Python test suite quantitatively proves that the multi-threaded C++ path is faster than a single-threaded Python baseline on the same operation.
    2. Memory allocation is provably zero: the array's `ctypes.data` pointer and `__array_interface__` base address are unchanged across the entire pipeline.
    3. Thread scaling is measurable: benchmarks compare 1-thread vs 4-thread execution to show actual parallel speedup.

- **Tasks**:

    **Benchmark Harness (Python)**
    - [ ] Write `python/bench.py` using `time.perf_counter_ns()` to measure wall-clock time of `process_quadrants` on `(8192, 8192, 3)` uint8 arrays (192 MB payload). Print results in microseconds.
    - [ ] Implement a pure-Python baseline (`arr[:] = 255 - arr`) performing the same inversion, and time it identically. Print both results and the speedup ratio.
    - [ ] Add a `pytest-benchmark` test in `python/test_bench.py` that benchmarks both the C++ and Python paths using the `benchmark` fixture for statistically rigorous comparison (automatic calibration, warmup, percentile reporting).

    **Zero-Copy Proof**
    - [ ] In `python/test_validation.py`, capture `arr.ctypes.data` and `arr.__array_interface__['data'][0]` before and after calling `process_quadrants`. Assert both are identical — the same heap address, not a copy.
    - [ ] Assert `arr.base is None` (the array owns its memory) before the call, and `arr.base is None` after — confirming no view indirection was introduced.
    - [ ] Assert `np.shares_memory(arr, arr)` remains true and `arr.flags['OWNDATA']` is still `True` after the C++ call mutates it.

    **Scaling Test**
    - [ ] Add an optional `process_single_thread(ImageBuffer& buf)` binding that runs all four quadrants sequentially on one thread, for controlled comparison.
    - [ ] Benchmark 1-thread vs 4-thread on `(8192, 8192, 3)` and assert that the 4-thread path is at least 1.5x faster (conservative bound accounting for thread overhead on small payloads).

    **Full Acceptance Script**
    - [ ] Write `python/run_validation.py` that runs the complete validation sequence: construct array, confirm pointer, run transform, confirm pointer unchanged, confirm pixel values, print `PASS` or `FAIL` for each check.
    - [ ] The script must exit with code 0 only if every check passes.

- **Acceptance Criteria**:
    - `uv run pytest python/ -v` passes all tests with zero failures.
    - `uv run python python/bench.py` prints a measurable speedup (>1x) for the C++ threaded path over the Python baseline on an 8192x8192 image.
    - `uv run python python/run_validation.py` prints `PASS` for all checks: pointer stability, pixel correctness, double-inversion idempotency, and dimension integrity.
    - No test allocates a second array of the same size — memory high-water mark stays at ~1x the input payload (verify via `tracemalloc` or `/proc/self/status` VmRSS).

- **Resources**:
    - [pytest-benchmark documentation](https://pytest-benchmark.readthedocs.io/) — Benchmark fixture for pytest. Read the **Usage** section for how to pass callables to `benchmark()` and how to interpret the output table (min, max, mean, stddev, rounds).
    - [Google Benchmark User Guide](https://google.github.io/benchmark/user_guide.html) — C++-side microbenchmark reference. Read **Passing Arguments** and **Multithreaded Benchmarks** if you want to add C++ benchmarks alongside the Python ones; otherwise use this as a mental model for rigorous benchmarking methodology.
    - [The N-dimensional array (ndarray) — NumPy Manual](https://numpy.org/doc/stable/reference/arrays.ndarray.html) — Memory layout reference. Re-read **Internal memory layout of an ndarray** and **Array attributes** — you will need `ctypes.data`, `__array_interface__`, `flags`, and `base` to write the zero-copy assertions.
