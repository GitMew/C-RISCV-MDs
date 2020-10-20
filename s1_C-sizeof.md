# CASS Notes --- C Programming --- Ex. 2

## On `sizeof()`

Four observations can be made w.r.t. C's in-built function `sizeof()`:
1. `sizeof()` is merely a compile-time trick. 
    You cannot dynamically calculate `sizeof()` inside a procedure for a passed array, since the compiler can't know the size of the passed array beforehand.
    
2. `sizeof()` only uses the type of its argument.
    Combined with the previous property, this means a procedure would default to the size of an array pointer (32 bits in this case).
    
3. If the compiler is able to statically determine the argument of `sizeof()`, it will specify the argument's subtype as much as possible.
    In the case of an array of size 8 being initialised in the same scope, the compiler won't just specify "`int*`", but actually, "`int[8]`" as type for `sizeof()` to parse.
    
4. `sizeof()` not only doesn't use the value of its argument, but it doesn't even resolve its argument.
    This not only means we can specify `a[0]` for an empty array, but `a[13]` for a 10-element array would work just fine.
    
    
## On array declaration syntax
    
In Java, we declare (and initialise) arrays as
```
int[s] my_array = {...}
```
to be read as "create a reference to the integer array object of size `s` called `my_array` with values `...`". 

In C, we declare
```
int my_array[s] = {...}
```
to be read as "create a reference to an integer `my_array` in memory, and now that you're at it, reserve `s` of those locations starting there, with values `...`".


## On arrays as return types for functions

In Java, `int[] myFunction(){...}` is everyday business. Alas, C simply does not allow to directly return arrays as return values for functions. Here are three arguments as to why:
1. The function would have to specify the size of the array it'd return, which it cannot do dynamically.

2. Arrays are created in the procedure's stack frame; when the procedure returns, the freshly created array virtually disappears (it might still exist under the stack pointer.

3. Arrays are, arguably, abstract data types. They don't jive with low-level programming, which focuses on as little abstraction as possible without writing straight-up assembly.

In-place functions (e.g. `void multArray(int arr[]){...}` would of course work.