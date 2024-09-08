+++
title = "C++ Notes"
description = "Benjamin Schwartz's C++ Notes"
date = "2024-09-05"
aliases = ["cpp"]
author = "Benjamin Schwartz"
+++

## Table of Contents

- [Type Deduction](#type-deduction)
  - [Template Type Deduction](#template-type-deduction)
    - [Array Arguments](#array-arguments)
    - [Function Arguments](#function-arguments)
  - [`auto` Type Deduction](#`auto`-type-deduction)
- [Value Categories](#value-categories)
  - [Reference Types](#reference-types)
  - [Forwarding References](#forwarding-references)
  - [`const` Qualifier](#`const`-qualifier)
  - [`constexpr` vs `const`](#`constexpr`-vs-`const`)
  - [`static` Keyword](#`static`-keyword)
  - [Casting](#casting)
  - [`std::move`](#`std::move`)
- [Concurrency](#concurrency)
  - [Threading with `std::thread`](#threading-with-`std::thread`)
    - [When to Use](#when-to-use)
    - [`join()`](#`join()`)
    - [`std::thread` with a Lambda](#`std::thread`-with-a-lambda)
    - [Multiple Threads](#multiple-threads)
    - [Using `std::jthread`](#using-`std::jthread`)
  - [Preventing Data Races and `std::mutex`](#preventing-data-races-and-`std::mutex`)
  - [Preventing Deadlock with `std::lock_guard`](#preventing-deadlock-with-`std::lock_guard`)
  - [Using `std::atomic` to Update a Shared Value](#using-`std::atomic`-to-update-a-shared-value)
- [The Compiler](#the-compiler)
  - [Compilation Unit](#compilation-unit)
- [Memory Management](#memory-management)
  - [Static vs Dynamic memory](#static-vs-dynamic-memory)
  - [Pointers vs References](#pointers-vs-references)
  - [Smart pointers](#smart-pointers)
    - [`std::shared_ptr`](#`std::shared_ptr`)
    - [`std::unique_ptr`](#`std::unique_ptr`)
- [Classes](#classes)
  - [What is the `vtable` ?](#what-is-the-`vtable`-?)
    - [`__vtbl*` pointer](#`__vtbl*`-pointer)

# Type Deduction

**Automatic detection** of the **data type**

## Template Type Deduction

For the following general template form:

```cpp
template<typename T>
void f(ParamType param);
f(expr);               // deduce T and ParamType from expr
```

*Three cases* are possible:

**CASE 1:**

`ParamType` is reference/pointer, but **not** a universal reference

- Reference → ignore reference part, then pattern-match `expr`'s type against `ParamType` to determine `T`
- Note below: “`const`ness” of the object becomes part of type deduced for `T`. If the parameter was `f(const T&)`, we would see that `T` would become `int` for each of the three cases (`const` no longer deduced for `T`).

```cpp
template<typename T>
void f(T& param);

int x = 27;
const int cx = x;
const int& rx = x;

f(x);   // T is int, param's type is int&
f(cx);  // T is const int, param's type is const int&
f(rx);  // T is const int, param's type is const int&
```

- Same example but with a pointer, very similar

```cpp
template<typename T>
void f(T* param);

int x = 27;
const int *px = &x;      // px is a pointer to x as const int 

f(&x);  // T is int, param's type is int*
f(px);  // T is const int, param's type is const int*
```

**CASE 2:**

`ParamType` is a universal reference

<aside>
💡

***Q: What is a [universal reference](https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers)?***

</aside>

There are two types associated with `&&`. 

- One is an rvalue reference.
- The other is a **universal reference**

Rule of thumb: If a variable or parameter is declared to have type `T&&` for some **deduced type `T` ,** then it is a universal reference → it may be either an lvalue ref, or an rvalue ref

- if `expr` is an lvalue, both `T` and `ParamType` are deduced to be lvalue references.
- if `expr` is an rvalue, “normal” rules apply (Case 1)

```cpp
template<typename T>
void f(T&& param);

int x = 27;
const int cx = x;
const int& rx = x;

f(x);       // x is lvalue => T is int&, so is param's type
f(cx);      // x is lvalue => T is const int&, so is param's type
f(rx);      // x is lvalue => T is const int&, so is param's type
f(27);      // 27 is rvalue => T is int, param's type is int&&
```

**CASE 3:**

`ParamType` is neither a pointer nor a reference ⇒ pass by value

- Pretty straightforward: `param` will always be a copy of the object passed in, so any reference or `const`ness is ignored.

A tricky example:

```cpp
template<typename T>
void f(T param);

const char* const ptr = "Hello";   // const ptr to a const object
f(ptr);                            // param's type is const char*
```

- `ptr` is a const pointer (address can’t be changed or set to `nullptr`) pointing to a `const` object `const char*` . As before, since we are passing by value, the `const`ness of the pointer is ignored, and the type deduced for `param` will just be `const char*` .

### Array Arguments

- **array types** are **different** to **pointer types.**
- arrays **decay** into pointers to the first element:

```cpp
const char name[] = "Benji";      // name is type const char[6]
const char *ptrToName = name;     // array decays to pointer

template<typename T>
void f(T param);
f(name);          // T's type will decay to const char * (ptr to first element)

// BUT...
template<typename T>
void f(T& param);
f(name);         // Can pass a *reference* to an array
								 // T's type will be const char(&)[6], with size included!
								 
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept {
	return N;      // Can *catch* the size of array and return it
}
```

NB: the `constexpr` above make the result available during compilation

### Function Arguments

- in a similar way to array arguments, functions can **decay** to **function pointers**

```cpp
template<typename T>
void f1(T param);

template<typename T>
void f2(T& param);

void someFunc(int, double);
f1(someFunc)    // param deduced as ptr-to-func void(*)(int, double)
f2(someFunc)    // param deduced as ref-to-func void(&)(int, double)

```

## `auto` Type Deduction

- is **literally the same** as template type deduction
- `auto` plays the role of `T` and type specifier (e.g. `const`, `&`, `const&`) for variable act as `ParamType`

**CASE 1:** (type specifier is **pointer/reference**, but not universal reference)

**CASE 2:** (type specifier is a **universal reference**)

**CASE 1:** (type specifier is **neither pointer nor reference**)

```cpp
auto x = 27;        // case 3
const auto cx = x;  // case 3
const auto& rx = x; // case 1

auto&& uref1 = x;   // x is int and lvalue, uref1's type is int&
auto&& uref2 = cx;  // x is const int and lvalue, uref2's type is const int&
auto&& uref3 = 27;  // 27 is int and rvalue, uref3's type is int&&

const char name[] = "Benji";    // name's type is const char[6]
auto arr1 = name;   // arr1's type is const char*
auto& arr2 = name;  // arr2's type is const char(&)[6]

void someFunc(int, double);
auto func1 = someFunc;  // func1 is void (*)(int, double)
auto& func2 = someFunc; // func2 is void (&)(int, double)
```

<aside>
💡

***Q: How does `auto` differ from template type deduction?***

</aside>

- Is to do with initialiser lists. `auto` *assumes* that a braced initialiser represents a `std::initializer_list` but template type deduction doesn’t.
- Common mistake: accidentally declaring a `std::initializer_list` variable.

```cpp
auto x1 = 27;        // type is int, value is 27
auto x2(27);         // same

auto x3 = { 27 };    // type is std::initializer_list<int>, value is { 27 }
auto x4{ 27 };       // same

template<typename T>
void f(std::initializer_list<T> initList);
f({1, 2, 3});        // T deduced as int, initList as std::initializer_list<int>
```

# Value Categories

<aside>
💡

***Q: What is an rvalue reference?***

</aside>

- Similar to lvalue refs, except they **can only be initalised with rvalues**. E.g. `int&& a = 50` since 50 is an `int` and a temporary object.
- Usually, **rvalue == temporary object** (not always)

<aside>
💡

***Q: What are [value categories](https://en.cppreference.com/w/cpp/language/value_category)?***

</aside>

[Copper Spice video](https://www.youtube.com/watch?v=wkWtRDrjEH4)

An *expression* has two factors: **data type** and **value category**. May produce a result, and generate a side-effect.

Main questions to ask: **Does expression have a name? Does it have a memory location? Can you take the address of the expression?**

- lvalue: has name, must be able to take address using & operator

```cpp
Widget *button = new Widget;
```

- Here, `button` is an lvalue, it is “pointer to Widget”, has a name, and can take its address.
- Also, `*button` is an lvalue, doesn’t have a name, can take the address of `*button` .

So, both the pointer *and the thing it points to in memory* are both lvalues.

NB: just because something is `const` doesn’t change its value category.

- rvalue: temporary value, doesn’t have a name, can’t take the address of it

```cpp
int someVar = 35; // someVar is lvalue, 35 is rvalue
35 = someVar;   // Not legal, since 35 is an rvalue and located on the left side
```

## Reference Types

<aside>
💡

***Q: What are references?***

</aside>

There are three types: **lvalue ref, const ref, rvalue ref**.

Pass by value: called function gets a *copy* of passed data.

- A function that passes by value can be given *either an lvalue or an rvalue**.***

```cpp
// PASS BY VALUE
class Widget{};
void func(Widget pb);  // pb is passed by value
Widget x;              // x is lvalue
func(x);               // call with lvalue valid
func(Widget{});        // call with rvalue valid

// PASS BY LVALUE REF
void func(Widget& pb);  // pb is passed by reference
func(x);               // call with lvalue valid
func(Widget{});        // call with rvalue ERROR 
// Caller would not be able to see changes made to temporary Widget

// PASS BY CONST REF
void func(const Widget& pb);  // pb is passed by reference
func(x);               // call with lvalue valid
func(Widget{});        // call with rvalue valid
// Both are valid because func CAN'T modify pb
// There are no changes that we have to worry about the visibility of

// PASS BY RVALUE REF
void func(Widget&& pb);  // pb is passed by rvalue reference
func(x);               // call with lvalue ERROR
func(Widget{});        // call with rvalue valid 
// Passing lvalue is not valid, because func() might change x
// Caller has promised not to look at value of x after func()
// Rvalue ref is fine because any changes made wouldn't be seen by caller
```

 Summary:

lvalue reference: Called method can modify data, and caller **will** observe changes made

const reference: Called method can **not** modify data

rvalue reference: Called method can modify data, and caller **will not** observe changes made

- Does **not** indicate **value category** but indicates **data type ⇒ does not** indicate how you can **use** the parameter
- rvalue reference data type declared using `&&` . **Can be on left side** of expression.
- Can **bind** an rvalue (category) to an rvalue reference, prolonging lifetime **as if it were an lvalue**

<aside>
💡

**KEY:** 

- A **data type** is a value, and the operations associated with it (such as int, pointer, string, hash, lvalue ref, rvalue ref)
- An **expression** is a **data type** (as above) AND a **value category** (lvalue, rvalue)
- Confusingly, an rvalue/lvalue reference is a **data type**
</aside>

Rvalue reference example:

```cpp
int&& func() { return 42; }         // Returns an rvalue reference to int

int main() {
	int&& foo = func();  // call to func() has value category rvalue
											 // rvalue is bound to foo, foo is declared rvalue reference
											 // foo has value category lvalue (has a name)
											 // foo has data type: rvalue reference, value category: lvalue
	int var = foo + 3;   // valid: foo is lvalue
	foo = 47;            // valid: foo is lvalue, data type is rvalue ref so we can assign to it
}

int& func() { return 42; }    // WON'T compile, 42 is rvalue
															// rvalue cannot bind to an lvalue reference
```

## Forwarding References

```cpp
Widget&& var1 = Widget{};
auto&& var2 = var1;
```

- `var1` is rvalue reference, so lifetime of temporary `Widget` is extended for as long as `var1` is in scope.
- `var2` uses `auto` which deduces type. A **deduced type** with `&&` becomes a **forwarding reference**. Will really be an lvalue reference like: `Widget& var2 = var1`.

Another example:

```cpp
// foo is an lvalue (has a name)
const Widget *foo;

// pass either lvalue/rvalue data type (since const ref)
// must pass data type of Widget             
void someMethod(const Widget &);

someMethod(*foo);    // call method with *foo
```

## `const` Qualifier

“Who is not allowed to change what”.

NB: *Read it backwards.* Clockwise/Spiral rule.

```cpp
const int var;          // const variable: var cannot be changed
const Widget& var;      // const reference
char *const var;        // const pointer
const char* var;        // pointer to const
void someMethod() const;         // someMethod() will not change this
void someMethod(const Widget x); // someMethod() will not change x 
```

## `constexpr` vs `const`

- **`const`** means “promise not to change”
- **`constepxr`** means “known at compile time”

## `static` Keyword

- Different meanings with different types
- **static variables**
    - in a **function**: space is allocated **only once** for the **lifetime** of program. Value in previous call gets carried through the next function call. Below example will print `0 1 2 3 4 5`
    
    ```cpp
    void demo() {
    	static int count = 0;
    	cout << count << " ";
    	++count;
    }
    int main() {
    	for (int i = 0; i < 5; ++i) {
    		demo();
    	}
    	return 0;
    }
    ```
    
    - in a **class**: initialised only **once** and are in a **separate static storage**. Are **shared across objects of the class**. Reference it using class name and `::` .
    
    ```cpp
    class A {
    public: 
    	static int i;
    }
    
    A::i = 1;
    
    int main() {
    	A obj;
    	cout << obj.i;
    }
    ```
    
- **static members of class**
    - **static class objects:** Objects when declared static have **lifetime of the program.**
    - **static functions in a class**: like static variables, don’t depend on object of the class. Invoke using the `.` operator
    
    ```cpp
    class A {
    public:
    	static void printMsg() { "Hello!\n" };
    }
    
    int main() {
    	A::printMsg();
    }
    
    ```
    

<aside>
💡

***Q: What does `static` mean when used as a global variable?***

</aside>

- If several classes were to include the same header file `Settings.h` , and this file contained the global variable `static bool boolVariable`
    - if we try to modify this variable from a class then **each compilation unit gets its own copy**. `static` makes the variable **local to the compilation unit**
    - We could fix this problem by using `extern` ⇒ tells compiler that variable exists in another compilation unit, and will be resolved by the linker

## Casting

From most safe to least safe:

- `static_cast` always defined, known at compile time
- `dynamic_cast` always defined, might fail at runtime
    - cast from base class to child class. Might get `nullptr` at runtime
- `const_cast` only used to remove const
- `reinterpret_cast` forcing compiler ⇒ “shut up”. Dangerous, use sparingly
- `(int)`C-style cast

## `std::move`

[Mike Shah](https://www.youtube.com/watch?v=2gUqyt5JTtM)

- allows for **efficient transfer of resources.** Transfer ownership **without copying.** We “adopt” or “steal” the value.
- in a way “moving the lifetime of an object”
- is literally just a **cast** to an **rvalue reference:** `static_cast<std::string&&>()`

# Concurrency

[Mike Shah Concurrency Series](https://www.youtube.com/playlist?list=PLvv0ScY6vfd_ocTP2ZLicgqKnvq50OCXM)

- **More throughput** since more tasks happening in parallel = better performance
- **Concurrency:** multiple things can happen at once, but **order matters,** might have to **wait on shared resources** (One coffee machine example, Orchestra example, Dinner conversation)
    - CS Example: memory allocator, File I/O, Network requests (awaiting data)
    - Single CPU core has **time-share system**, might be multiple cores.
    - **Multiple cores** solve the problem of too much heat being generated as transistors are packed closer and closer.
- **Parallelism:** Everything happens at once, instantaneously (Two coffee machines)

## Threading with `std::thread`

- **Thread =** “lightweight process”
    - 1 Process (application) can have many threads.
    - Thread execution starts with **main thread**
    - Has own **stack for local variables**
        - Data registers, stack pointers, program counter
    - But shares **code, data and kernel context**
        - Shared libraries, run-time heap, r/w data, read-only code/data
        - kernel context: VM structures, descriptor table, brk pointer
- Allows us to execute **multiple control flows** at the same time

### When to Use

- Heavy computation
    - e.g. GPU for graphics, resolve complex computation
- Separate work
    - Gives performance and **simplifies logic** of problem

### `join()`

**Spawning thread** of a new thread (i.e. parent) can wait on child thread with `.join()`.

Otherwise, the **program will be destroyed** before `myThread` has finished (when `main` returns).

Enforces **order of execution. Blocks** the main thread.

```cpp
void test(int x) { 
	std::cout << "Hello from thread!" << std::endl
};
int main() {
	std::thread myThread(&test, 100);
	myThread.join();
	std::cout << "hello from main thread" << std::endl;
}
```

- At some point, both threads are executing concurrently (maybe one same or different cores)

### `std::thread` with a Lambda

```cpp
int main() {
	auto lambda = [](int x) {
		std::cout << "Hello from thread!" << std::endl
	}
	std::thread myThread(lambda, 100);
	myThread.join();
	std::cout << "hello from main thread" << std::endl;
}
```

### Multiple Threads

```cpp
int main() {
	auto lambda = [](int x) {
		std::cout << "Hello from thread! " << std::this_thread::get_id() << std::endl
		std::cout << "Argument passed in: " << x << std::endl;
	}
	
	// Spawn 10 threads
	std::vector<std::thread> threads;
	for (int i = 0; i < 10; ++i) {
		threads.push_back(std::thread(lambda, i));
	}
	
	// Join 10 threads
	for (int i = 0; i < 10; ++i) {
		threads[i].join();
	}
	std::cout << "hello from main thread" << std::endl;
}
```

- **thread id** is a unique identifier for each thread
- `this_thread` is a namespace for currently executing thread
- printing this out will probably be quite jumbled

### Using `std::jthread`

- is a `std::thread` with support for auto-joining and cancellation
    - uses concept of **RAII**, when destructor is called, calls `request_stop()` and then `join()`
- is in C++20, avoids joining errors

## Preventing Data Races and `std::mutex`

- It’s possible that a **globally accessible variable** across threads is **updated** **simultaneously**
    - is a **data race**, gives non-deterministic results
- `std::mutex` - Mutual Exclusion
    - like having “one key” that only one person can use at once ⇒ reading/writing a variable
- `lock` locks the mutex, blocks if `mutex` not available. `unlock` unlocks the mutex (duh)
    - locks protect a **critical region** (e.g. the shared value). Is **atomic section of code** (smallest unit that can be accessed).
    - `try_unlock` returns if fails to lock

## Preventing Deadlock with `std::lock_guard`

- **Deadlock** happens when a `mutex` lock never gets returned - other threads are blocked.
- `lock_guard` is a `mutex` **wrapper** that uses RAII-style mechanism (when destructor called, any `mutex`'s held will be released, even when there is an **exception as below**)
    - `lock_guard` ”holds” the `mutex`

```cpp
std::mutex gLock;
static int shared_value = 0;

void shared_value_increment() {
	std::lock_guard<std::mutex> lockGuard(gLock);
		try {
			++shared_value;
			throw "aborting...";
		} catch(...) {
			std::cout << "handle exception";
			return;
		}
}

// Use jthread to auto-join
int main() {
	std::vector<std::jthread> jthreads;
	for (int i = 0; i < 1000; ++i) {
		jthreads.push_back(std::jthread(shared_value_increment));
	}
}
```

- `lock_guard` is better than manually handling locks and unlocks, especially in the case of excpetions. Enforces RAII principles

## Using `std::atomic` to Update a Shared Value

- removes the need of `lock_guard` and `mutex`

```cpp
static std::atomic<int> shared_value = 0;
void shared_value_increment() {
	++shared_value;
}

int main() {
	std::vector<std::thread> threads;
	for (int i = 0; i < 1000; ++i) {
		threads.push_back(std::thread(shared_value_increment));
	}
	for (int i = 0; i < 1000; ++i) {
		threads[i].join();
	}
}
```

- **ONLY** `operator++` , `operator--` , etc. are supported for modifying atomic values
    - Using `shared_value = shared_value + 1` is **not atomic**
- Implemented using a **hardware mechanism**
- can create **custom atomics**, but it has to be a trivially copyable type (i.e. types that are copybable by copying bits in memory or with `memcpy()` ). Types with a **virtual function** or **not stored in contiguous memory** could not be made atomic. Reason: **memory alignment**. [This article](https://ryonaldteofilo.medium.com/atomics-in-c-what-is-a-std-atomic-and-what-can-be-made-atomic-part-1-a8923de1384d) has more
    - Can check `is_lock_free()` to see if implemented with `mutex` or similar

# The Compiler

## Compilation Unit

***Q: What is a compilation unit exactly?***

1. **`#include`** is handled by pre-processor ⇒ all these `#include` directives are replaced by the content of the header files, as well as any `#define`, `#ifdef`, etc. 
2. Therefore, **header file has no separate life**, compiler has no knowledge of them
3. **compilation unit** is therefore the source file **after** the pre-processor has done its job.

# Memory Management

## Static vs Dynamic memory

- **statically allocated memory**: allocated at compile time

```cpp
int main() {
	// Statically allocated memory at compile time
	int i;
	int a[10];
	
	// Dynamically allocated memory at run time
	int* iptr = new int;
	int* aptr = new int[10];
	delete iptr;
	delete aptr;	
}
```

- **dynamically allocated memory**: don’t know ahead of time how much memory program will need. Use pointers for this and `new` keyword
- `delete` frees the space of the object being pointed at, allowing it to be overwritten

## Pointers vs References

<aside>
💡

***Q: What is the difference between pointers and references?***

</aside>

- **Both** are implemented by storing the **address** of an object

```cpp
// Pointer:
int a = 10;
int *p = &a;
// OK
int *p;
p = &a;
// OK
int b = 6;
p = &b;

// References:
int &p = a;
// NOT CORRECT (declare and initialise in two steps)
int &p;
p = a;
// NOT CORRECT (reassign)
int &p = a;
int &p = b;
```

- **Reference** can be thought of as a **`const` pointer** with automatic indirection (the `*` is automatically applied for you).
    - Must be **declared and initialized** in the **same step.**
    - A pointer can be **re-assigned** but a reference **cannot**
- **Pointer** takes up size on the stack, and has **own memory address**. **Reference** **shares the same memory address** with the original variable, and takes up **no space** on the stack

|  | References | Pointers |
| --- | --- | --- |
| Reassignment | Cannot be reassigned | Can be reassigned |
| Memory Address | Shares same address as original variable | Has own memory address |
| Function | Is referring to another variable | Is storing the address of the variable |
| Null Value | Doesn’t have null value | Can be `nullptr` |
- **Performance** is the **same** (both pointers under the hood)
- **Use references:** function params, return types
- **Use pointers:** `nullptr` needed, pointer arithmetic, data structures like LL, Tree, etc.

## Smart pointers

[Code, Tech, and Tutorials](https://www.youtube.com/watch?v=u_FEZDfBPk8)

[C++ From Scratch: std::shared_ptr](https://www.youtube.com/watch?v=4bdp9aHzuQY)

- `#include <memory>`

### `std::shared_ptr`

- Share access to data, manage memory for you, doesn’t use `new` or `delete`
- same functionality as pointer, also has `unique()` and `use_count()`  ⇒ how many others are holding a reference, when this is 0 it calls `delete`

```cpp
std::shared_ptr<float> stuff = std::make_shared<float>(20.f);
```

- `stuff.reset()` releases the memory, can set it to a new pointer
- As with RAII, when a `shared_ptr` goes out of scope it decrements the `use_count` and/or frees the associated memory
- Size is about **double** of a normal pointer

### `std::unique_ptr`

- More **isolated control**: only allows **one reference at a time** to control memory, when goes out of scope it calls`delete` on associated memory
- has only `get()` and a few other functions (no `use_count()` etc.)

```cpp
std::unique_ptr<float> stuff = std::make_unique<float>();
*stuff = 10.f;
std::unique_ptr<float> other_stuff = std::make_unique<float>(*stuff.release());
```

- `release()` returns a pointer to the type, and calls `delete` on the old memory

```cpp
void another_scope(std::unique_ptr<float>& uptr) {
	std::unique_ptr<float> pointertime = std::move(uptr);
}
```

- take a reference to `uptr` and we can use `std::move` to **destroy the old `unique_ptr`** (but not the data) and assign the new `pointertime` to the **same data**
- **address** will stay the same. The **data stays in the same place.**

# Classes

## What is the `vtable` ?

- **“virtual table” ⇒** supports **dynamic dispatch**
- **virtual** keyword creates a **virtual table**

<aside>
💡

**KEY:**

- Functions are defined is contained **in the class** itself (e.g. `B::bar()`)
- **Every class** has its own **`vtable`**
- Each **object** contains a `__vtbl*` pointer to the `vtable` for that class
- Each **entry in the `vtbl`** points to the **correct** function definition
- It goes: Object’s `__vtbl*` pointer ⇒ Class’ `vtbl` ⇒ Class’ function definition
    - The **method name** is used as an **index** into the `vtbl`
</aside>

### `__vtbl*` pointer

- **base class** contains a **`__vtbl*`** pointer, to a table which has the virtual function definition
- **derived class** would also contain **its own `__vtbl*` pointer**
- **Each entry** in the “**virtual table**” points to the **correct instance of the function** (e.g. back to the derived class function).
- **overhead:** 4B or 8B pointer for `__vtbl*`

```cpp
class Base {
public:
	Base(){};
	virtual ~Base() {};
	virutal void MemberFunc() {
		std::cout << "Base::MemberFunc()\n";
	};

class Derived : public Base {
public:
	Derived(){};
	~Derived(){};
	void MemberFunc() override {
		std::cout << "Derived::MemberFunc()\n";
	}
};

int main(){
	// Create a Base* that can point to 'Base' or 'Derived' (anything that 'is-a' Base).
	// Also a vtbl is created for this instance of the object
	Base* instance = new Derived;
	
	// Vtbl points us to the correct member function
	instance->MemberFunc();
	delete instance;
	
	
	instance = new Base;
	base->MemberFunc();
	delete instance;
	return 0;
}
```

<aside>
💡

***Q: Why do we make the Base destructor virtual?***

</aside>

- ensures that **when derived classes go out of scope** they are **deletion** of **each class in heirarchy** happens ****in **correct order** ⇒ avoids memory leaks
- Use when you might **delete an instance** of a **derived** through a **pointer to base class**
- Destruction happens from **bottom→up** (i.e. derived destructor called, then base). (The inverse is true when we call constructor, it goes **base then derived**).

```cpp
Base *b = new Derived();
delete b;

// OUTPUT:
Base Constructor Called
Derived Constructor called
Derived destructor called
Base destructor called
```
