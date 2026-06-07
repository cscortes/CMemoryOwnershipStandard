# C Naming Convention Standard

> For a taxonomy of the memory problems this standard is designed to prevent, see
> [CMemoryProblems.md](CMemoryProblems.md).

---

## Table of Contents

- [Principle](#principle)
- [Prerequisites](#prerequisites)
- [Part I — Function Naming Convention](#part-i--function-naming-convention)
  - [The Three Reserved Verbs](#the-three-reserved-verbs)
  - [The Observer Default](#the-observer-default)
  - [Module Lifecycle Convention](#module-lifecycle-convention)
  - [Edge Cases](#function-naming-edge-cases)
  - [Audit Rules](#audit-rules)
  - [Anti-Patterns](#function-naming-anti-patterns)
- [Part II — Variable Naming Convention](#part-ii--variable-naming-convention)
  - [Ownership Prefixes](#ownership-prefixes)
  - [Length Discipline](#length-discipline)
    - [sz — Null-Terminated Strings](#sz--null-terminated-strings)
    - [Buf and BufView — Binary Buffers](#buf-and-bufview--binary-buffers-as-mandatory-structs)
    - [The buf_* API](#the-buf_-api)
    - [ary — Typed Arrays](#ary--typed-arrays)
    - [The Binary/Text Discipline Rule](#the-binarytext-discipline-rule)
  - [Compound Prefix Reference](#compound-prefix-reference)
  - [Connection to Function Naming](#connection-to-function-naming)
  - [Edge Cases](#length-discipline-edge-cases)
  - [Anti-Patterns](#variable-naming-anti-patterns)
- [Part III — Safe Allocation Layer](#part-iii--safe-allocation-layer)
  - [Architecture](#architecture)
  - [Banned Routines](#banned-routines)
  - [mem_* API](#mem_-raw-untyped-allocation)
  - [psz_* API](#psz_-string-allocation)
    - [String Building and Formatting](#string-building-and-formatting)
  - [buf_* API](#buf_-binary-buffer-allocation)
  - [Enforcement: no_std_alloc.h](#the-forbidden-header-no_std_alloch)
  - [Build System Integration](#build-system-enforcement)
  - [Where stdlib Remains Permitted](#where-standard-routines-remain-permitted)
  - [Debug Extensions](#debug-extensions-optional)
- [Code Review Checklist](#code-review-checklist)
- [Glossary](#glossary)

---

## Principle

Every function either changes who owns a resource, or it does not. The name must make this unambiguous.

This is grounded in **linear type theory**: every owned resource has exactly one owner at any point in time. When that changes, the function name must say so.

---

## Prerequisites

### Target C Standard: C99

This standard targets **C99 (ISO/IEC 9899:1999)**. Every feature defined here — the `mem_*` API, `Buf`/`BufView` structs, `psz_*` API, and `no_std_alloc.h` — is implementable without any C11 or later features.

**Why C99 and not C11:**

C11 was considered and rejected as the baseline for two reasons. First, it offers no feature that this standard strictly requires — every mechanism here works in C99. Second, C99 has broader toolchain support, particularly on embedded targets (IAR, Keil/MDK, ARM Compiler 5, older GCC cross-compilers) where C11 adoption has been slower. Targeting C99 keeps the standard applicable to the widest range of projects.

C11 does offer improvements that are worth using *if your toolchain supports it*:

| C11 Feature | Benefit to This Standard | Required? |
|-------------|--------------------------|-----------|
| `_Static_assert` | Cleaner portable fallback in `no_std_alloc.h`; struct size assertions | No |
| VLAs made optional | C99 mandates VLA support; C11 makes it optional (VLAs are a footgun) | No |
| `<stdatomic.h>` | Required if reference counting is added later | No |

Compile with `--std=c99` (GCC/Clang) or `/TC /std:c11` (MSVC with C99 semantics). Do not rely on compiler defaults, which vary.

### Required Standard Headers

| Header | Used for |
|--------|----------|
| `<stdint.h>` | `uint8_t`, `int32_t`, `SIZE_MAX`, `UINT8_MAX` |
| `<stddef.h>` | `size_t`, `NULL`, `ptrdiff_t` |
| `<stdbool.h>` | `bool`, `true`, `false` |
| `<stdarg.h>` | `va_list`, `va_start`, `va_end`, `va_copy` |
| `<stdio.h>` | `vsnprintf` (string formatting in `psz_new_fmt`) |
| `<string.h>` | `memcpy`, `memmove`, `memset`, `memcmp`, `strlen` |
| `<assert.h>` | `assert` (debug-mode precondition checks) |
| `<stdlib.h>` | `malloc`, `calloc`, `realloc`, `free` — **`mem.c` only** |

### Compiler Requirements

| Compiler | Minimum Version | Notes |
|----------|-----------------|-------|
| GCC | 4.9 | Full C99; `__attribute__((error(...)))` available since GCC 4.3 |
| Clang | 3.3 | Full C99; `__attribute__((error(...)))` supported |
| MSVC | VS 2019 (19.20) | First version with reliable C99 coverage. Earlier versions lack `va_copy`, have broken `vsnprintf` semantics, and `<stdbool.h>` is incomplete. If VS2019+ is unavailable, provide compatibility shims for these three. |
| IAR C/C++ | 8.x | C99 mode via `--c99` flag |
| ARM Compiler | 6.x (armclang) | C99 via `--c99` |

### MSVC Specific Notes

MSVC prior to VS2019 has three concrete gaps relevant to this standard:

1. **`va_copy` is missing** — use `#define va_copy(dst, src) ((dst) = (src))` as a shim on MSVC < 2019. This works for x86/x64 where `va_list` is a pointer.
2. **`vsnprintf` returns -1 on truncation** (C89 behavior) instead of the required length — the two-pass length-measurement technique in `psz_new_fmt_v` breaks. Use `_vscprintf` on MSVC < 2019 to measure length.
3. **`<stdbool.h>` is missing on MSVC < 2013** — provide `typedef int bool; #define true 1; #define false 0`.

All three shims are isolated to a single compatibility header, not scattered through the codebase.

---

## Part I — Function Naming Convention

---

### The Three Reserved Verbs

Only three verbs carry ownership meaning. All other verbs are implicitly ownership-neutral.

| Verb | Ownership Effect | Caller Obligation |
|------|-----------------|-------------------|
| `_new` | Produces an owned resource | Must eventually call `_drop` or `_cleanup` |
| `_drop` | Consumes a heap-allocated resource | Resource is invalid after the call |
| `_move` | Transfers ownership from one variable to another | Source is invalid after the call |

#### Structure

```
<module>_<verb>[_<qualifier>]
```

```c
buf_new()                  // caller owns result
buf_new_clone(src)         // caller owns result; src is unaffected
buf_drop(buf)              // buf is freed and invalid after this
buf_move(dst, src)         // ownership of src transfers into dst; src is invalid after
```

---

### The Observer Default

Any function whose name contains none of `_new`, `_drop`, or `_move` makes **no ownership claim**. It borrows its arguments and returns either a value or a non-owned pointer.

```c
buf_get_len(buf)           // reads; no ownership change
buf_set_capacity(buf, n)   // mutates state; no ownership change
buf_is_empty(buf)          // predicate; no ownership change
buf_find(buf, val)         // search; no ownership change
```

Return type and `const` qualify the borrow direction:
- `const T *` — read-only borrow into internal state; caller must not free it
- `T *` — read-write borrow into internal state; caller must not free it
- `T` (value) — copy of data, no ownership concern

---

### Module Lifecycle Convention

Some modules require one-time global initialization before any instances can be created — loading a shared library, binding a port, seeding a PRNG, initializing a thread pool. These are not instance operations and do not produce an owned resource, so `_new` / `_drop` do not apply. They are not stack-struct operations, so `_init` / `_cleanup` do not apply either.

The reserved words for module-level lifecycle are `_startup` and `_shutdown`.

```c
// Initialize the network subsystem. Call once at program startup, before any net_* calls.
// Returns 0 on success, non-zero on failure (sets errno).
int  net_startup(void);

// Tear down the network subsystem. Call once at program exit, after all net_* objects are dropped.
// Idempotent — safe to call even if net_startup failed or was never called.
void net_shutdown(void);
```

**Rules:**

- `_startup` returns `int` (0 = success, non-zero = failure). It may fail — a port may be unavailable, a library may not load.
- `_shutdown` is always `void` and must be **idempotent**: calling it multiple times or after a failed `_startup` must not crash or corrupt state.
- Every `_startup` must have exactly one corresponding `_shutdown`.
- No other function in the module is safe to call before a successful `_startup`, or after `_shutdown`.
- `_startup` and `_shutdown` are **not thread-safe**. They are called from the main thread before worker threads start and after they stop.
- A module with no global state needs no `_startup` / `_shutdown`. The absence of these functions signals the module is stateless at the global level.

**Distinction from `_init` and `_new`:**

| Verb | Operates on | Allocates | Caller provides |
|------|------------|-----------|-----------------|
| `_new` | A new heap instance | Yes (heap struct) | Nothing |
| `_init` | A caller-provided stack struct | Yes (sub-resources) | Stack memory |
| `_startup` | Module-global state | Maybe (static/singleton) | Nothing |

```c
// Full lifecycle example
int main(void) {
    if (net_startup() != 0) { return 1; }
    if (crypto_startup() != 0) { net_shutdown(); return 1; }

    // ... application runs, net_* and crypto_* objects created and dropped ...

    crypto_shutdown();
    net_shutdown();
    return 0;
}
```

Shutdown is called in reverse startup order. If `_startup` for module B fails, only the modules successfully started before it are shut down.

---

### Function Naming Edge Cases

#### 1. Conditional Allocation

A function that sometimes allocates and sometimes returns an internal pointer is **forbidden** under this convention. It makes the caller's obligation ambiguous.

**Wrong:**
```c
// Returns internal pointer OR allocates depending on input — caller cannot know
char *buf_get_or_alloc_str(Buf *buf);
```

**Right:** split into two functions.
```c
const char *buf_get_str(Buf *buf);       // always a non-owned borrow
char       *buf_new_str_copy(Buf *buf);  // always a new owned copy
```

---

#### 2. Arrays of Owned Pointers

`_new` implies **deep ownership**: the caller owns the container and every element within it. `_drop` is responsible for freeing all levels.

```c
char **lines_new_from_file(const char *path);  // owns array and each string
void   lines_drop(char **lines, size_t count); // frees each string, then the array
```

The depth of ownership is an implementation detail of `_drop`. The caller's obligation is always the same: call `_drop` exactly once.

---

#### 3. Output Parameter Pattern

When the function allocates and returns via a pointer-to-pointer, it still produces an owned resource. Use `_new_into` to signal this.

```c
// *out receives a newly allocated Buf; caller owns it
int buf_new_into(Buf **out, size_t capacity);
```

When the caller provides the memory and the function only fills it, there is no ownership transfer — it is an observer and uses no reserved verb.

```c
// Caller owns buf; function writes into caller-provided memory
int buf_read_file(Buf *buf, const char *path);
```

---

#### 4. Stack-Allocated Structs

`_drop` always frees the struct itself (heap-allocated). For structs that live on the stack but hold heap-allocated sub-resources, use `_cleanup` instead. `_cleanup` releases sub-resources but does not free the struct.

```c
Buf buf;
buf_init(&buf, 1024);     // observer — fills caller-provided stack memory
buf_set_data(&buf, ptr);  // observer — sets a field
buf_cleanup(&buf);        // drops sub-resources only; buf itself is still valid stack memory
```

Rule: if you called `_new`, pair it with `_drop`. If you called `_init` on a stack variable, pair it with `_cleanup`.

---

#### 5. Sub-Resource Ownership Transfer

When a function takes ownership of one of its arguments (a field, a buffer, a string), qualify `_move` with the sub-resource name.

```c
// buf takes ownership of data — caller must NOT free data after this
void buf_move_data(Buf *buf, void *data, size_t len);
```

Reads as: "move data into buf's ownership." The caller's `data` pointer is invalid for freeing after this call.

To extract a sub-resource with ownership transfer (the inverse):

```c
// Caller receives ownership of buf's internal data — buf no longer owns it
void *buf_move_out_data(Buf *buf);
```

---

#### 6. Error-Returning Constructors

`_new` returns `NULL` on allocation failure. No drop is required for a `NULL` result — this is standard C.

```c
Buf *buf = buf_new(1024);
if (!buf) { return ERR_OOM; }   // no buf_drop needed here
// ...
buf_drop(buf);
```

---

#### 7. Returning Internal Pointers (Non-Owned Borrows)

Returning a pointer into internal state is an observer. The caller must never free the returned pointer, and its lifetime is bound to the owning struct.

```c
const uint8_t *buf_get_data(const Buf *buf);  // non-owned; invalid after buf_drop
```

If the caller needs the data to outlive the struct, they must clone it:

```c
uint8_t *buf_new_data_copy(const Buf *buf);   // owned copy; caller must free it
```

---

### Audit Rules

These three greps cover all ownership flow in a codebase:

```sh
grep -r '_new'   # every allocation site — each result must have a matching _drop or _cleanup
grep -r '_drop'  # every deallocation site
grep -r '_move'  # every ownership transfer
```

An unmatched `_new` with no corresponding `_drop` or `_cleanup` on every exit path is a leak. This is mechanically verifiable.

---

### Function Naming Anti-Patterns

| Pattern | Problem | Fix |
|---------|---------|-----|
| `buf_create` / `buf_destroy` | Not the reserved words; not grep-auditable as a system | Use `buf_new` / `buf_drop` |
| `buf_free` | Conflicts with stdlib `free`; ambiguous | Use `buf_drop` |
| `buf_get_or_alloc` | Ambiguous ownership — caller cannot know if they own the result | Split into `_get_*` and `_new_*` |
| `buf_copy` | Does "copy" produce an owned result? Not clear | Use `buf_new_clone` |
| `buf_transfer` | Not the reserved word for ownership transfer | Use `buf_move` |

---

## Part II — Variable Naming Convention

Function names encode what happens to ownership. Variable names encode **what kind of memory** the variable is and **who owns it**. Together they form a complete, readable contract at every call site.

The prefix system has two orthogonal axes: **ownership** (who is responsible for freeing) and **length discipline** (how the size of the data is known). These axes are independent and combine into compound prefixes.

---

### Ownership Prefixes

#### What Ownership Means in C

Every byte of heap memory has exactly one owner. The owner is the variable (or the struct containing the variable) responsible for eventually freeing that memory. In C there is no runtime enforcement — the convention is the only guard.

Ownership violations produce two classes of bug:
- **Double free**: two variables believe they own the same memory; both call free. Result: heap corruption, usually a crash or silent exploit vector.
- **Use-after-free**: the owner frees the memory but another variable still holds a pointer to it and reads or writes. Result: undefined behavior; often an exploitable vulnerability.

The prefix convention makes ownership visible at the declaration site and at every use site, so violations are catchable in code review without running the code.

#### The Three Ownership States

**`p` — Heap-allocated, owned**

The variable holds a pointer to memory that was obtained from `malloc`, `calloc`, or a `_new` function. This scope is responsible for freeing it exactly once. It must be paired with a `_drop` call on every exit path (normal return, early return, and error path).

```c
char    *pszName    = str_new_clone(vszInput);  // this scope owns pszName
uint8_t *pbufPacket = recv_new_packet(sock);    // this scope owns pbufPacket
size_t   pbufPacketLen = ...;

// ... use them ...

str_drop(pszName);          // exactly one drop
packet_drop(pbufPacket);    // exactly one drop
```

Copying a `p` pointer does not transfer ownership — it creates a second raw pointer to the same memory, which is illegal under this convention. To hand ownership to another scope, use a `_move` function. To give a temporary view to another scope, use a `v` variable.

**`v` — View, borrowed, not owned**

The variable holds a pointer into memory owned by someone else — a struct, another variable, a string literal, or a buffer allocated in a calling scope. This scope must never free it. Its lifetime is bounded by the owner's lifetime: once the owner calls `_drop`, this pointer is dangling and must not be used.

```c
const char *vszPath = config_get_path(cfg);    // cfg owns the string; we borrow it
                                                // do NOT free vszPath
                                                // do NOT use vszPath after cfg is dropped
```

`const` qualifies read-only views. A mutable view (`T *` without `const`) allows writing into the owner's memory; use sparingly and document the lifetime explicitly.

**No prefix — Stack or static value**

Stack-allocated variables and `static` variables carry no ownership prefix because they are not heap memory. They need no `_drop`. However, if the struct or array contains pointer fields that are heap-allocated, those sub-resources must be released with `_cleanup` when the containing scope exits.

```c
char    szTemp[256];     // stack, automatic storage — no drop needed
uint8_t bufHeader[16];   // stack binary buffer — no drop needed
size_t  bufHeaderLen = sizeof(bufHeader);

Buf buf;
buf_init(&buf, 1024);    // stack Buf; buf may hold internal heap sub-resources
// ...
buf_cleanup(&buf);       // releases sub-resources; does not free &buf (stack, not heap)
```

Static variables live for the program's entire lifetime. They are never freed and never need `_cleanup` unless explicitly designed as a one-time teardown.

#### Ownership State Transitions

| Transition | How | Result |
|-----------|-----|--------|
| Heap memory enters scope | Call `_new` | Declare as `p*` |
| Ownership transfers to another scope | Call `_move` | Caller's `p*` becomes invalid; receiver declares their own `p*` |
| Temporary borrow given to another scope | Pass pointer without `_move` | Receiver declares as `v*`; owner retains responsibility |
| Heap memory leaves scope | Call `_drop` | `p*` variable is invalid; must not be used |
| Sub-resources released (stack struct) | Call `_cleanup` | Struct fields pointing to heap are freed; struct itself remains valid |

#### Ownership Edge Cases

**Assigning a `p` to a `v`**: Legal — you are creating a view of memory you own. The view is only valid while you hold ownership.

```c
char       *pszMessage = str_new_clone(src);
const char *vszMessage = pszMessage;           // valid view into owned string

display(vszMessage);                           // fine — pszMessage still alive

str_drop(pszMessage);
// vszMessage is now dangling — must not be used past this point
```

**Passing a `p` into a function that takes `v`**: Always legal. You are lending what you own. The function must not free it.

**Returning a `v` from a function**: The returned view is valid only as long as the source it borrows from is alive. This must be documented. Name the function as an observer (no `_new`, `_drop`, or `_move`).

```c
const char *config_get_name(const Config *cfg);  // returns vsz; valid while cfg is alive
```

**Two `p` variables pointing to the same address**: Forbidden. Exactly one owner at any time. If you need shared access, one holder keeps `p`, all others use `v`.

**Null pointer**: A null `p` pointer means no resource was allocated (e.g., `_new` returned NULL on failure). No `_drop` is needed for a null pointer — both `free(NULL)` and well-written `_drop` functions are safe on NULL. Always check before use.

**`static` pointer to heap memory**: A `static char *pszConfig` that is allocated once and never freed is a legitimate pattern (singleton, cached resource). It is still `p` in the sense that it owns heap memory, but the drop never happens. Document this explicitly; it is the only acceptable exception to "every `p` must be dropped."

---

### Length Discipline

#### Why Length Discipline Exists

C has no universal way to know how many bytes a pointer covers. Two fundamentally different contracts exist in the C ecosystem:

- **Implicit length**: a sentinel byte (`\0`) marks the end. The length is found by scanning.
- **Explicit length**: a separate integer holds the byte count. The data has no terminator.

Mixing these disciplines is one of the most common sources of C vulnerabilities. A function written for one contract used with data following the other contract will either truncate data (if it trusts a terminator that does not exist as intended) or read past the end of the buffer (if it scans for a terminator that never comes, or comes late). The prefix makes the discipline visible.

#### `sz` — Null-Terminated Strings

`sz` stands for **string, zero-terminated**. The pointed-to data is a sequence of `char` values ending with a `\0` byte. The length is implicit: `strlen` scans forward until it finds `\0`.

**What it guarantees:**
- The `\0` byte exists somewhere in the allocated region.
- Every byte before `\0` is meaningful character data.
- No `\0` bytes appear in the middle of the data (if they did, functions reading up to `\0` would silently truncate).

**When to use `sz`:**
- Data is human-readable text (file paths, names, log messages, configuration values, command-line arguments).
- The data will be passed to C stdlib string functions (`strlen`, `strcpy`, `strcmp`, `strcat`, `strtol`, `printf %s`).
- The data is UTF-8: UTF-8 encoding guarantees that null bytes appear only as the null terminator, so UTF-8 text is `sz`-compatible.

**When NOT to use `sz`:**
- The data is binary (network packets, encrypted content, compressed data, serialized structs, image pixels, audio samples).
- The data comes from an external source and its content is not validated (it might contain embedded nulls).
- The length is provided by a protocol header rather than being determined by scanning.

**Length access cost**: O(n) — requires scanning every byte to find `\0`. For short strings this is negligible. For long strings in hot paths, cache the result of `strlen` in a local variable.

```c
// Normal sz usage
const char *vszGreeting = "Hello, world";          // string literal → vsz
char        szUsername[64];                         // stack buffer for user input
char       *pszFullPath = path_new_join(dir, file); // heap-owned → psz

size_t len = strlen(vszGreeting);   // O(n) scan; fine for short strings
printf("%s\n", pszFullPath);        // stdlib function expects sz contract

path_drop(pszFullPath);
```

#### `Buf` and `BufView` — Binary Buffers as Mandatory Structs

A binary buffer is **never** represented as a bare pointer. Pointer and length are inseparable — splitting them into two independent variables is the exact failure mode that creates orphaned lengths, mismatched units, and accidental `strlen` calls on binary data. The solution is to make the pairing structural and enforced by the type system.

Two types are defined for all binary buffer use:

```c
// Owns its heap-allocated data. Allocate with buf_new / buf_init. Free with buf_drop / buf_cleanup.
typedef struct {
    uint8_t *data;
    size_t   len;
} Buf;

// A non-owning window into any byte sequence. Always passed and stored by value. Never freed.
typedef struct {
    const uint8_t *data;
    size_t         len;
} BufView;
```

`Buf` is the owned form. `BufView` is the borrowed form. There is no third option — a bare `uint8_t *` with a separate length variable is a convention violation and a review error.

**What `Buf` guarantees:**
- `buf.data` points to a valid, heap-allocated byte array of exactly `buf.len` bytes.
- The allocation was produced by the `buf_*` API and must be released by the same API.
- No terminator is assumed. Every byte in `[buf.data, buf.data + buf.len)` is valid data.

**What `BufView` guarantees:**
- `view.data` points into memory owned by someone else — a `Buf`, a stack array, a string literal, or an external source.
- `view.len` reflects the number of valid bytes reachable from `view.data`.
- `BufView` is always passed by value (16 bytes on 64-bit). It is never heap-allocated and never freed.

**When to use `Buf` / `BufView`:**
- Data is binary: network packets, file contents, cryptographic keys, ciphertext, compressed streams, pixel data, audio samples.
- Data is text from an untrusted or external source (validate before promoting to `sz`).
- Data whose length comes from a protocol header rather than a null terminator.
- Any data that might contain `\0` bytes as valid content.

**When NOT to use `Buf` / `BufView`:**
- The data is definitively null-terminated text that will only be passed to C stdlib string functions. Use `sz` and the string APIs together.

---

#### The `buf_*` API

All `Buf` allocation goes through the `buf_*` module. Direct `malloc` calls for binary buffers are forbidden — they bypass the struct contract and return a bare pointer.

```c
// ── Heap allocation ──────────────────────────────────────────────────────────
// Caller receives a Buf * and must call buf_drop exactly once.

Buf    *buf_new(size_t len);
// Allocates len bytes. Data is uninitialized. Returns NULL on failure.

Buf    *buf_new_zeroed(size_t len);
// Allocates len bytes, zero-filled. Returns NULL on failure.

Buf    *buf_new_copy(const uint8_t *data, size_t len);
// Allocates len bytes and copies from data. Returns NULL on failure.

Buf    *buf_new_clone(const Buf *src);
// Allocates and copies all bytes from src. Returns NULL on failure.

Buf    *buf_new_from_view(BufView view);
// Allocates and copies all bytes described by view. Returns NULL on failure.

void    buf_drop(Buf *buf);
// Frees buf->data, then frees the Buf struct itself. Safe on NULL.


// ── Stack allocation ─────────────────────────────────────────────────────────
// Caller provides stack-allocated Buf. Must call buf_cleanup exactly once.

void    buf_init(Buf *buf, size_t len);
// Allocates len bytes into buf->data. buf itself is caller-managed (stack).

void    buf_init_zeroed(Buf *buf, size_t len);
// Allocates len zero-filled bytes into buf->data.

void    buf_cleanup(Buf *buf);
// Frees buf->data. Does NOT free buf itself (it is on the stack).


// ── Resize ───────────────────────────────────────────────────────────────────
// Invalidates all BufView values derived from this Buf.

int     buf_resize(Buf *buf, size_t newLen);
// Reallocates buf->data to newLen bytes. Updates buf->len. Returns 0 on success.


// ── BufView construction (no allocation; returns by value) ───────────────────

BufView buf_view(const Buf *buf);
// Returns a BufView covering all of buf.

BufView buf_slice(const Buf *buf, size_t offset, size_t len);
// Returns a BufView into buf starting at offset. Asserts offset + len <= buf->len.

BufView bufview_of(const uint8_t *data, size_t len);
// Wraps a raw pointer + length into a BufView. No allocation.

BufView bufview_of_sz(const char *sz);
// Wraps a null-terminated string as a BufView over its bytes (excludes the '\0').


// ── Comparison ───────────────────────────────────────────────────────────────

bool    buf_equals(BufView a, BufView b);
// True if a.len == b.len and all bytes match.

int     buf_compare(BufView a, BufView b);
// memcmp semantics: negative, zero, or positive.
```

**Usage examples:**

```c
// Heap-owned Buf: produced by _new, released by _drop
Buf *pCiphertext = crypto_new_encrypt(buf_view(pPlain), key);
if (!pCiphertext) { return ERR_OOM; }

memcpy(dest, pCiphertext->data, pCiphertext->len);  // explicit length — correct

crypto_drop(pCiphertext);


// Stack Buf: initialized with buf_init, released with buf_cleanup
Buf header;
buf_init_zeroed(&header, 16);

memcpy(header.data, &magic, 4);
header.data[4] = version;

send(sock, header.data, header.len, 0);
buf_cleanup(&header);


// BufView: wraps existing memory; no allocation; no cleanup
BufView vPacket = bufview_of(recv_buf, recv_len);
BufView vPayload = buf_slice(pFrame, 4, pFrame->len - 4);

process_packet(vPayload);   // function receives by value — cheap, safe
```

#### `ary` — Typed Arrays

`ary` means **a contiguous sequence of typed elements** with an explicit element count. It differs from `buf` in that the unit of measurement is elements, not bytes. The size in bytes is always `count * sizeof(ElementType)`.

**When to use `ary`:**
- Arrays of structs, pointers, integers, or any typed element.
- The elements are individually addressable by index.
- The operation is "iterate over N elements of type T", not "process N raw bytes".

**Mandatory companion rule**: Every `ary` variable must be accompanied by a `Count` variable with the same prefix and base name.

```c
Token   *paryTokens;        // heap-owned array of Token structs
size_t   paryTokensCount;   // number of Token elements (not bytes)

paryTokens      = lexer_new_tokenize(vszSource);
paryTokensCount = lexer_get_last_count();

for (size_t i = 0; i < paryTokensCount; i++) {
    process_token(&paryTokens[i]);
}

lexer_drop(paryTokens);
```

#### The Binary/Text Discipline Rule

`sz` and `buf` encode fundamentally different contracts and are **mutually exclusive**. Crossing this boundary is the direct cause of most C buffer overflow and truncation bugs.

**The rule**: If a byte sequence may contain `\0` bytes that are valid data, or if its length is determined by anything other than scanning for `\0`, it is `buf`. No exceptions.

**Why this matters — the classic mistakes:**

```c
// Scenario: receiving a network packet that happens to contain a null byte
uint8_t received[1024];
ssize_t receivedLen = recv(sock, received, sizeof(received), 0);

// WRONG — treats binary packet as a string
char  *pszPacket = (char *)received;
size_t wrongLen  = strlen(pszPacket);  // stops at first 0x00 byte in the packet
                                        // wrongLen may be far less than receivedLen
                                        // bytes after the first 0x00 are silently lost

// RIGHT — wrap in BufView; length is always explicit
BufView vPacket = bufview_of(received, (size_t)receivedLen);
process_packet(vPacket);               // all bytes processed; len travels with the data
```

```c
// Scenario: reading a file that contains binary data
FILE *f = fopen("data.bin", "rb");

// WRONG — stack array named sz; file data is binary, not a string
char   szFileBuf[4096];
size_t n = fread(szFileBuf, 1, sizeof(szFileBuf), f);
printf("File content: %s\n", szFileBuf);  // printf stops at first 0x00; rest invisible

// RIGHT — stack Buf via buf_init; len reflects actual bytes read
Buf    fileBuf;
buf_init(&fileBuf, 4096);
fileBuf.len = fread(fileBuf.data, 1, fileBuf.len, f);
fwrite(fileBuf.data, 1, fileBuf.len, stdout);   // all bytes written
buf_cleanup(&fileBuf);
```

**When text has a known length**: Even if you are working with text and you know its length from a protocol header, use `BufView` — not `sz` — because the source of truth for length is the header, not a null terminator. Promote to `sz` only after validating that no embedded nulls exist and the null terminator is genuinely the contract.

```c
// Protocol sends: 2-byte length prefix + UTF-8 text (may or may not be null-terminated)
uint16_t headerLen = ntohs(*(uint16_t *)cursor);
cursor += 2;

// Wrap in BufView — length comes from the protocol, not from scanning for '\0'
BufView vText = bufview_of(cursor, headerLen);

// If you need to pass to a C string API, first validate and copy:
char *pszText = str_new_from_view(vText);   // validates no embedded nulls, appends '\0'
if (!pszText) { return ERR_INVALID_TEXT; }
printf("%s\n", pszText);
str_drop(pszText);
```

---

### Compound Prefix Reference

Ownership and length discipline combine into a fixed set of prefixes and types. There are no other valid combinations.

| Prefix | Ownership | Discipline | Declaration form | Notes |
|--------|-----------|-----------|-----------------|-------|
| `p` | heap-owned | single struct/value | `T *pFoo` | Must `_drop` |
| `psz` | heap-owned | null-terminated string | `char *pszFoo` | Must `_drop` or `free` |
| `p` on `Buf *` | heap-owned | binary buffer | `Buf *pFoo` | Must `buf_drop`; len in struct |
| `pary` | heap-owned | typed array | `T *paryFoo` | Must `_drop`; pair with `paryFooCount` |
| `v` | borrowed | single struct/value | `const T *vFoo` | Must NOT free |
| `vsz` | borrowed | null-terminated string | `const char *vszFoo` | Must NOT free |
| `v` on `BufView` | borrowed | binary buffer | `BufView vFoo` | By value; len in struct; never freed |
| `vary` | borrowed | typed array | `const T *varyFoo` | Must NOT free; pair with `varyFooCount` |
| *(none)* | stack/static | null-terminated string | `char szFoo[N]` | No drop; `_cleanup` if sub-resources |
| *(none)* on `Buf` | stack | binary buffer | `Buf foo` | `buf_cleanup`; len in struct |
| *(none)* | stack/static | typed array | `T aryFoo[N]` | No drop; pair with `aryFooCount` |
| *(none)* | stack/static | single struct/scalar | `T foo` | No drop; `_cleanup` if sub-resources |

The key change from earlier conventions: **binary buffers are always `Buf` or `BufView` structs**. A bare `uint8_t *` with a separate length variable is a violation. The struct enforces that data and length travel together.

```c
// Full example showing all forms in context
void process_request(const Config *cfg) {

    // Borrowed views — v prefix; no drop needed
    const char *vszEndpoint = config_get_endpoint(cfg);  // borrowed sz
    BufView     vSecret     = config_get_key(cfg);       // borrowed binary view

    // Stack values for intermediate work — no prefix
    char szLogMsg[256];   // stack sz
    Buf  nonce;
    buf_init_zeroed(&nonce, 16);  // stack Buf; buf_cleanup when done

    // Heap-owned results from _new — p prefix; must drop
    char *pszUrl     = url_new_build(vszEndpoint, "/api");  // owned sz
    Buf  *pPayload   = payload_new_encode(vSecret);         // owned binary Buf

    snprintf(szLogMsg, sizeof(szLogMsg), "Sending to %s", pszUrl);
    log_info(szLogMsg);

    send_request(pszUrl, buf_view(pPayload));

    buf_cleanup(&nonce);
    url_drop(pszUrl);
    buf_drop(pPayload);
}
```

---

### Connection to Function Naming

The variable prefix is determined by which function produced the value. This is not coincidence — it is the design. The function naming convention and the variable naming convention form one coherent system.

| Function verb | Variable form | Contract |
|---------------|---------------|---------|
| `_new` returning `T *` | `T *pFoo` | Caller owns — must `_drop` |
| `_new` returning `Buf *` | `Buf *pFoo` | Caller owns — must `buf_drop` |
| `_new` returning `char *` | `char *pszFoo` | Caller owns — must `_drop` or `free` |
| Any observer returning `const T *` | `const T *vFoo` | Borrowed — must NOT drop |
| Any observer returning `BufView` | `BufView vFoo` | Borrowed by value — must NOT free |
| Any observer returning `const char *` | `const char *vszFoo` | Borrowed — must NOT free |
| `_init` on a stack struct | `T foo` / `Buf foo` | Stack — `_cleanup` or `buf_cleanup` |
| `_move` | receiver takes `p*`; source's pointer is invalid | Ownership transferred |

A mismatch between the producing function and the receiving variable is always a bug or a naming error:

```c
// Correct — _new produces owned Buf → Buf *p prefix
Buf *pPayload = packet_new_encode(&msg);
send(sock, pPayload->data, pPayload->len, 0);
buf_drop(pPayload);

// Correct — _new produces owned sz → char *psz prefix
char *pszName = user_new_name(id);
printf("%s\n", pszName);
user_drop_name(pszName);

// BUG — observer returns BufView (borrowed) but assigned to Buf * (implies ownership)
Buf *pKey = config_get_key(cfg);   // wrong type — config_get_key returns BufView
buf_drop(pKey);                     // WRONG — double free or invalid free

// Correct — observer returns BufView → BufView v prefix
BufView vKey = config_get_key(cfg);
// no free — we borrow from cfg; cfg_drop will release the underlying data

// BUG — _get returns vsz but named psz (implies ownership)
char *pszPath = config_get_path(cfg);  // wrong prefix — this is a vsz
free(pszPath);                          // WRONG — invalid free

// Correct
const char *vszPath = config_get_path(cfg);
// no free
```

---

### Length Discipline Edge Cases

**UTF-8 text is `sz`-compatible**: UTF-8 encodes all non-null characters without using the byte `0x00`. The null terminator remains unambiguous. UTF-8 strings are safe to use as `sz`. However, character count ≠ byte count — use `strlen` for bytes, not characters.

**ASCII text from a trusted source is `sz`**: Command-line arguments, environment variables, and string literals are null-terminated by the platform or compiler. These are `vsz` (borrowed views into the process's own memory).

**Text received over a network is `buf` until validated**: The remote peer may send embedded null bytes, whether accidentally or maliciously. Treat all externally received text as `buf` first. After validating there are no embedded nulls and the content is the expected encoding, it is safe to promote to `sz`.

```c
// Received from network → BufView first (borrowed from the socket's internal buffer)
BufView vInput = socket_get_recv_view(sock);

// Validate and promote to sz for display
char *pszDisplay = str_new_from_view(vInput);   // validates no embedded nulls, appends '\0'
if (!pszDisplay) {
    log_error("Input contains embedded nulls or invalid encoding");
    return ERR_BAD_INPUT;
}
printf("Input: %s\n", pszDisplay);
str_drop(pszDisplay);
```

**Empty buffers and zero-length strings**: Both are valid. A `Buf` with `len = 0` is a legal zero-length allocation. A `char *pszEmpty` pointing to a single `\0` byte is a valid empty string. Handle both without special-casing unless the API explicitly prohibits them.

**Slicing a `Buf` into a `BufView`**: Use `buf_slice`. The view's `len` must not exceed the remaining bytes past the offset. `buf_slice` asserts this in debug builds.

```c
Buf *pFrame = recv_new_frame(sock);
// pFrame->data: [4-byte header][payload bytes...]

// Skip the header; view only the payload
BufView vPayload = buf_slice(pFrame, 4, pFrame->len - 4);
process_payload(vPayload);

buf_drop(pFrame);
// vPayload is now dangling — must not be used past this point
```

**Resizing a `Buf`**: Use `buf_resize`. It calls `realloc` internally and updates `buf->len`. Any `BufView` derived from the old allocation is dangling after `buf_resize` — all views must be re-derived from the updated `buf->data` pointer afterward.

```c
Buf *pAccum = buf_new_zeroed(64);

// ... fill pAccum ...

if (need_more_space) {
    if (buf_resize(pAccum, 128) != 0) { buf_drop(pAccum); return ERR_OOM; }
    // Any BufView into pAccum taken before this point is now invalid.
    // Re-derive views after resize:
    BufView vWindow = buf_view(pAccum);
}

buf_drop(pAccum);
```

---

### Variable Naming Anti-Patterns

| Pattern | Problem | Fix |
|---------|---------|-----|
| `uint8_t *data; size_t dataLen;` | Bare pointer + separate length — enforcement is convention only | Use `Buf *pData` or `BufView vData` |
| `uint8_t *data` alone | No length at all — unrepresentable without the struct | Use `Buf *pData` (owned) or `BufView vData` (borrowed) |
| `strlen(buf.data)` | String length on binary data — stops at first `0x00` | Use `buf.len` |
| `char *pszFoo` containing binary bytes | Embedded nulls silently truncate; string APIs misbehave | Reclassify as `Buf *pFoo` |
| `char *pPath = config_get_path(cfg)` | `p` implies ownership; observer returns a borrow | Use `const char *vszPath` |
| `BufView vSlice` used after `buf_drop(pOwner)` | Dangling pointer; `pOwner->data` was freed | Restructure lifetimes or clone into `Buf *pCopy = buf_new_from_view(vSlice)` |
| `BufView vOld` used after `buf_resize(pBuf, n)` | `realloc` may move the allocation; old data pointer is invalid | Re-derive with `buf_view(pBuf)` after resize |
| `Buf *pA = pB` (pointer copy) | Two `p` variables claim ownership of the same Buf | Second user must be `BufView vA = buf_view(pB)` |
| `buf.data` passed to a `sz` API directly | Binary data fed to `strlen`, `printf %s`, `strcpy` | Promote to `sz` via `str_new_from_view(buf_view(&buf))` after validation |
| `malloc(n)` used directly for binary data | Bypasses the `buf_*` API; returns bare pointer with no length | Use `buf_new(n)` |

---

## Part III — Safe Allocation Layer

Direct use of `malloc`, `calloc`, `realloc`, `free`, `strdup`, and `strndup` is **banned** in all application code. They return bare pointers with no length, no type context, and no ownership signal — everything this convention exists to prevent.

A three-layer API replaces them. Each layer is built on the one below it. Only the bottom layer may touch the C standard allocation routines, and only in one translation unit.

### Architecture

```
┌─────────────────────────────────────────────────┐
│  User code / module APIs                         │
│  buf_new, buf_drop, foo_new, foo_drop,           │
│  psz_new_clone, psz_drop, ...                    │
├─────────────────────────────────────────────────┤
│  mem_*  (raw untyped allocation)                 │
│  mem_new, mem_new_zeroed, mem_resize, mem_drop   │
├─────────────────────────────────────────────────┤
│  C standard library  ← ONLY mem.c may reach here│
│  malloc, calloc, realloc, free                   │
└─────────────────────────────────────────────────┘
```

---

### Banned Routines

| Banned routine | Problem | Replacement |
|----------------|---------|-------------|
| `malloc(n)` | Returns untyped pointer; no length; no ownership signal | `mem_new(n)`, `buf_new(n)`, or `T_new()` |
| `calloc(count, size)` | Same as malloc; overflow check absent in many implementations | `mem_new_zeroed(n)` or `mem_new_array(count, size)` |
| `realloc(p, n)` | Dangerous return: on failure returns NULL but `p` is still valid; `p = realloc(p, n)` leaks on failure | `mem_resize(&p, n)` |
| `free(p)` | No naming discipline; called on wrong pointer type silently | `mem_drop(p)`, `buf_drop(buf)`, `psz_drop(psz)` |
| `strdup(s)` | Returns unownable `char *`; no prefix; allocation invisible at call site | `psz_new_clone(vsz)` |
| `strndup(s, n)` | Same as strdup | `psz_new_clone_n(vsz, n)` |

**Not banned** — these are memory *operations*, not allocation. They work on already-allocated memory and carry no ownership implications:

`memcpy`, `memmove`, `memset`, `memcmp`, `memchr`

---

### `mem_*` — Raw Untyped Allocation

`mem.h` declares the only functions permitted to produce or release raw heap memory. These are used inside `_new` and `_drop` implementations. User code calls the typed APIs instead.

```c
// mem.h

// Allocate size bytes. Uninitialized. Returns NULL on failure.
// size must be > 0; calling with 0 is a programming error (asserts in debug).
void *mem_new(size_t size);

// Allocate size bytes, zero-filled. Returns NULL on failure.
void *mem_new_zeroed(size_t size);

// Allocate count * elemSize bytes, zero-filled.
// Checks for multiplication overflow before allocating.
// Returns NULL on failure or overflow.
void *mem_new_array(size_t count, size_t elemSize);

// Resize the allocation at *p to newSize bytes.
// On success: *p updated to new address, old content preserved up to min(old, new) bytes.
//             Returns 0.
// On failure: *p UNCHANGED, old allocation still valid and must still be freed.
//             Returns -1.
// Caller must handle failure and eventually free *p via mem_drop.
int   mem_resize(void **p, size_t newSize);

// Free a heap allocation. Safe on NULL (no-op).
void  mem_drop(void *p);
```

**Why `mem_resize` takes `void **`**: The standard `realloc` returns the new pointer (which may differ from the old one on a successful resize). Code that writes `p = realloc(p, n)` leaks the original allocation when `realloc` fails and returns NULL. `mem_resize` fixes this by updating `*p` in place only on success, leaving it unchanged on failure — the caller always has a valid pointer to work with.

```c
// The classic realloc bug — do NOT do this:
uint8_t *p = malloc(64);
p = realloc(p, 128);   // BUG: if realloc fails, p is NULL; original 64 bytes leaked

// Correct pattern with mem_resize:
void *p = mem_new(64);
if (mem_resize(&p, 128) != 0) {
    // p still points to the original 64-byte allocation — free it cleanly
    mem_drop(p);
    return ERR_OOM;
}
// p now points to the 128-byte allocation
```

---

### `psz_*` — String Allocation

`psz.h` declares all heap allocation for null-terminated strings. No string is heap-allocated without going through this API.

```c
// psz.h

// Allocate len + 1 bytes (room for len characters plus the null terminator).
// Content is uninitialized (except the null terminator at position len).
// Caller must psz_drop. Returns NULL on failure.
char *psz_new(size_t len);

// Allocate and copy vsz. Equivalent to strdup. Caller must psz_drop.
// Returns NULL on failure.
char *psz_new_clone(const char *vsz);

// Allocate and copy at most maxLen characters from vsz, then null-terminate.
// Equivalent to strndup. Caller must psz_drop. Returns NULL on failure.
char *psz_new_clone_n(const char *vsz, size_t maxLen);

// Allocate and copy bytes from view, appending a null terminator.
// Returns NULL if view contains embedded '\0' bytes (would violate sz contract).
// Returns NULL on allocation failure.
// On success, caller owns the result and must psz_drop.
char *psz_new_from_view(BufView view);

// Free a heap-allocated string. Safe on NULL.
void  psz_drop(char *psz);
```

The distinction between `psz_new_clone` (from another `sz`) and `psz_new_from_view` (from binary data) is intentional. The latter validates that the binary data contains no embedded nulls before promoting it to the `sz` contract. This is the only sanctioned path from binary data to string data.

### String Building and Formatting

`sprintf` and `snprintf` require a pre-allocated buffer of known size, which reintroduces the guessing-game of "how big does the buffer need to be?" The formatting API extends `psz_*` with two functions that allocate the exact required size automatically.

```c
// psz.h additions

// Allocate and format a string, following printf conventions.
// The allocation is exactly the size needed — no truncation, no waste.
// Caller must psz_drop. Returns NULL on format error or allocation failure.
char *psz_new_fmt(const char *fmt, ...);

// va_list form of psz_new_fmt, for use inside other variadic functions.
// Caller must psz_drop. Returns NULL on format error or allocation failure.
char *psz_new_fmt_v(const char *fmt, va_list args);
```

**Implementation — why this is safe in C99:**

The two-pass technique uses `vsnprintf(NULL, 0, fmt, args)`, which C99 defines to return the number of characters that *would* have been written (excluding the null terminator) without writing anything. This gives the exact allocation size needed. `va_copy` (C99) allows the argument list to be consumed twice safely.

```c
// psz.c

char *psz_new_fmt_v(const char *fmt, va_list args) {
    va_list args_copy;
    va_copy(args_copy, args);
    int len = vsnprintf(NULL, 0, fmt, args_copy);
    va_end(args_copy);

    if (len < 0) return NULL;   // malformed format string

    char *result = psz_new((size_t)len);   // allocates len + 1 bytes, sets result[len] = '\0'
    if (!result) return NULL;

    vsnprintf(result, (size_t)len + 1, fmt, args);
    return result;
}

char *psz_new_fmt(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    char *result = psz_new_fmt_v(fmt, args);
    va_end(args);
    return result;
}
```

**MSVC < 2019 caveat:** `vsnprintf` on older MSVC returns `-1` on truncation rather than the required length (MSVC followed the C89 behavior). Use `_vscprintf(fmt, args)` instead of `vsnprintf(NULL, 0, fmt, args)` when targeting MSVC < 2019. This is isolated to `psz.c` and requires no changes to callers.

**Usage:**

```c
// Build a heap string from dynamic values — no manual size calculation
char *pszLabel = psz_new_fmt("item[%zu]: %s (score=%d)", index, vszName, score);
if (!pszLabel) { return ERR_OOM; }

log_info(pszLabel);
psz_drop(pszLabel);


// In a variadic wrapper — forward args via psz_new_fmt_v
char *log_new_entry(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);
    char *pszEntry = psz_new_fmt_v(fmt, args);
    va_end(args);
    return pszEntry;   // caller owns; must psz_drop
}
```

---

### `buf_*` — Binary Buffer Allocation

Specified in full under [Part II — The `buf_*` API](#the-buf_-api). `buf_new`, `buf_drop`, `buf_init`, and `buf_cleanup` are built on `mem_new`, `mem_drop`, and `mem_resize`. They are the typed face of the allocation layer for binary data.

---

### The Forbidden Header: `no_std_alloc.h`

Including this header in a translation unit causes a **compile-time error** if any banned routine is called directly. It redefines each banned name as a macro that expands to an undefined identifier or a `__attribute__((error(...)))` function — whichever produces the clearest diagnostic on the target toolchain.

```c
// no_std_alloc.h
// Include AFTER all standard headers. Bans direct use of standard allocation
// routines. Only mem.c is exempt (do not include this header there).

#ifndef NO_STD_ALLOC_H
#define NO_STD_ALLOC_H

#if defined(__GNUC__) || defined(__clang__)

// GCC and Clang: emit a compile-time error with a descriptive message.
extern void *_banned_malloc(size_t)
    __attribute__((error("malloc is banned. Use mem_new(), buf_new(), or T_new().")));
extern void *_banned_calloc(size_t, size_t)
    __attribute__((error("calloc is banned. Use mem_new_zeroed() or mem_new_array().")));
extern void *_banned_realloc(void *, size_t)
    __attribute__((error("realloc is banned. Use mem_resize(&p, n).")));
extern void  _banned_free(void *)
    __attribute__((error("free is banned. Use mem_drop(), buf_drop(), or psz_drop().")));
extern char *_banned_strdup(const char *)
    __attribute__((error("strdup is banned. Use psz_new_clone().")));
extern char *_banned_strndup(const char *, size_t)
    __attribute__((error("strndup is banned. Use psz_new_clone_n().")));

#define malloc(n)       _banned_malloc(n)
#define calloc(n, s)    _banned_calloc(n, s)
#define realloc(p, n)   _banned_realloc(p, n)
#define free(p)         _banned_free(p)
#define strdup(s)       _banned_strdup(s)
#define strndup(s, n)   _banned_strndup(s, n)

#else

// Portable fallback: expand to an undefined identifier.
// Produces a compilation error (unknown identifier) with a name that points
// to the replacement. Less precise message but zero toolchain dependency.
#define malloc(n)       BANNED_use_mem_new(n)
#define calloc(n, s)    BANNED_use_mem_new_zeroed_or_mem_new_array(n, s)
#define realloc(p, n)   BANNED_use_mem_resize(p, n)
#define free(p)         BANNED_use_mem_drop_or_buf_drop_or_psz_drop(p)
#define strdup(s)       BANNED_use_psz_new_clone(s)
#define strndup(s, n)   BANNED_use_psz_new_clone_n(s, n)

#endif
#endif // NO_STD_ALLOC_H
```

**Include order matters**: standard headers (`<stdlib.h>`, `<string.h>`) must be included *before* `no_std_alloc.h`. The forbidden header shadows the library declarations with macros; the declarations themselves remain in scope for internal use by `mem.c`.

```c
// Correct include order in application code:
#include <stdlib.h>        // stdlib declares malloc, etc. normally
#include <string.h>
#include "mem.h"
#include "psz.h"
#include "buf.h"
#include "no_std_alloc.h"  // now shadow the banned names — must be last standard inclusion
#include "mymodule.h"
```

---

### Build System Enforcement

Manual include-order discipline is fragile. The build system enforces the ban globally without requiring each file to opt in.

**GCC / Clang (Makefile or CMake):**

```makefile
# Force-include no_std_alloc.h into every translation unit except mem.c
CFLAGS += -include no_std_alloc.h

# Exempt mem.c from the ban
mem.o: CFLAGS := $(filter-out -include no_std_alloc.h, $(CFLAGS))
```

```cmake
# CMakeLists.txt
target_compile_options(myproject PRIVATE -include ${CMAKE_SOURCE_DIR}/include/no_std_alloc.h)
set_source_files_properties(src/mem.c PROPERTIES COMPILE_OPTIONS "")
```

`-include file` is equivalent to `#include "file"` at the top of every translation unit. Combined with the GCC/Clang error attribute, violations produce compile-time errors — not link errors, not runtime crashes.

**MSVC:**

```
/FI no_std_alloc.h
```

MSVC does not support `__attribute__((error(...)))`, so the portable fallback branch of `no_std_alloc.h` applies. Violations produce unresolved-identifier errors at compile time.

---

### Where Standard Routines Remain Permitted

Exactly one file in the codebase may include `<stdlib.h>` for allocation purposes without including `no_std_alloc.h`:

**`mem.c`** — the implementation of `mem_new`, `mem_new_zeroed`, `mem_new_array`, `mem_resize`, and `mem_drop`.

All other uses of `<stdlib.h>` are for non-allocation purposes (`EXIT_SUCCESS`, `abs`, `qsort`, `bsearch`, etc.) and are permitted — the forbidden header only redefines the six allocation symbols.

```c
// mem.c — the ONLY file allowed to call malloc/calloc/realloc/free directly
// Do NOT include no_std_alloc.h here.

#include <stdlib.h>
#include "mem.h"

void *mem_new(size_t size) {
    assert(size > 0);
    return malloc(size);
}

void *mem_new_zeroed(size_t size) {
    assert(size > 0);
    return calloc(1, size);
}

void *mem_new_array(size_t count, size_t elemSize) {
    // Check for overflow before calling calloc
    if (elemSize != 0 && count > SIZE_MAX / elemSize) return NULL;
    return calloc(count, elemSize);
}

int mem_resize(void **p, size_t newSize) {
    assert(p != NULL);
    assert(newSize > 0);
    void *next = realloc(*p, newSize);
    if (!next) return -1;
    *p = next;
    return 0;
}

void mem_drop(void *p) {
    free(p);   // free(NULL) is defined to be a no-op
}
```

---

### Debug Extensions (Optional)

In debug builds, `mem.c` can be replaced with an instrumented version that adds:

- **Canary bytes** before and after each allocation, checked on `mem_drop` to detect overwrites
- **Allocation tracking**: a table of `{pointer, size, file, line}` entries to detect leaks at program exit
- **Use-after-free detection**: poison freed memory with a sentinel byte pattern (`0xDD`) and assert on access via guarded pages or shadow memory

These are implementation details of `mem.c` and are invisible to all callers. The API surface is identical in debug and release builds. Switch implementations by swapping `mem.c` at compile time; no application code changes.

```makefile
# Debug build uses instrumented allocator
ifeq ($(BUILD), debug)
    SRC := $(filter-out src/mem.c, $(SRC))
    SRC += src/mem_debug.c
endif
```

---

## Code Review Checklist

A human-readable checklist for reviewers. The audit greps in Part I catch mechanical violations; this checklist catches reasoning errors that grep cannot see.

---

### Per Function

- [ ] **`_new` on every exit path**: Every code path that calls `_new` (or `buf_new`, `psz_new_*`, `mem_new`) eventually calls the matching `_drop`, `buf_drop`, `psz_drop`, or `mem_drop` — including all early-return and error-handling paths.
- [ ] **No conditional allocation**: The function either always allocates or never allocates. It does not return an owned pointer on some paths and a borrowed pointer on others.
- [ ] **`_drop` is not called on NULL without a guard** if the `_drop` implementation does not handle NULL — or confirm the implementation is NULL-safe.
- [ ] **`_move` source is not used after the call**: Any variable passed to a `_move` function is treated as invalid immediately after. It is not read, written, or freed again.
- [ ] **Observer functions free nothing**: Functions named without `_new`, `_drop`, or `_move` do not call `free`, `buf_drop`, `psz_drop`, or `mem_drop` on any of their arguments.
- [ ] **`_startup` / `_shutdown` are paired**: Every `module_startup()` call is matched with a `module_shutdown()` on all exit paths, including failure paths. Shutdown order is the reverse of startup order.
- [ ] **Error paths drop everything already allocated**: If a function allocates multiple resources and the second allocation fails, the first is dropped before returning.

```c
// Example of correct error-path cleanup
Buf *pHeader = buf_new(16);
if (!pHeader) { return ERR_OOM; }

Buf *pPayload = buf_new(256);
if (!pPayload) {
    buf_drop(pHeader);   // ← must drop pHeader before returning
    return ERR_OOM;
}
// ...
buf_drop(pPayload);
buf_drop(pHeader);
```

---

### Per Variable Declaration

- [ ] **`p` prefix → caller owns**: Every variable declared with a `p` prefix (`Buf *pFoo`, `char *pszFoo`, `Token *paryFoo`) was produced by a `_new` function and will be released by a `_drop` function.
- [ ] **`v` prefix → caller borrows**: Every `v`-prefixed variable (`const char *vszFoo`, `BufView vFoo`, `const Token *varyFoo`) was produced by an observer function and is never freed.
- [ ] **No bare `uint8_t *` or `void *`**: All binary data is `Buf *` (owned) or `BufView` (borrowed). A bare byte pointer with a separate length variable is a violation.
- [ ] **`BufView` not used after its source is dropped**: Trace the lifetime of every `BufView`. If the `Buf` or array it views is dropped before the `BufView` goes out of scope, the view is dangling.
- [ ] **`BufView` not used after `buf_resize`**: Any resize of the source `Buf` invalidates all `BufView` values derived from it. Re-derive with `buf_view` after resize.
- [ ] **`ary` variables have a companion `Count`**: Every `T *paryFoo` is accompanied by a `size_t paryFooCount` with matching prefix and base name.

---

### Per File

- [ ] **No banned allocation calls**: `malloc`, `calloc`, `realloc`, `free`, `strdup`, `strndup` do not appear — unless this file is `mem.c`.
- [ ] **`no_std_alloc.h` is force-included or explicitly included**: The build system's `-include` flag covers this file, or the file includes `no_std_alloc.h` explicitly after all standard headers.
- [ ] **Grep `_new` matches `_drop` / `_cleanup`**: Run `grep -n '_new'` on the file. For each result, confirm a matching drop/cleanup exists on every exit path of the owning scope.
- [ ] **Grep `_move` is not followed by source use**: Run `grep -n '_move'` on the file. For each result, confirm the source variable is not dereferenced, assigned, or freed afterward.
- [ ] **No `sz`/`buf` mixing**: No `strlen` or `printf %s` applied to a `Buf`'s `data` field. No `BufView` or `Buf` passed to a function that expects a null-terminated `char *` without going through `psz_new_from_view` first.
- [ ] **Module lifecycle functions called correctly**: If the file calls functions from a module that has `_startup`/`_shutdown`, confirm those are called in the correct order and scope.

---

### At Code-Integration / PR Level

- [ ] **New modules**: Any new `<module>.h` that introduces allocating functions also provides `<module>_drop` and (if applicable) `<module>_cleanup`, `<module>_startup`, `<module>_shutdown`.
- [ ] **Third-party library boundaries**: Any pointer returned by a third-party library that must be freed with the library's own function is **never** freed with `mem_drop`, `buf_drop`, or `psz_drop`. It is wrapped or documented separately.
- [ ] **No `#include <stdlib.h>` outside `mem.c`**: Standard allocation headers are not included in application code, since `no_std_alloc.h` macros would shadow them anyway — but an accidental include before the forbidden header is a latent hazard.

---

## Reference

**[CMemoryProblems.md](CMemoryProblems.md)** — A taxonomy of 34 distinct C memory failure modes across 9 categories. Use this as the authoritative list of what this standard is designed to prevent. Each section of this standard maps directly to one or more problem categories:

| This standard | Prevents (CMemoryProblems.md categories) |
|--------------|------------------------------------------|
| `_new` / `_drop` pairing and audit greps | Category 4: Memory Leaks (#11, #13, #15) |
| Single-owner rule (`p` prefix, no pointer copies) | Category 5: Invalid Frees (#17); Category 6: Use After Free (#22, #23) |
| `v` prefix and lifetime discipline | Category 6: Use After Free (#23, #24, #25) |
| `Buf` / `BufView` mandatory struct (no bare pointer) | Category 7: Buffer Overflows (#26, #28, #29) |
| `sz` / `buf` discipline (no mixing binary and text) | Category 7: Buffer Overflows (#26, #28) |
| `mem_new_array` overflow check | Category 1: Failure to Allocate (#3) |
| `mem_resize(&p, n)` safe realloc pattern | Category 4: Memory Leaks (#12) |
| `buf_new_zeroed` / `mem_new_zeroed` | Category 3: Use Before Initialization (#8, #10) |
| `no_std_alloc.h` banning stdlib allocation | All categories — prevents raw pointer escape |

---

## Glossary

Definitions of terms used throughout this standard. Terms in **bold** within a definition are themselves defined in this glossary.

---

**Borrow / View**
Temporary access to a resource without taking **ownership**. The borrower may read (and with a mutable borrow, write) the resource, but must not free it. The borrow is only valid while the **owner** is alive. In variable naming, expressed with the `v` prefix for pointers or the `BufView` type for binary data.

---

**BufView**
A struct containing a `const uint8_t *data` pointer and a `size_t len`, representing a **borrowed**, non-owning window into a byte sequence. Always passed and stored by value (16 bytes on 64-bit). Never heap-allocated, never freed. Produced by `buf_view`, `buf_slice`, `bufview_of`, or `bufview_of_sz`.

---

**Buf**
A struct containing a `uint8_t *data` pointer and a `size_t len`, where `data` is **heap-allocated** and owned by the struct. The mandatory type for all owned binary buffers. Produced by `buf_new*` (heap) or `buf_init` (stack). Released by `buf_drop` (heap) or `buf_cleanup` (stack sub-resources).

---

**Companion variable**
A count or size variable that must always appear alongside an `ary` typed-array variable because the two cannot be used independently. An `ary` without its `Count` is a convention violation. Example: `Token *paryItems` requires `size_t paryItemsCount`.

---

**Drop**
The act of releasing an **owned** heap resource — freeing its memory and marking it invalid. Corresponds to the `_drop` function verb. After a drop, the variable must not be read, written, or dropped again. Compare with **cleanup**, which releases sub-resources of a stack-allocated struct without freeing the struct itself.

---

**Cleanup**
The act of releasing heap-allocated sub-resources held by a **stack-allocated** struct, without freeing the struct itself (which is on the stack). Corresponds to the `_cleanup` verb. Paired with `_init`. Compare with **drop**.

---

**Heap allocation**
Memory obtained from the runtime memory manager via `mem_new`, `buf_new`, or `psz_new_*`. Persists until explicitly freed with a matching drop function. Always represented in variable names with the `p` prefix.

---

**Length discipline**
The rule that determines how the byte-count of a data sequence is known. Two disciplines exist: **sz** (null-terminated — length is implicit, found by scanning to `\0`) and **Buf** (explicit — length is stored in the `len` field of the struct). These disciplines are mutually exclusive and must never be mixed.

---

**Linear ownership**
A memory model, derived from linear type theory in computer science, in which every heap-allocated resource has **exactly one owner** at any point in time. Ownership may be **transferred** (via `_move`) but not shared. This eliminates double-free and the most common class of use-after-free bugs. The foundational principle of this standard.

---

**Module**
A logical unit of code identified by a shared naming prefix (e.g., `buf_*`, `net_*`, `crypto_*`). Corresponds to a header/source file pair (`buf.h` / `buf.c`). All functions in a module share the prefix. A module may have **module lifecycle** functions (`_startup` / `_shutdown`) if it manages global state.

---

**Module lifecycle**
The pair of one-time global initialization and teardown functions for a module that manages global state. Named `<module>_startup` and `<module>_shutdown`. Called from the main thread before and after all other use of the module. Not the same as **instance lifecycle** (`_new` / `_drop`).

---

**Move**
Transferring **ownership** of a resource from one variable to another. After a move, the source variable is invalid and must not be used. Corresponds to the `_move` function verb. The receiver declares its variable with the `p` prefix.

---

**Observer**
A function that does not produce, consume, or transfer **ownership**. It **borrows** its arguments and returns either a value type or a borrowed pointer. Named without any of the three reserved verbs (`_new`, `_drop`, `_move`). Making no ownership claim is the default — the absence of a reserved verb is itself the signal.

---

**Owner / Ownership**
The scope or variable responsible for eventually freeing a heap-allocated resource. Exactly one owner exists at any time (**linear ownership**). In variable naming, ownership is expressed with the `p` prefix. Ownership is acquired via `_new`, released via `_drop`, and transferred via `_move`.

---

**Reserved verb**
One of the five function-name words that carry defined ownership meaning in this standard: `_new`, `_drop`, `_move`, `_init`, `_cleanup`. Plus the module-lifecycle words `_startup` and `_shutdown`. No other verb carries ownership semantics — all others are implicitly **observer** verbs.

---

**Stack allocation**
Memory allocated in the call stack as a local variable. Automatically released when the enclosing scope exits. No `_drop` is required. If the stack variable is a struct with heap-allocated fields, `_cleanup` is required to release those sub-resources.

---

**Static allocation**
Memory with program lifetime — either a `static` local variable, a global variable, or a module-level singleton allocated in `_startup`. Never freed with `_drop`. The only sanctioned exception to "every `p` must eventually be dropped."

---

**Sub-resource**
A heap-allocated field within a struct. When the containing struct is dropped or cleaned up, its sub-resources must also be freed. `_drop` frees the struct and all its sub-resources. `_cleanup` frees only the sub-resources, leaving the struct itself intact (for stack-allocated structs).

---

**sz** (string, zero-terminated)
A null-terminated byte sequence (`char *`) where length is implicit — found by scanning forward to the `\0` byte. The contract for human-readable text passed to C stdlib string functions. Incompatible with data that may contain embedded null bytes. See **length discipline** and compare with **Buf**.
