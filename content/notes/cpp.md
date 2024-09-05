+++
title = "C++ Notes"
description = "Benjamin Schwartz's C++ Notes"
date = "2024-09-05"
aliases = ["cpp"]
author = "Benjamin Schwartz"
+++

# Effective Modern C++

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
