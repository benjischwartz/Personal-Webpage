+++
title = "C++ Notes"
description = "Benjamin Schwartz's C++ Notes"
date = "2024-09-05"
aliases = ["cpp"]
author = "Benjamin Schwartz"
+++

## Table of Contents

- [Effective Modern C++](#effective-modern-c++)
- [Type Deduction](#type-deduction)
  - [Template Type Deduction](#template-type-deduction)
    - [Array Arguments](#array-arguments)
    - [Function Arguments](#function-arguments)
  - [`auto` Type Deduction](#`auto`-type-deduction)
- [Value Categories](#value-categories)
  - [Reference types](#reference-types)
  - [Forwarding references](#forwarding-references)
  - [Const qualifier](#const-qualifier)
  - [Constexpr vs const](#constexpr-vs-const)
  - [Casting](#casting)

# Type Deduction

## Template Type deduction

For the following general template form:

```cpp
template<typename T>
void f(ParamType param);
f(expr);               // deduce T and ParamType from expr
```

Three cases are possible:

1. `ParamType` is reference/pointer, but **not** a universal reference
    1. Reference → ignore reference part, then pattern-match `expr`'s type against `ParamType` to determine `T`
2. `ParamType` is a universal reference
3. `ParamType` is neither a pointer nor a reference

<aside>
💡

***Q: What is a [universal reference](https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers)?***

</aside>

There are two types associated with `&&`. 

- One is an rvalue reference.

## Value Categories

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

### Reference types

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

### Forwarding references

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

### Const qualifier

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

### Constexpr vs const

- **`const`** means “promise not to change”
- **`constepxr`** means “known at compile time”

### Casting

From most safe to least safe:

- `static_cast` always defined, known at compile time
- `dynamic_cast` always defined, might fail at runtime
    - cast from base class to child class. Might get `nullptr` at runtime
- `const_cast` only used to remove const
- `reinterpret_cast` forcing compiler ⇒ “shut up”. Dangerous, use sparingly
- `(int)`C-style cast
