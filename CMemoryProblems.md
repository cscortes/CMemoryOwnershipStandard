# C Memory Allocation Problems

A taxonomy of bad memory usage patterns related to allocation in C. 34 distinct failure modes across 9 categories.

---

## Memory Allocation & Deallocation API Overview

### Standard C Library (`<stdlib.h>`)

| Function | Signature | Allocates | Initializes | Notes |
|----------|-----------|-----------|-------------|-------|
| `malloc` | `void *malloc(size_t size)` | Heap | No | Returns NULL on failure |
| `calloc` | `void *calloc(size_t nmemb, size_t size)` | Heap | Yes (zero) | Performs `nmemb * size` internally — safer for arrays |
| `realloc` | `void *realloc(void *ptr, size_t size)` | Heap | No (new tail) | NULL ptr → behaves like malloc; size 0 → impl-defined |
| `free` | `void free(void *ptr)` | — | — | NULL ptr is a no-op; ptr must be the original allocator pointer |

### C11 Aligned Allocation (`<stdlib.h>`, C11)

| Function | Signature | Notes |
|----------|-----------|-------|
| `aligned_alloc` | `void *aligned_alloc(size_t alignment, size_t size)` | `alignment` must be a power of two; `size` must be a multiple of `alignment` |

### POSIX Extensions

| Function | Signature | Notes |
|----------|-----------|-------|
| `posix_memalign` | `int posix_memalign(void **memptr, size_t alignment, size_t size)` | Returns error code; frees with `free()` |
| `reallocarray` | `void *reallocarray(void *ptr, size_t nmemb, size_t size)` | Overflow-safe `realloc`; available on BSD/Linux (glibc 2.26+) |

### Stack Allocation (Non-Standard)

| Function | Signature | Notes |
|----------|-----------|-------|
| `alloca` | `void *alloca(size_t size)` | Allocates on the **call stack** — no `free()` needed or allowed; freed on function return. Not in C standard; avoid in production |
| `_alloca` | `void *_alloca(size_t size)` | MSVC equivalent of `alloca` |

### POSIX Memory Mapping (`<sys/mman.h>`)

| Function | Signature | Notes |
|----------|-----------|-------|
| `mmap` | `void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset)` | Maps pages directly from OS; used for large or file-backed allocations |
| `munmap` | `int munmap(void *addr, size_t length)` | Must match `mmap` — do NOT pass to `free()` |

### Windows-Specific APIs

| Function | Pair | Notes |
|----------|------|-------|
| `HeapAlloc` / `HeapFree` | `HeapReAlloc` | Process-heap or custom heap; do NOT mix with `free()` |
| `VirtualAlloc` / `VirtualFree` | — | Page-granularity; used for large regions or guard pages |
| `LocalAlloc` / `LocalFree` | — | Legacy; avoid in new code |
| `GlobalAlloc` / `GlobalFree` | — | Legacy; avoid in new code |
| `CoTaskMemAlloc` / `CoTaskMemFree` | — | COM interop; required when crossing COM boundaries |

### C++ (Know the Boundary — Mismatching is UB)

| Allocator | Deallocator | Notes |
|-----------|-------------|-------|
| `new T` | `delete` | Single object |
| `new T[]` | `delete[]` | Array — mismatching `delete` vs `delete[]` is UB |
| `::operator new(size)` | `::operator delete` | Raw, non-constructing |

> **Rule:** Every allocation API has exactly one correct deallocation counterpart. Crossing them is undefined behavior (see problem #21).

### Deprecated / Avoid

| Function | Reason |
|----------|--------|
| `memalign` | Superseded by `posix_memalign` and `aligned_alloc` |
| `valloc` | Page-aligned but deprecated; not portable |
| `pvalloc` | Rounds up to page size; Linux-only, deprecated |
| `brk` / `sbrk` | Direct heap pointer manipulation; never use in application code |

---

## Category 1: Failure to Allocate

| # | Name | Description |
|---|------|-------------|
| 1 | **Null dereference after failed alloc** | Not checking the return of `malloc`/`calloc`/`realloc` before using the pointer |
| 2 | **Zero-size allocation** | Calling `malloc(0)` — behavior is implementation-defined; result may be NULL or a unique non-dereferenceable pointer |
| 3 | **Integer overflow in size calculation** | `malloc(count * sizeof(T))` where the multiplication wraps before being passed in |

---

## Category 2: Wrong Allocation Size

| # | Name | Description |
|---|------|-------------|
| 4 | **Off-by-one** | Allocating `n` bytes for a string that needs `n+1` (missing null terminator) |
| 5 | **Sizeof pointer instead of type** | `malloc(sizeof(ptr))` instead of `malloc(sizeof(*ptr))` |
| 6 | **Sizeof wrong type** | Allocating for a base type but storing a derived/larger struct |
| 7 | **Missing alignment padding** | Manual struct layout where field offsets don't account for alignment, then casting the pointer |

---

## Category 3: Use Before / Without Initialization

| # | Name | Description |
|---|------|-------------|
| 8 | **Uninitialized heap memory read** | Using `malloc`'d memory without initializing it (`malloc` does not zero; `calloc` does) |
| 9 | **Partial initialization** | Initializing some fields of a struct but reading unset fields later |
| 10 | **Stale data from realloc** | `realloc` grows a buffer; new tail bytes are uninitialized, but code reads them |

---

## Category 4: Memory Leaks

| # | Name | Description |
|---|------|-------------|
| 11 | **Plain leak** | Allocated pointer goes out of scope or is overwritten without being freed |
| 12 | **Lost pointer on realloc failure** | `ptr = realloc(ptr, n)` — if `realloc` returns NULL, original `ptr` is lost |
| 13 | **Early return without free** | Error-path `return` skips the `free` at the end of a function |
| 14 | **Leak in data structure** | Freeing a list head but not the nodes, or freeing a struct but not its pointer fields |
| 15 | **Ownership transferred but never freed** | Caller allocates, passes to callee which stores it; nobody frees it at end of life |
| 16 | **Cyclic reference** | Two heap objects point to each other; neither gets freed because both appear "in use" |

---

## Category 5: Invalid Frees

| # | Name | Description |
|---|------|-------------|
| 17 | **Double free** | `free()` called twice on the same pointer |
| 18 | **Free of stack memory** | `free()` called on a pointer to a local variable or a string literal |
| 19 | **Free of interior pointer** | `free(ptr + offset)` — must always free the original pointer returned by the allocator |
| 20 | **Free of non-heap global** | `free()` on a statically allocated buffer |
| 21 | **Mismatched allocator** | Allocated with `new` (C++) or a custom allocator, freed with `free`, or vice versa |

---

## Category 6: Use After Free

| # | Name | Description |
|---|------|-------------|
| 22 | **Classic use-after-free** | Reading or writing through a pointer after `free()` has been called on it |
| 23 | **Dangling pointer alias** | Two pointers to the same allocation; one path frees it, the other still uses it |
| 24 | **Returned pointer to freed memory** | Function frees internally, returns the pointer, caller dereferences it |
| 25 | **Callback fires after free** | Object is freed, but a registered callback still holds and dereferences a pointer to it |

---

## Category 7: Buffer Overflows / Out-of-Bounds

| # | Name | Description |
|---|------|-------------|
| 26 | **Heap buffer overflow (write)** | Writing past the end of an allocated region — corrupts heap metadata or adjacent objects |
| 27 | **Heap buffer underflow** | Writing before the start of the allocation (negative index) |
| 28 | **Read out of bounds** | Reading past the allocated size — information disclosure or crash |
| 29 | **Realloc shrink, stale size** | Buffer shrunk via `realloc`, but old size is still used as the access bound |

---

## Category 8: Type / Aliasing Violations

| # | Name | Description |
|---|------|-------------|
| 30 | **Type-punning via cast** | Casting `malloc`'d memory to an incompatible struct type and accessing it |
| 31 | **Strict aliasing violation** | Accessing the same memory through pointers of unrelated types (undefined behavior under C99+) |

---

## Category 9: Allocator Contract Violations

| # | Name | Description |
|---|------|-------------|
| 32 | **realloc on non-heap pointer** | Passing a stack or static pointer to `realloc` |
| 33 | **Using freed memory as realloc arg** | Passing an already-freed pointer to `realloc` |
| 34 | **Mixing allocators in a shared buffer** | Part of a buffer managed by one allocator, part by another |
