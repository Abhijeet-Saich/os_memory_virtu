# OSTEP Chapter 14 — Interlude: Memory API

**The question this chapter answers:** when a C program needs memory, where does it come from, and who cleans it up?

Concepts are in the order they build on each other. Each one ends with the key takeaway.

---

## Part 1 — Two kinds of memory

### 1. Stack memory (automatic)

Every function call gets space for its local variables. The compiler sets it aside when the function starts and takes it back when the function returns.

```c
void func() {
    int x;   // appears when func() is called
}            // gone when func() returns
```

- Called a *stack* because calls nest: the last function called is the first to finish, so the newest space is always removed first.
- You never ask for it or give it back. That's why it's also called **automatic memory**.

**Takeaway:** a local variable lives exactly as long as its function call. After the return, its space is reused by the next call — so anything that must outlive the function can't live here.

### 2. Addresses and pointers

Memory is a long row of boxes, one byte each. Each box has a number: its **address**.

A variable has two numbers attached to it: its **value** and its **address**.

A **pointer** is a variable whose value is an address.

```
address:   1000          2000
          +-----+       +------+
   x      |  5  |    p  | 1000 |     p points to x
          +-----+       +------+
```

```c
int *p;   // "p is a pointer to an int"
```

- The pointer is an ordinary variable with its own address (2000 here).
- JS comparison: `const b = a;` with an object — both refer to the same object. A C pointer is that idea with the address visible as a number.

**Takeaway:** if a function's local pointer dies, only the pointer disappears. The data it pointed to is untouched.

### 3. Heap memory (explicit)

A second region where data stays alive until **you** say you're done. Function calls returning have no effect on it.

```c
int *x = malloc(sizeof(int));
```

This one line uses both kinds of memory:

```
      STACK                    HEAP
   address 2000              address 5000
    +------+                  +-----+
 x  | 5000 |  ------------>   |  ?  |
    +------+                  +-----+
```

- `malloc(...)` reserves room on the heap and returns its address.
- `int *x` is a local variable on the stack that stores that address.

To hand heap data back to a caller, return the **value** of the pointer (the address). The local variable dies; the copied address and the heap data survive.

JS also has a heap, but a garbage collector frees it for you. C has none.

**Takeaway:** pointer on the stack, data on the heap. You allocate it, you free it.

---

## Part 2 — The API

### 4. `malloc()`

```c
#include <stdlib.h>
void *malloc(size_t size);
```

- Input: a number of **bytes**. `malloc` knows nothing about ints or lists.
- Output: the address where your block starts, or `NULL` if that request failed (usually not enough memory).
- `NULL` is a special pointer value (in practice 0) meaning "points to nothing". Using it crashes — check it before trusting the result.
- `#include <stdlib.h>` lets the compiler check you're calling `malloc` correctly.

**Casting** (`(int *) malloc(...)`): `malloc` returns `void *`, a plain address with no type. A cast only reassures the reader; it's not needed for correctness.

### 5. `sizeof`

Hardcoding sizes (`malloc(40)`) breaks across machines, because type sizes vary.

```c
int *x = malloc(10 * sizeof(int));   // count × size of one
```

- `sizeof` is resolved at **compile time**: the compiler replaces it with a plain number before the program runs. That's why it's an operator, not a function.
- It only knows **types**, nothing that happened at run time.

| Code | `sizeof(x)` gives | Why |
|---|---|---|
| `int *x = malloc(10 * sizeof(int));` | 8 (on 64-bit) | `x`'s type is "pointer" |
| `int x[10];` | 40 | the type itself says "10 ints" |

**Takeaway:** you can't ask a pointer how big its block is. If you need that size later, store it yourself.

### 6. `free()`

```c
int *x = malloc(10 * sizeof(int));
/* ... use it ... */
free(x);
```

- Input: exactly the address `malloc` returned. Returns nothing.
- **`free` does not change your pointer.** `x` still holds the old address afterwards.
- How does it know the size? The **allocation library** (the code behind `malloc`/`free`) records each block's size at `malloc` time and looks it up at `free` time. The compiler can't know it, since sizes can be decided at run time.

| Who knows | What |
|---|---|
| Compiler | type sizes (`sizeof`) |
| Allocation library | block sizes, at run time |

**Takeaway:** every `malloc` should be matched by exactly one `free`.

---

## Part 3 — The seven common errors

All of these **compile without warnings**. "It compiled" or "it ran" ≠ "it is correct".

### Background: dereferencing

```c
*x = 5;   // go to the address in x, and put 5 there
```

- `x` = the address. `*x` = the thing at that address.
- `x[i]` reaches item `i` of a block. Room for 10 means valid indexes 0–9.

### Background: strings in C

There is no string type. A string is chars in a row, ending with a zero byte `\0`.

```
 +---+---+---+---+---+----+
 | h | e | l | l | o | \0 |    5 chars + 1 end marker = 6 bytes
 +---+---+---+---+---+----+
```

- Nothing stores the length; code finds the end by walking to `\0`.
- `strlen(s)` counts chars, **not** the `\0` → 5 for "hello".
- `strcpy(dst, src)` copies **including** the `\0` → 6 bytes.
- Rule: `malloc(strlen(s) + 1)`. Never `sizeof` on a `char *` (gives the pointer size).

### Error 1 — Forgetting to allocate

```c
int *x;    // room for an address, not for an int
*x = 5;    // writes to a garbage address
```

- `x` holds leftover bytes = a random address. Writing there usually triggers a **segmentation fault** (segfault): the OS stops the program for touching memory it isn't allowed to.
- Fix: put an address from `malloc` into `x` first.
- **The garbage is in the pointer itself.**

**Takeaway:** declaring a pointer and allocating memory are two separate steps.

### Error 2 — Not allocating enough (buffer overflow)

```c
int *x = malloc(10 * sizeof(int));
x[10] = 7;    // one past the end — C doesn't check
```

String version: `malloc(strlen(src))` then `strcpy` → the `\0` lands one byte past the end.

Possible outcomes (you don't control which):
- nothing visible (unused bytes, or the library gave a bit extra)
- silent damage to other data, showing up later elsewhere — also how many security attacks work
- a crash

**Takeaway:** a program can run fine 100 times and still be incorrect. Correctness is about what the code does, not what you observed.

### Error 3 — Forgetting to initialize (uninitialized read)

```c
int *x = malloc(sizeof(int));
printf("%d\n", *x);   // read before any write
```

- `malloc` reserves, it doesn't clean. The block holds leftovers.
- C has no `undefined` like JS — just garbage interpreted as a value.
- Often zero the first time, random later.
- **The pointer is good; the garbage is in the block.**

**Takeaway:** write before you read.

### Error 4 — Forgetting to free (memory leak)

- A block never freed stays reserved for the program's whole life.
- Worst case: the only pointer was a local variable that's gone — now it can never be freed.
- **Short-lived programs:** the OS reclaims everything at exit, so little harm (still free as a habit).
- **Long-running programs** (servers, databases): small leaks add up until memory runs out.
- **Garbage collectors don't prevent this.** If something still references unused data, the collector must keep it.

Example: a Node server pushing every request into a global array and never removing them → the array is reachable, so every request is kept → memory grows forever.

### Error 5 — Freeing too early (dangling pointer)

```c
int *x = malloc(sizeof(int));
free(x);       // block no longer yours
*x = 99;       // x still holds the address — C allows it
```

Trace (block at 5000, reused by a later `malloc`):

| Line | `x` | `y` | Box 5000 |
|---|---|---|---|
| `x = malloc` | 5000 | | garbage |
| `free(x)` | 5000 | | available |
| `y = malloc` | 5000 | 5000 | garbage |
| `*y = 10` | 5000 | 5000 | 10 |
| `*x = 99` | 5000 | 5000 | **99** |

`y`'s value changes, but the bug is in `x`'s code — possibly far away. Very hard to find.

**Defensive habit:** set the pointer to `NULL` after freeing. Then a mistaken use crashes immediately, at the real mistake, and `*y` stays 10.

**Takeaway:** "belongs to" is a rule in your head, not something C enforces. Two pointers with the same number refer to the same memory.

### Error 6 — Freeing twice (double free)

```c
free(x);
free(x);
```

- The library's records can become inconsistent. Result is **undefined behaviour**: C promises nothing (crash, nothing, or corruption later).
- `free(x)` is really `free(5000)` — it frees whatever is at that address. If that block was reused by `y`, the second free silently makes `y` dangling.
- Usually caused by two parts of a program each thinking they own a block.
- `free(NULL)` does nothing, so the `NULL` habit makes a repeat free harmless.

Same idea with copies:
```c
int *q = p;
free(q);
*p = 5;    // p is dangling too — same address
```

### Error 7 — Freeing the wrong address (invalid free)

```c
free(x + 3);   // middle of a block
free(&n);      // stack variable; & = "address of"
```

- The library looks the address up in its records, finds nothing, and doesn't know the size → undefined behaviour.
- Stack memory is never yours to free.

### The three rules behind all seven

1. Only use memory you've allocated, within its bounds — errors 1, 2, 3
2. Free every block exactly once — errors 4, 6
3. Only free what `malloc` gave you, and don't use it afterwards — errors 5, 7

**Core idea:** pointers are numbers. `malloc`, `free`, and `*` all act on the address. Variable names and ownership exist only in the programmer's head.

---

## Part 4 — Underneath `malloc` (OS support)

### 9. Library calls vs system calls

Two levels of memory management:

| Level | Who | How you reach it |
|---|---|---|
| 1 | The OS — owns all real memory | **system call** (relatively slow) |
| 2 | The allocation library, inside your program | **library call** (`malloc`, `free`) |

The library is a wholesaler:
1. Asks the OS for a big chunk (system call).
2. Splits it into the small blocks you `malloc`.
3. `free` usually returns a block to the library's pool, not the OS.
4. Only goes back to the OS when the pool runs out.

So most `malloc` calls never touch the OS. And at exit the OS takes back the whole region regardless — that's why leaks vanish when a short program ends.

**Takeaway:** after `free`, OS-reported memory usually doesn't drop, because the block went back to the library's pool.

### 10. `brk` / `sbrk` — the end of the heap

```
 start of heap                      break
      |                               |
      +-------------------------------+ - - - - - - -
      |  usable                       |  not yours
      +-------------------------------+ - - - - - - -
```

- The **break** is the end of the heap. Past it → segfault.
- `brk(addr)` moves the break to an address; `sbrk(n)` moves it by `n` bytes.
- Outward = heap grows. Inward = memory goes back to the OS.
- **Never call these yourself** — the library assumes it alone controls the break.
- Vocabulary: *break* ≠ *breakpoint* (a debugger pause point).

Limitation: the heap can only shrink from the end. Blocks A, B, C in order with C still in use → freeing A and B can't return them to the OS. This is one reason long-running programs often keep high memory use.

### 11. `mmap` — memory outside the heap

```
   +--------+                 +-----------+
   |  heap  |   ...gap...     |  mmap'd   |
   +--------+                 |  region   |
            ^ break           +-----------+
```

- Asks the OS for a separate region, independent of the break.
- Can be returned to the OS on its own anytime (`munmap`).
- **Anonymous** region = not tied to a file; backed by **swap space** (disk used as memory overflow). Covered later in virtual memory.
- Real allocators: small requests from the heap, very large ones often via `mmap` — so freeing a large block can actually lower OS-reported memory.

---

## Part 5 — Other calls

### 12. `calloc` — allocate and zero

```c
int *x = calloc(10, sizeof(int));   // count, size of one → all zeros
```

- Targets error 3: reading before writing gives 0, not garbage.
- Zero isn't always the right start value — you still set other values yourself.
- Slightly more work; plain `malloc` is fine if you'll overwrite everything.

| Read without writing | Result |
|---|---|
| `calloc` | 0, guaranteed |
| `malloc` | leftover value, not guaranteed (may even look like 0) |

### 13. `realloc` — resize a block

```c
int *bigger = realloc(x, 20 * sizeof(int));   // old address, new total size
```

General case:
1. Allocate a new larger block elsewhere
2. Copy the old contents in
3. Free the old block
4. Return the new address

After a move, the old pointer `x` is wrong in two ways:
- **using** it → dangling pointer (error 5); may still "work" because freed bytes aren't erased until reused
- **freeing** it → double free (error 6)

Only use the address `realloc` returned.

Cost: growing copies every byte. Growing a big block one item at a time copies it over and over (homework Q8 measures this).

---

## Tools (for the homework)

- **gdb** — debugger; pause and inspect a running program. Compile with `-g` for variable names.
- **valgrind** (memcheck, `--leak-check=yes`) — watches every memory access; catches bugs that "seem to work".
- **purify** — older commercial equivalent.

## Further reading (from the chapter)

- *The C Programming Language* — Kernighan & Ritchie
- *Advanced Programming in the UNIX Environment* — Stevens & Rago, Ch. 7
- *Exterminator* — Novark et al., PLDI 2007 (automatic detection/correction of memory errors)