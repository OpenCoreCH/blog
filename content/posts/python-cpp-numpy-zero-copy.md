+++
title = "Exposing C/C++ Data as a Python NumPy Array"
date = "2021-12-26"
author = "Roman Böhringer"
authorTwitter = "romanboehr"
cover = ""
tags = ["cpp", "python"]
keywords = ["numpy", "np"]
description = "I recently needed to use memory that was allocated inside a C++ library in a Python application which expects a NumPy array without performing any copies. With ctypes, this can be implemented quite easily."
showFullContent = false
+++

I recently needed to use memory that was allocated inside a C++ library in a Python application which expects a NumPy array without performing any copies. With `ctypes`, this can be implemented quite easily.
Let's say we have a shared library `libcpp.so` where a function `get_shared_memory` returns the pointer to an array of doubles that is stored on the heap:

{{< code language="cpp" >}}
double* get_shared_memory(std::size_t num) {
    auto p = new double[num];
    ...
    return p;
}
{{< /code >}}

The function can be called from Python with `ctypes`:

{{< code language="python" >}}
num_elem = 42

libcpp = ctypes.CDLL("libcpp.so")
libcpp.get_shared_memory.restype = ctypes.c_void_p 
pointer = libcpp.get_shared_memory(num_elem)
{{< /code >}}

Afterwards, `pointer` can be interpreted from Python as a pointer to a double:

```python
double_p = ctypes.cast(pointer, ctypes.POINTER(ctypes.c_double))
```

At this point, `double_p` is a object of type `LP_c_double`. It can already be used to set and get values of the C++ array, for instance we can do:

```python
double_p[0] = 1.0
print(double_p[2])
```

But we cannot use a `LP_c_double` as a normal Python lists. For instance, there is no way for Python to know the length of the underlying buffer and calling `len(double_p)` will fail.
However, we can also use `pointer` to create a NumPy array on top of the existing memory thanks to `np.ctypeslib.as_array`:

```python
np_buff = np.ctypeslib.as_array((ctypes.c_double * num_elem).from_address(pointer))
```

`np_buff` can now be used as a normal NumPy array. When you are done using it, you need to `free` the allocated memory in C++ again, as you will leak memory otherwise. The pointer that is returned in `get_shared_memory` should therefore be stored somewhere.