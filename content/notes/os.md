+++
title = "OS Notes"
description = "Benjamin Schwartz's OS Notes"
date = "2024-09-09"
aliases = ["os"]
author = "Benjamin Schwartz"
+++

## Table of Contents

- [The Stack & The Heap](#the-stack-&-the-heap)
  - [The Heap](#the-heap)
  - [The Stack](#the-stack)
    - [Stack Overflow](#stack-overflow)
    - [Stack Frames](#stack-frames)
- [Virtual Memory](#virtual-memory)

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

Solves **three problems:**

1. Not enough memory
2. Memory fragmentation
3. Security

<aside>
💡

***Q: What actually is a CPU register?***

</aside>

**Temporary storage** for quick processing. Might hold **instruction, address,** or **data.** 

Composed of **multiple flip-flops** (logic circuits that can be 0 or 1, maintain state until trigger is received)
