+++
title = "OS Notes"
description = "Benjamin Schwartz's OS Notes"
date = "2024-09-09"
aliases = ["os"]
author = "Benjamin Schwartz"
weight = 2
+++

## Table of Contents
- [The Stack & The Heap](#the-stack-&-the-heap)
  - [The Heap](#the-heap)
  - [The Stack](#the-stack)
    - [Stack Overflow](#stack-overflow)
    - [Stack Frames](#stack-frames)
- [Virtual Memory](#virtual-memory)
  - [Not Enough Memory](#not-enough-memory)
  - [Memory Fragmentation](#memory-fragmentation)
  - [Security](#security)
  - [Physical Memory](#physical-memory)
  - [Implementation](#implementation)
    - [Page Table Mapping](#page-table-mapping)
    - [Translation Lookaside Buffer (TLB)](#translation-lookaside-buffer-(tlb))
    - [Memory Management Unit (MMU)](#memory-management-unit-(mmu))
    - [Multi-level Page Tables](#multi-level-page-tables)

# The Stack & The Heap

A **program** uses memory that is divided into **five main segments:**

1. **code** segment (compiled code, is read-only)
2. **bss** segment (zero-initialised global and static variables)
3. **data** segment (initialised global and static variables)
4. **heap** (dynamically allocated variables)
5. **stack** (local variables, stack frames incl. parameters, etc.)

## The Heap

A.k.a **free store**. 

`new` operator returns a pointer to some region of the heap. `delete` returns the memory to the heap. Handled by **allocator**

- Used for **large chunks of memory** (arrays, structs/classes), as well as data with **unkown size** at compile time, or **size that might change**
- **slow** to allocate, **slow** to dereference pointer (compared to directly accessing a variable)
- can **leak memory** if not careful ⇒ stays allocated until specifically deallocated, or **program ends** (and OS does cleanup)

## The Stack

A.k.a **call stack**. 

Tracks **active functions** (haven’t returned yet). Allocates functions parameters & local variables.

Program begins by **pushing main()** onto call stack. 

- is **fixed-size** chunk of memory
- items being pushed/popped are **stack frames**
- **stack pointer** (contained in “marker” register) tells us where we are in the stack ⇒ no cleanup required

Stack can **grow up** or **grow down** depending on architecture

Has a **limited size**. Can be 1MB or 8MB (g++/clang for Unix)

### Stack Overflow

Means **all stack memory** has been allocated ⇒ overflows into other sections of memory

Can happen as a result of **allocating too many variables**, or **recursive function calls**. 

### Stack Frames

Keeps track of **all data** associated with a **function call.** Consists of:

1. **return address** (instruction beyond function call)
2. **function arguments**
3. **local variables**
4. saved **copies of registers** that need to be restored

When this is **popped** off the stack:

1. **Registers restored** from call stack
2. **stack frame popped** off top
3. **return value** handled and CPU resumes exection at this address

<aside>
💡

***Q: Why is stack faster than heap?***

</aside>

- **Pushing** **to stack** is faster (allocator doesn’t have to search for new place to put data, always at the top of the stack)
- **Allocating on heap** is more work (must find big enough space, perform bookkeping to prepare for next allocation)
- **Accessing data on heap** is **slower** (must follow pointer). On stack, data is close to other data so processor can do its job faster

<aside>
💡

***Q: How do threads interract with stack/heap?***

</aside>

Threads share **all segments except the stack**. (Data, Bss, Code, Heap)

Threads have **independent call stacks**, but memory in other threads’ stacks is **still accessible**

# Virtual Memory

[Tech With Nikola](https://www.youtube.com/watch?v=A9WLYbE0p-I)

***Solution:** Give **each program** its **own memory space.*** We need a mapping from “virtual” address space to “physical” address space.

## Not Enough Memory

- In **old** days, CPU’s could support up to 4GiB of RAM (32-bit registers ⇒ $2^{32} \approx 4$ GiB)
- Modern CPU’s support **64-bit** address space (16 million TiB theoretical limit)

<aside>
💡

***Q: What actually is a CPU register?***

</aside>

**Temporary storage** for quick processing. Might hold **instruction, address,** or **data.** 

Composed of **multiple flip-flops** (logic circuits that can be 0 or 1, maintain state until trigger is received)

**Solution:**

- When a program **tries to access more data than can fit on ram**, virtual memory allows the **disk** to be used ⇒ **“SWAP Memory”**
- OS automatically moves **oldest data** to disk, and then moves the **accessed data** to RAM, and updates the mapping ⇒ “**Page Fault”** (not ideal, quite slow. Therefore, more RAM is better)

## Memory Fragmentation

- Memory gets **split** in such a way that **we can’t allocate memory to programs** (even if we had enough)
- **virtual memory** provides a layer of **indirection** which allows us to split a program’s memory across separate areas in RAM

**Solution:**

- From programs POV, nothing has changed ⇒ address space is contiguous

## Security

- Two programs trying to access the same memory, **corrupting** each others **data**

**Solution:**

- Both programs have their **own virtual memory address space**, each mapped to **different physical locations**.
- Sometimes, programs may want to **share data**. Fix: have parts of program **map to the same physical space** (e.g. reading the **same shared library**, read the same memory in RAM)

## Physical Memory

- Basically **refers to RAM** (but RAM makes up only **part** of the physical address space)
- BUT… RAM is **not the only physical device** that CPU can access
    - **MMIO/PMIO** Devices are mapped to physical memory
- Part of **RAM** is **reserved for OS** at boot, the rest is used by programs

## Implementation

- Mapping from virtual to physical address is done with a **page table.**
    - **EACH PROGRAM** has its own **Page table**
- **Physical and virtual** memory is divided into **4KiB** chunks called **pages** ⇒ map pages rather than addresses
- **Page table** contains **Page table entries (PTE’s).**
    - 1 PTE per 4KiB of memory ⇒ about 1 million PTE’s ⇒ 4MiB per Page Table (each entry 4B)
- Must move **4KiB** of data, but nearby addresses are often accessed at the same time

### Page Table Mapping

- **offset** is calculated by subtracting the **start address of virtual page**
    - same offset is used to calculate the physical address. **Last 12 bits are the same**
    - remaining **20 bits** are retrieved from the **page table** (Virtual Pg # ⇒ Physical Pg #)
- **Page Fault!** ⇒ page table mapping says memory on disk, CPU raises exception
    - OS evicts LRU memory from RAM. If **dirty**, OS writes it back to disk (meaning program has **written** something to it **after loading it from disk**), otherwise no need to write
    - OS then loads correct one from disk
    - Is **very slow**, OS will probably switch to another program in the meantime
        - **DMA** can do this loading while CPU is doing something else

<aside>
💡

***Q: Where is the page table located in memory?***

</aside>

- Any process has a memory map divided into **user space** and **kernel space.** RAM itself can be logically divided into kernel/user space.
- Page table is in **kernel space.** Physically, it is in RAM and is the **same space for all processes**. Note: **user space** is **unique** to each process.

### Translation Lookaside Buffer (TLB)

- very **small** and **fast cache** ⇒ allows us to find physical address very quickly
- Hit time < 1 cycle, typically 4096 entries. Miss penalty: 10 to 100 cycles
- Modern CPU’s have **two TLB’s ⇒** 1 for **instructions,** 1 for **data**. Typically 2 levels of TLB caching
- if TLB **full**, **remove LRU entry** (and load new entry)

### Memory Management Unit (MMU)

- Responsible for **address translation** and **generating page faults**
- Usually on CPU board and programmed by OS

### Multi-level Page Tables

- 2 or more levels of page tables. **Technique to save space**
- Reduces memory wastage for storing page tables ⇒ **only the outermost table** will reside in **main memory**. Other page tables will be brought to the main memory **as needed** (can be swapped to disk).
- Virtual address **gets divided** into sections, and each section indexes into a different page table. The result is then added together with the offset
- **linux** uses **5-Level Page tables**
