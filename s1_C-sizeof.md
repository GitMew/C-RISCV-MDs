# CASS Notes --- C Programming --- `sizeof()`

Four observations can be made w.r.t. C's in-built function `sizeof()`:
1. `sizeof()` is merely a compile-time trick. 
    You cannot dynamically calculate `sizeof()` inside a procedure for a passed array, since the compiler can't know the size of the passed array beforehand.
    
2. `sizeof()` only uses the type of its argument.
    Combined with the previous property, this means a procedure would default to the size of an array pointer (32 bits in this case).
    
3. If the compiler is able to statically determine the argument of `sizeof()`, it will specify the argument's subtype as much as possible.
    In the case of an array of size 8 being initialised in the same scope, the compiler won't just specify "`int*`", but actually, "`int[8]`" as type for `sizeof()` to parse.
    
4. `sizeof()` not only doesn't use the value of its argument, but it doesn't even resolve its argument.
    This not only means we can specify `a[0]` for an empty array, but `a[13]` for a 10-element array would work just fine.
    