# Guiding Light: The Philosophy Behind This Standard

This document explains **why** this standard exists and **how** its rules were shaped. It is not a reference — [CStandard.md](CStandard.md) is the reference. This is the reasoning behind the decisions so that when you encounter an edge case, you can derive the right answer from first principles rather than looking for a rule that does not exist yet.

Read this before reading the standard. It will make everything else make more sense.

---

## Background: C's Memory Problem Is Not Accidental

C was designed in 1972 by Dennis Ritchie at Bell Labs for writing Unix. It was designed for small programs, small teams, and hardware where every byte mattered. Manual memory management was a deliberate choice: the programmer is in control, and that control is the point.

Fifty years later, C is still essential. It runs operating system kernels, embedded firmware, network stacks, game engines, cryptographic libraries, and real-time control systems. It runs on 8-bit microcontrollers with 2 KB of RAM and on supercomputers with petabytes of storage. Nothing else has its combination of performance, portability, and hardware access.

The problem is that the programming model has not changed, but everything around it has. Programs are millions of lines long. Teams are dozens of engineers. A single function is read, modified, and debugged by people who did not write it. Manual memory management at this scale is not just difficult — it is statistically guaranteed to produce bugs.

### The Numbers Are Not Small

In 2019, Microsoft disclosed that approximately **70% of CVEs in their products over the prior decade were memory safety violations** — use-after-free, buffer overflow, double-free, and their variants. Google reported the same percentage for high-severity bugs in Chrome. The NSA and CISA have both published guidance recommending organizations move to memory-safe languages where possible.

These are not exotic, hard-to-find bugs in obscure codebases. They are the routine output of C development at scale, in some of the most well-resourced software organizations in the world, written by talented engineers following the best practices of their time.

### Why C Still Matters Despite This

Moving to a memory-safe language is the right call for many projects. For many others, it is not possible:

- **Embedded and real-time systems** require deterministic behavior and no garbage collector pauses
- **Existing codebases** of tens of millions of lines cannot be rewritten
- **Hardware interfaces** require precise memory layout and direct register access
- **Safety-critical domains** (automotive, aerospace, medical) have qualification requirements for specific compilers and languages
- **Interoperability** — nearly every language has a C FFI; C is the lingua franca of system interfaces

C is not going away. The question is not "should we use C?" but "how do we use C without producing the same class of bugs that have caused decades of security vulnerabilities?"

### What C Is Missing

The language gives you memory allocation and deallocation. It gives you pointers. It does not give you:

1. **Ownership tracking** — the compiler does not know who is responsible for freeing any given pointer
2. **Bounds checking** — reading or writing past the end of an array is undefined behavior, not an error
3. **Length coupling** — there is no built-in connection between a pointer and how many bytes it covers
4. **Allocator discipline** — `malloc`, `calloc`, `new`, `HeapAlloc`, and `mmap` all produce memory that cannot be freed with each other's deallocators
5. **Lifetime enforcement** — a pointer to freed memory is indistinguishable from a valid pointer

A convention cannot fix all of these. But it can make the most dangerous ones **visible**, **auditable**, and **automatically enforced** — which is what this standard is designed to do.

### What This Standard Does Not Do

Be honest about the limits:

- It does not make C memory-safe in the theoretical sense. A determined developer can still cause bugs.
- It does not replace tools. Use Valgrind, AddressSanitizer, and static analyzers in addition to following this standard, not instead of.
- It does not prevent all classes of memory bug — only the patterned, nameable ones that come from unclear ownership, mixed length disciplines, and unguarded allocation.

What it does: makes ownership **unambiguous by name**, makes violations **detectable by grep and compiler**, and makes the most common allocation mistakes **structurally impossible**.

---

## The Seven Principles

### 1. This Standard Is for Teams

A naming convention followed by one person on a solo project helps no one. The value of this standard is entirely in its **consistency across a codebase and across a team**. A function written by one developer must be instantly readable by another developer who has never seen it, without needing to inspect the implementation.

This shapes every decision:

- Rules are stated as absolutes, not guidelines. "Always use `_new`" not "consider using `_new`."
- The vocabulary is small enough to memorize. Five reserved verbs. Two prefix characters. Three struct types. A new developer should internalize the full system in a day.
- The rules apply uniformly. There are no "senior developer exceptions" or "performance-critical carve-outs." Inconsistency is the enemy of readability.
- Code review becomes a checklist, not an expertise contest. A junior developer can verify ownership contracts without deep knowledge of the codebase.

**The test**: if a developer who has been away from this codebase for six months can read a new function and immediately know who owns what — without reading the implementation — the standard is working.

---

### 2. One Way to Do It — No Options, No Debates

Every time a developer faces a decision that the standard should answer, there must be exactly **one right answer**. Not two acceptable answers. Not "it depends." One.

When there are multiple acceptable ways to express the same thing, teams fragment. Half the team writes `pBuffer`, half writes `pbuf`. Code review devolves into style debates. New developers don't know which convention to follow so they invent a third. The codebase becomes a patchwork of dialects that nobody fully understands.

This is why the design process for this standard explicitly narrowed options at each step:

| Choice | Options considered | Decision |
|--------|-------------------|----------|
| Verb for producing a resource | `_create`, `_new`, `_make`, `_alloc` | `_new` — one word |
| Verb for releasing a resource | `_free`, `_destroy`, `_release`, `_drop` | `_drop` — avoids stdlib collision |
| Binary buffer with length | Pointer + companion variable, or struct | Mandatory `Buf` struct — structurally enforced |
| Raw allocation function | `malloc` with conventions, or new API | `mem_new` — banned the alternative |

When you find yourself wanting to introduce an alternative to something the standard already answers, that is a signal to re-read the reasoning behind the existing rule — not to create an exception.

**Anti-pattern to avoid:**
```c
// Three developers, three conventions — all "acceptable"
Token  *create_token(void);      // developer A's style
Token  *token_alloc(void);       // developer B's style
Token  *TokenNew(void);          // developer C's style
```

**Correct:**
```c
// One convention, always
Token *token_new(void);
```

---

### 3. Enforce by Automation First, Convention Second

Human discipline degrades under pressure. Deadlines, context-switching, and complexity all erode adherence to manual conventions. A rule that can only be enforced by humans reviewing code will be violated — not because developers are careless, but because humans are fallible under load.

The design principle: **if a rule can be enforced by the compiler, the build system, or a script, enforce it there**. Human code review is the last line of defense, not the first.

Examples of this principle in the standard:

| Rule | How it is enforced |
|------|--------------------|
| Do not call `malloc` directly | `no_std_alloc.h` makes it a compile error |
| Binary buffers always carry their length | `Buf` struct — structurally impossible to separate them |
| String formatting allocates exact size | `psz_new_fmt` — no manual size calculation possible |
| Allocation sites are auditable | Three grep commands cover all ownership flow |
| Debug builds detect leaks and overflows | `mem_debug.c` swap — zero application code change |

**The test for a new rule**: before adding a rule that developers must follow manually, ask "can the compiler, build system, or a static tool enforce this?" If yes, enforce it there. If no, make the rule as grep-auditable as possible.

**Anti-pattern to avoid:**
```c
// Convention-only: "remember to always free this"
// Nothing stops a developer from forgetting
void *alloc_thing(size_t n);
void  free_thing(void *p);
```

**Correct — structurally enforced:**
```c
// The Buf struct makes it impossible to have data without length
// no_std_alloc.h makes it impossible to call malloc accidentally
Buf *buf_new(size_t len);
void buf_drop(Buf *buf);
```

---

### 4. Keep It Simple — Complexity Is the Enemy of Adoption

A standard that is too complex to follow is worse than no standard. A partially-followed standard creates a codebase that mixes idioms, confusing everyone. Simplicity is not a nice-to-have — it is the precondition for the standard working at all.

Every time complexity was added to the standard, the question asked was: **does this complexity prevent a real bug, or does it just satisfy a theoretical completeness concern?** Theoretical completeness that makes the standard harder to follow was cut.

Examples of deliberate simplification:

- **Three reserved verbs, not ten.** Some edge cases are slightly awkward to express with three verbs. That awkwardness is a feature — it forces API designers to clarify their intent.
- **Two prefix characters, not eight.** The ownership prefix is `p` or `v`. There is no `h` for handle, no `lp` for long pointer, no `rg` for range. Those distinctions were real in 16-bit Windows programming; they add complexity without corresponding safety benefit today.
- **`Buf` has `len`, not `len` and `cap`.** A growable buffer with capacity tracking is a common pattern, but it is also a separate concern from binary data safety. The standard covers binary data safety. Capacity tracking belongs in your specific data structure, not the base convention.

**Anti-pattern to avoid:**
```c
// Too many options — which should I use?
char  *make_string(size_t n);
char  *alloc_string(size_t n);
char  *string_create(size_t n);
char  *new_string(size_t n);
```

**Correct — one obvious choice:**
```c
char *psz_new(size_t len);
```

**When simplicity conflicts with completeness**: choose simplicity. A simple rule followed consistently is more valuable than a complete rule followed inconsistently.

---

### 5. Rules Must Be Logical, Not Arbitrary

Arbitrary rules get ignored, rationalized away, or "extended" by developers who think they understand the spirit better than the letter. Rules that follow from first principles are self-reinforcing: once you understand the principle, you can derive the rule yourself, apply it to new situations, and identify violations without memorizing a list.

Every rule in this standard traces back to one of two first principles:

**Principle A — Linear ownership**: every resource has exactly one owner. This is not a style preference; it is a theorem. If two variables claim ownership of the same memory, one of them is wrong, and the program will eventually crash or corrupt.

**Principle B — Length discipline**: a pointer without a length is incomplete information. Operating on incomplete information produces wrong behavior. The form of the incompleteness (missing terminator vs. missing length) determines the class of bug.

Trace any rule in the standard back to one of these principles and you can verify that it is necessary, not arbitrary:

| Rule | Traces to |
|------|-----------|
| `_new` must pair with `_drop` | Principle A — one owner, one lifetime |
| No conditional allocation | Principle A — caller cannot know if they own the result |
| `Buf` is a mandatory struct | Principle B — pointer and length are inseparable facts |
| `sz` and `Buf` are mutually exclusive | Principle B — their length contracts contradict each other |
| `mem_resize` takes `void **` | Principle A — the pointer must update atomically with the allocation |

**When a rule feels arbitrary**: trace it back. If you cannot connect it to Principle A or Principle B, it may need to be re-examined.

---

### 6. Examples Always — Good Patterns and Anti-Patterns

Abstract rules are hard to apply. A developer who has seen a rule violated — and understands why that violation is dangerous — will never make the same mistake. A developer who has only read an abstract rule will.

Every rule in the standard has:
- A concrete "do this" example showing correct usage in context
- A concrete "not this" (anti-pattern) example showing the bug the rule prevents

The anti-pattern examples are the more important of the two. They show the failure mode, not just the convention. A developer who sees the `p = realloc(p, n)` pattern and understands that it leaks the original allocation on failure will remember that rule permanently. The same developer who only reads "`mem_resize` takes a pointer-to-pointer" may forget.

**Template for any new rule:**

```
Rule: [one sentence statement of the rule]

Good:
[code showing correct pattern]

Anti-pattern:
[code showing the violation]
// Comment explaining the specific bug this causes
```

Both examples must compile (or nearly compile) and be realistic. Toy examples that no one would actually write are not useful. The anti-pattern should be something a competent developer could plausibly write under normal circumstances.

---

### 7. Ground Every Rule in a Real Problem

Every rule in this standard exists because of a class of real bugs that real software has suffered. A rule without a traceable problem is a style preference, not a safety measure — and style preferences do not justify the cognitive overhead of a standard.

The traceability is documented in [CMemoryProblems.md](CMemoryProblems.md), which catalogs 34 distinct C memory failure modes. Every significant rule in [CStandard.md](CStandard.md) maps to one or more entries in that catalog.

Before adding a new rule to the standard, identify which problem in the catalog it prevents. If it does not prevent a cataloged problem, either add the problem to the catalog (with a real-world description) or do not add the rule.

**Examples of problem-to-rule traceability:**

| Real problem | Rule it motivated |
|-------------|------------------|
| `p = realloc(p, n)` leaks original on failure (#12) | `mem_resize(&p, n)` pattern |
| `strlen` on binary data stops at first `0x00` (#26, #28) | `sz` and `Buf` are mutually exclusive |
| Double-free from two variables claiming ownership (#17) | Single-owner rule; `p` and `v` prefix distinction |
| Missing `free` on early return (#13) | `_new`/`_drop` pairing audit |
| `malloc(count * sizeof(T))` integer overflow (#3) | `mem_new_array(count, elemSize)` with overflow check |
| `malloc` return unchecked, then dereferenced (#1) | `mem_new` returns NULL; `_new` functions propagate NULL |

---

## How to Apply These Principles to New Situations

When you encounter a situation the standard does not explicitly address, apply the principles in order:

1. **Does linear ownership (Principle A) give you the answer?** Determine who owns the resource. Use `_new`/`_drop`/`_move` accordingly.

2. **Does length discipline (Principle B) give you the answer?** Determine if the data has embedded nulls or gets its length from a source other than scanning. Use `Buf`/`BufView` accordingly.

3. **Is there one obvious right answer, or multiple acceptable ones?** If multiple, pick the simplest and document why.

4. **Can the compiler or a script enforce the choice?** If yes, do that first.

5. **Write the example before writing the rule.** If the correct usage is not clear enough to illustrate with a ten-line code snippet, the rule is not clear enough to add to the standard.

---

## What This Standard Is Not

- **It is not a comprehensive C style guide.** Brace placement, indentation, and comment style are out of scope. Use a separate linter for those.
- **It is not a replacement for tools.** Run AddressSanitizer, Valgrind, and a static analyzer. This standard and those tools are complementary.
- **It is not optional on a codebase that has adopted it.** Partial adoption is worse than no adoption. Inconsistency is the enemy of readability.
- **It is not finished.** C codebases encounter new patterns. When the standard does not cover a situation, apply the seven principles, document the decision, and add it to the standard.

---

## Further Reading

- [CStandard.md](CStandard.md) — The full naming convention and safe allocation standard
- [CMemoryProblems.md](CMemoryProblems.md) — The 34 C memory failure modes this standard addresses
