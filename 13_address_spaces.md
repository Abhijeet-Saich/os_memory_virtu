# Address Spaces: Revision Notes

*OSTEP, Chapter 13: The Abstraction: Address Spaces*

**The chapter in one sentence:** the OS gives every process its own private, pretend view of memory (an *address space*) and secretly translates each address the process uses into a real location in RAM.

---

## 1. The story: each problem and its fix

| # | Situation | Problem | Fix |
|---|-----------|---------|-----|
| 1 | Early machines: one program, real addresses | A buggy write can land in the OS. Nobody cared, since only one user was affected | None needed yet |
| 2 | Machines are expensive and the CPU idles during I/O | Wasted hardware | **Multiprogramming**: keep several programs ready, switch when one waits on I/O |
| 3 | Many users want quick responses | Batch jobs make people wait | **Time sharing**: switch rapidly between programs |
| 4 | Switching by saving all of memory to disk | Far too slow, because disks are slow and memory is big | Keep all processes in memory at once; a switch then only saves the CPU registers |
| 5 | Many processes living side by side in RAM | One process can read or corrupt another's memory | **Protection**, delivered through virtual memory |
| 6 | Each process is loaded at a different physical place | Programs should not have to care where they landed | **Address space**: every process sees memory starting at 0 |

---

## 2. The address space

The address space is **memory as the running program sees it**. It has three main regions:

| Region | Holds | Changes size? |
|--------|-------|---------------|
| **Code** | The program's instructions | No |
| **Heap** | Memory requested while running (`malloc` in C, objects and arrays in JS) | Yes |
| **Stack** | Function-call bookkeeping: local variables, arguments, return addresses | Yes |

The book's 16KB example:

```
 0KB  +----------------+
      |      Code      |
 1KB  +----------------+
      |      Heap      |
 2KB  +-------|--------+
      |       v        |   heap grows toward higher addresses
      |                |
      |     (free)     |
      |                |
      |       ^        |   stack grows toward lower addresses
15KB  +-------|--------+
      |     Stack      |
16KB  +----------------+
```

**Why heap and stack sit at opposite ends:** both need room to grow. Placed next to each other they would block one another while free space sat unused elsewhere. At opposite ends they share the one free middle.

> The book draws address 0 at the top of the page. So its "top" means address 0, and its "heap grows downward" means toward larger addresses.

---

## 3. Virtual vs. physical addresses

| | Virtual address | Physical address |
|---|---|---|
| What it is | A position inside a process's own address space | A real location in RAM |
| Who uses it | The program | Only the OS and hardware |

- Every memory access is **translated** from virtual to physical by the OS and hardware together.
- The same virtual address in two processes leads to two different physical places.
- **Any address a program can print is virtual.**

**Example.** Process A is loaded at physical 320KB and Process B at 192KB. Both read virtual address 1KB:

- A really touches physical **321KB**
- B really touches physical **193KB**

The simplest mental model is `physical = where the process starts + virtual address`. The real mechanism comes in later chapters.

---

## 4. The three goals of virtual memory

| Goal | Meaning |
|------|---------|
| **Transparency** | The program cannot tell its memory is virtualized. Here the word means "invisible", not "open". |
| **Efficiency** | Translation happens on every access, so it must be fast and use little extra memory. This is why hardware helps. |
| **Protection** | A process cannot read or write anything outside its own address space. |

Protection gives **isolation**: one process can crash, or be malicious, without harming other processes or the OS.

---

## 5. What I saw on my own Mac

| Observation | What it means |
|-------------|---------------|
| `sysctl hw.memsize` gave 17,179,869,184 bytes | 16 GB of physical RAM |
| `top` showed 16G used, 405M unused | Normal. The OS fills spare RAM with a **file cache** and frees it on demand. "Unused" is not the same as "available". |
| `top` showed `VM: 46T vsize` on a 16GB machine | Address spaces are **sparse**: huge ranges of promised addresses, mostly never touched. Only the used parts need physical memory. |
| Reading `argv[1]` with no argument gave a segmentation fault | The program read virtual address 0 (`NULL`). No mapping exists there, so the hardware trapped and the OS killed the process. Protection in action. |

---

## 6. Interview quick-fire

**What is an address space?**
The memory a process sees: its code, heap and stack, addressed from 0 upward, private to that process.

**Virtual vs. physical address?**
Virtual is what the program uses. Physical is the real RAM location. The OS and hardware translate one into the other on every access.

**Why virtualize memory at all?**
Protection between processes, a simple and uniform view for programmers, and efficient sharing of one physical memory among many processes.

**Why not save the whole memory to disk on every process switch?**
Too slow. Keeping processes in memory means a switch only saves registers.

**How can total virtual memory exceed physical RAM?**
Virtual address ranges cost nothing until used. Only the touched parts are backed by physical memory.

**What is a segmentation fault?**
An access to an address the process has no right to use. The hardware detects it during translation and the OS ends the process.

**Stack vs. heap?**
The stack is automatic and tied to function calls: memory appears on call and vanishes on return. The heap is requested explicitly and lives until it is released.

**Why does a healthy system show very little free memory?**
The OS uses idle RAM as cache. Empty RAM is wasted RAM.

---

## 7. C concepts met so far

**Compile, then run.** C source is translated into a separate runnable file first.

```
cc memory-user.c -o memory-user     # compile; -o names the output
./memory-user 100                   # run; ./ means "in this folder"
```

**`#include`** brings in descriptions of functions so you can call them.

| Header | Stands for | Gives you |
|--------|------------|-----------|
| `stdio.h` | standard input/output | `printf` |
| `stdlib.h` | standard library | `atoi`, `malloc` |

**`main` and command-line arguments.** Execution starts in `main(int argc, char *argv[])`.

| For `./memory-user 100` | Value |
|---|---|
| `argc` | `2` (count of words, program name included) |
| `argv[0]` | `./memory-user` |
| `argv[1]` | `100`, as text |

Always check `argc` before reading `argv[1]`. If the argument is missing, `argv[1]` is `NULL`.

**Text is not a number.** `atoi` converts a string of digits to an `int`. It returns 0 when it cannot find a number.

**`printf` placeholders**

| Placeholder | Prints |
|---|---|
| `%d` | an `int` |
| `%s` | a string |
| `%p` | an address |
| `\n` | a line break (`printf` does not add one for you) |

**Pointers: a value vs. where it lives**

| Syntax | Meaning |
|---|---|
| `x` | the value |
| `&x` | the address of `x` |
| `int *` | type: "address of an int" |
| `char *` | type: "address of a character", which is how C holds a string |
| `NULL` | the "points at nothing" address, 0 |

A pointer is a virtual address stored in a variable. This is where C meets this chapter.

**Reading compiler messages**

- Format: `file:line:column: message`, with `^` marking the exact spot.
- An **error** means nothing was built. A **warning** means it was built but is probably wrong. Fix both.
- The compiler's suggested fix repairs the types. It does not know what you meant.
- When unsure what a variable holds, print it.

**`man <name>`** opens the built-in manual, for example `man atoi`. Press `q` to quit.

---

## 8. Still to do

- [ ] Add the `argc` check to `memory-user.c`
- [ ] Allocate the requested megabytes with `malloc`
- [ ] Loop forever, touching every entry
- [ ] Watch memory with `top` while it runs
- [ ] Inspect the process's address space with `vmmap`