# C Memory Ownership Standard

A naming convention and safe allocation standard for C, grounded in **linear type theory**: every heap-allocated resource has exactly one owner at any point in time, and every function name makes that contract unambiguous.

Targets **C99** with support for GCC, Clang, and MSVC 2019+.

---

## The Problem

C has no built-in ownership semantics. Nothing in the language prevents you from freeing a pointer twice, forgetting to free it at all, passing binary data to a string function, or losing a pointer on a failed `realloc`. These are not exotic bugs — they are the routine consequences of convention gaps, and they account for a large share of real-world CVEs.

See [CMemoryProblems.md](CMemoryProblems.md) for a taxonomy of **34 distinct C memory failure modes** across 9 categories that this standard is designed to prevent.

---

## The Standard

[CStandard.md](CStandard.md) defines the full convention in three parts.

### Part I — Function Naming

Three reserved verbs carry ownership meaning. All other verbs are ownership-neutral by default.

| Verb | Meaning | Caller obligation |
|------|---------|-------------------|
| `_new` | Produces an owned resource | Must call `_drop` or `_cleanup` |
| `_drop` | Consumes a heap resource | Resource is invalid after the call |
| `_move` | Transfers ownership | Source is invalid after the call |

```c
Buf *pResult = packet_new_encode(&msg);   // _new → caller owns
send(sock, pResult->data, pResult->len);
buf_drop(pResult);                         // _drop → released
```

Any function not named with one of these three verbs makes no ownership claim — it borrows its arguments and returns a non-owned value.

Two additional verbs cover lifecycle at the stack and module level:

| Verb | Meaning |
|------|---------|
| `_init` / `_cleanup` | Stack-allocated struct initialization and sub-resource release |
| `_startup` / `_shutdown` | Module-level global initialization and teardown |

### Part II — Variable Naming

Variable names encode **ownership** and **length discipline** as a compound prefix, making the memory contract visible at the declaration site.

**Ownership prefix:**

| Prefix | Meaning |
|--------|---------|
| `p` | Heap-allocated, owned — must `_drop` |
| `v` | Borrowed view — must NOT free |
| *(none)* | Stack or static — no heap concern |

**Length discipline:**

| Type | Discipline | When to use |
|------|-----------|-------------|
| `char *pszName` | Null-terminated (`sz`) | Human-readable text for C string APIs |
| `Buf *pData` | Explicit length in struct | Binary data, untrusted input, any data with possible null bytes |
| `BufView vData` | Borrowed binary view | Non-owning window into a `Buf` or raw byte array |
| `T *paryItems` + `size_t paryItemsCount` | Typed array with count | Arrays of structs or typed elements |

`sz` and `Buf` are **mutually exclusive**. Binary data that may contain `\0` bytes is always `Buf` — never `sz`. This is the boundary that prevents most C buffer overflows and truncation bugs.

```c
// Owned heap resources — p prefix
char    *pszUrl    = url_new_build(vszEndpoint, "/api");
Buf     *pPayload  = payload_new_encode(vSecret);

// Borrowed views — v prefix; never freed
const char *vszHost = config_get_host(cfg);
BufView     vSecret = config_get_key(cfg);

// Stack values — no prefix
char szLogMsg[256];
Buf  nonce;
buf_init_zeroed(&nonce, 16);

// ... use ...

buf_cleanup(&nonce);
url_drop(pszUrl);
buf_drop(pPayload);
```

### Part III — Safe Allocation Layer

Direct use of `malloc`, `calloc`, `realloc`, `free`, `strdup`, and `strndup` is **banned** in all application code. A three-layer API replaces them.

```
┌──────────────────────────────────────────┐
│  User code: buf_new, foo_new, psz_new_*  │
├──────────────────────────────────────────┤
│  mem_*: raw untyped allocation           │
├──────────────────────────────────────────┤
│  C stdlib: malloc/calloc/realloc/free    │
│  (mem.c only)                            │
└──────────────────────────────────────────┘
```

The ban is enforced at compile time via `no_std_alloc.h`, which redefines each banned symbol as a macro that triggers a compiler error. The build system force-includes it into every translation unit except `mem.c`.

Key safety improvements over the standard API:

- **`mem_resize(&p, n)`** instead of `realloc(p, n)` — fixes the classic pattern where `p = realloc(p, n)` leaks the original on failure
- **`Buf` struct** instead of `uint8_t * + size_t` — pointer and length are inseparable by type, not by convention
- **`psz_new_fmt(fmt, ...)`** — allocates the exact size needed for a formatted string; no manual size guessing

---

## Audit

Ownership flow across an entire codebase is auditable with three greps:

```sh
grep -r '_new'   # every allocation site
grep -r '_drop'  # every deallocation site
grep -r '_move'  # every ownership transfer
```

An unmatched `_new` with no corresponding `_drop` on every exit path is a leak. This is mechanically verifiable.

---

## Contents

| File | Description |
|------|-------------|
| [guidinglight.md](guidinglight.md) | The philosophy and reasoning behind the standard — start here |
| [CStandard.md](CStandard.md) | The full naming convention and allocation standard |
| [CMemoryProblems.md](CMemoryProblems.md) | Taxonomy of 34 C memory failure modes this standard prevents |

---

## License

This standard is released into the public domain. Use it, adapt it, and enforce it without restriction.
