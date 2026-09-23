# HW 2 — Pointers, References & the Cost of a Copy

## 1. Machine and Build Specifications

* **CPU:** Apple M2 (8-core CPU: 4 performance cores, 4 efficiency cores)
* **RAM:** 8 GB Unified Memory
* **Operating System:** macOS Tahoe 26.6.2
* **Compiler & Version:** `clang++ 16.0.0 (clang-1600.0.26.6)`
* **Compilation Flags:** `g++ -std=c++17 -O2 -Wall hw2.cpp -o hw2`
* **Timing Clock:** `std::chrono::steady_clock` (monotonic)
* **Warm-Up Protocol:** Warm-up batches (50 for Table 2, 5 for Table 3) were executed prior to measurement to bring CPU caches, branch predictors, and clock frequencies into steady state.
* **Sampling:** 1,000 samples (2,000 calls per sample) for Table 2; 200 samples (1 full traversal per sample) for Table 3. Results report percentile distributions (`p50`, `p99`, `p99.9`) and `mean`.
* **Noise Reduction:** All non-essential background applications (browsers, background services) were closed. The system was connected to AC power with power-saving modes disabled to prevent dynamic CPU frequency scaling during benchmarking.

---

## 2. Benchmark Results

### Table 1 — Swap Correctness
| Function | `a` Before | `b` Before | `a` After | `b` After | Result |
| :--- | :---: | :---: | :---: | :---: | :---: |
| `swap_ref` | 3 | 9 | 9 | 3 | OK |
| `swap_ptr` | 3 | 9 | 9 | 3 | OK |
| `swap_ptr(&a, &a)` | 5 | 5 | 5 | 5 | OK |

*Data source:*

### Table 2 — Pass 648-Byte Struct: Value vs. Const Reference
| Variant | p50 (ns/call) | p99 (ns/call) | p99.9 (ns/call) | Mean (ns/call) |
| :--- | :---: | :---: | :---: | :---: |
| `sum_by_value(Big)` | 45.250 | 82.604 | 93.750 | 48.918 |
| `sum_by_cref(const Big&)` | 23.895 | 30.855 | 33.000 | 24.224 |

* **p50 Ratio (Value / Const Ref):** 1.89x
* **Correctness:** `sum_by_value` = 7191.0, `sum_by_cref` = 7191.0 (OK)
* **Execution Parameters:** 1,000 samples x 2,000 calls

### Table 3 — Traverse 1,048,576 Ints: Contiguous Vector vs. Linked List
| Variant | p50 (ns/elem) | p99 (ns/elem) | p99.9 (ns/elem) | Mean (ns/elem) |
| :--- | :---: | :---: | :---: | :---: |
| `sum_vector` (Contiguous) | 0.043 | 0.071 | 0.149 | 0.047 |
| `sum_list` (Pointer Chase) | 10.409 | 14.107 | 76.515 | 10.994 |

* **p50 Ratio (List / Vector):** 240.54x
* **Bytes Touched:** Vector = 4.0 MB, List = 16.0 MB
* **Correctness:** `sum_vector` = 549,755,289,600, `sum_list` = 549,755,289,600, `expected` = 549,755,289,600 (OK)
* **Execution Parameters:** 200 samples x 1 full traversal

### Table 4 — Build & Method
| Metric | Specification |
| :--- | :--- |
| **Compiler** | `clang++ 16.0.0 (clang-1600.0.26.6)` |
| **Optimisation** | `-O2`/`-O3` (Release) OK |
| **Language Standard** | `__cplusplus` = `201703L` (C++17) |
| **Architecture** | `arm64` |
| **Clock Source** | `std::chrono::steady_clock` (monotonic) |
| **Warm-Up** | Yes — whole batches before the first sample |
| **Reported Metrics** | `p50` / `p99` / `p99.9` over samples, plus `mean` |
| **Dead-Code Guard** | `doNotOptimize()` on every result |
| **Machine Specs** | Apple M2, 8GB RAM, macOS Tahoe 26.6.2 |

---

## 3. Explanation A — `swap_ref` vs `swap_ptr`

In both `swap_ref` and `swap_ptr`, the underlying machine code passes an 8-byte memory address in a CPU register, making the argument passing cost identical (1 register / 8 bytes per argument). In `swap_ref`, `a` and `b` are syntax-level aliases for the caller's variables, so the compiler automatically dereferences them without explicit syntax. In `swap_ptr`, `a` and `b` are pointer variables holding raw addresses, requiring the explicit dereference operator (`*`) to access or modify caller values. If a developer writes `int* t = a; a = b; b = t;` inside `swap_ptr`, it merely swaps local copies of the pointer addresses on the function's stack frame, leaving the caller's memory completely unchanged.

Pointers explicitly express non-existence via `nullptr`, whereas references legally cannot be null in standard C++. Passing a `nullptr` to `swap_ptr` leads to undefined behavior (a segmentation fault) unless handled. In this implementation, explicit null checks were intentionally omitted to avoid conditional branch instructions on hot paths, shifting the obligation to caller contracts. For an HFT (High-Frequency Trading) hot path, `swap_ref` is superior: it guarantees non-nullability at compile time without branch predictor overhead (`if (!a) return;`), eliminates null-dereference crashes, and yields identical assembly performance with cleaner syntax.

---

## 4. Explanation B — Contiguous vs Pointer Chasing

Despite executing identical summation arithmetic over 1,048,576 integers, `sum_list` is ~240.54x slower than `sum_vector` due to memory hierarchy constraints:

* **Cache Lines & Spatial Locality:** CPU memory subsystems transfer data in 64-byte cache lines. `sum_vector` stores contiguous 4-byte integers; a single 64-byte cache line loads 16 contiguous integers, achieving 100% data payload efficiency (64 useful bytes out of 64 fetched). Conversely, `sum_list` uses 16-byte `Node` structures (`int v` + 4-byte padding + `Node* next`). Because nodes are linked in a shuffled order, a 64-byte line fetch retrieves a single node yielding only 4 useful integer bytes, resulting in 93.75% wasted bandwidth (60 useless bytes fetched per line).
* **Hardware Prefetcher:** The CPU hardware prefetcher detects the vector's continuous memory stride (+4 bytes) and speculatively populates L1/L2 cache lines before instructions request them. Linked list node addresses are randomly distributed across heap memory, rendering spatial prediction impossible and causing frequent cache misses.
* **Dependent-Load Chain:** Pointer chasing imposes a strict data dependency: the processor cannot determine the address of node $i+1$ until the load for `p->next` at node $i$ finishes. This prevents out-of-order execution engines from speculatively issuing parallel memory requests, forcing memory latency stalls to serialize sequentially instead of overlapping in flight.
* **Course Relevance:** This cache locality penalty is the foundational reason why high-performance trading engine structures—such as the Order Book in HW 7—use flat contiguous arrays rather than node-based structures like `std::map` or pointer-linked trees.

### Copy Cost Verification
For `Big` (648 bytes), `sum_by_value` takes 45.250 ns vs 23.895 ns for `sum_by_cref`, an overhead delta of $\approx 21.355 \text{ ns}$ per call to copy 648 bytes. This equates to a copy throughput of:

$$\text{Throughput} = \frac{648 \text{ Bytes}}{21.355 \times 10^{-9} \text{ Seconds}} \approx 30.35 \text{ GB/s}$$

This aligns closely with expected order-of-magnitude memory and L1 cache copy bandwidth on modern Apple Silicon hardware (~100 GB/s peak system bandwidth), proving that the measured delta directly reflects physical byte-copy overhead.