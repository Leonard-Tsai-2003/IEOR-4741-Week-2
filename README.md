# HW 2 — Pointers, References & the Cost of a Copy

## Benchmark Results

### Table 1 — Swap Correctness
| Function | `a` Before | `b` Before | `a` After | `b` After | Result |
| :--- | :---: | :---: | :---: | :---: | :---: |
| `swap_ref` | 3 | 9 | 9 | 3 | OK |
| `swap_ptr` | 3 | 9 | 9 | 3 | OK |
| `swap_ptr(&a, &a)` | 5 | 5 | 5 | 5 | OK |

### Table 2 — Pass 648-Byte Struct: Value vs. Const Reference
| Variant | p50 (ns/call) | p99 (ns/call) | p99.9 (ns/call) | Mean (ns/call) |
| :--- | :---: | :---: | :---: | :---: |
| `sum_by_value(Big)` | 45.250 | 82.604 | 93.750 | 48.918 |
| `sum_by_cref(const Big&)` | 23.895 | 30.855 | 33.000 | 24.224 |

* **p50 Overhead Ratio (Value / Const Ref):** 1.89x
* **Correctness:** `sum_by_value` = 7191.0, `sum_by_cref` = 7191.0 (OK)

### Table 3 — Traverse 1,048,576 Ints: Contiguous Vector vs. Linked List
| Variant | p50 (ns/elem) | p99 (ns/elem) | p99.9 (ns/elem) | Mean (ns/elem) |
| :--- | :---: | :---: | :---: | :---: |
| `sum_vector` (Contiguous) | 0.043 | 0.071 | 0.149 | 0.047 |
| `sum_list` (Pointer Chase) | 10.409 | 14.107 | 76.515 | 10.994 |

* **p50 Latency Ratio (List / Vector):** 240.54x
* **Memory Footprint:** Vector = 4.0 MB, List = 16.0 MB
* **Correctness:** `sum_vector` = 549,755,289,600, `sum_list` = 549,755,289,600 (OK)

### Table 4 — Build & Environment
| Setting | Specification |
| :--- | :--- |
| **Compiler** | `clang++ 16.0.0` (`clang-1600.0.26.6`) |
| **Optimization** | `-O2` / `-O3` (Release) |
| **Language Standard** | C++17 (`__cplusplus` = `201703L`) |
| **Architecture** | `arm64` |
| **Clock Source** | `std::chrono::steady_clock` (Monotonic) |
| **Machine Specs** | Apple M2, 8 GB RAM, macOS Tahoe 26.6.2 |

---

## Technical Explanations

### 1. Inlining Effects and `HW2_NOINLINE`
Without `HW2_NOINLINE`, optimization (`-O2`) allows the compiler to inline `sum_by_value` directly at the caller site and eliminate the physical 648-byte stack copy entirely. This causes both rows to collapse onto identical execution times (~0 ns difference). `HW2_NOINLINE` forces an explicit ABI call boundary, isolating and accurately measuring the ~1.89x runtime overhead incurred by copying 648 bytes versus passing an 8-byte pointer address.

### 2. Cache Locality: Contiguous Array vs. Pointer Chasing
`sum_vector` traverses sequentially aligned 4-byte integers (4.0 MB total), enabling hardware prefetchers to stream cache lines directly into CPU L1/L2 caches ahead of execution (~0.043 ns/element). Conversely, `sum_list` traverses randomly permuted 16-byte nodes (16.0 MB total) by chasing next pointers; because addresses cannot be predicted by prefetching hardware, the execution incurs frequent memory stalls and cache misses, resulting in a ~240.54x performance penalty (~10.409 ns/element).