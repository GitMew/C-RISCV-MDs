# CASS Notes --- C Programming --- `sizeof(x)`
Sometimes, you want to do address math for arrays in C. At some point, you'll come across the `sizeof(x)` builtin function, which has surprisingly strange functionality. On the face of it, it returns the amount of bytes used for the argument `x`. However, working with arrays and `sizeof()`, four observations can be made:

1. `sizeof()` is merely a compile-time trick. 
    
    Inside a function, you cannot dynamically calculate `sizeof()` for an array that was passed as an argument, since the compiler can't know the (albeit fixed) length of the passed array beforehand.
    
2. `sizeof()` only uses the type of its argument.

    Indeed, this means that inside a function, C would default to returning the size of a passed array pointer (4 bytes if the argument is of type `int[]`) instead of the total amount of bytes used for the entire array (length times element size). For example:
    ```c
    int getArraySize(int unknown[]) {
        return sizeof(unknown);
    }
    int known[8] = {};
    int total_bytes = getArraySize(known);  // => int total_bytes = sizeof(int*);   =>   int total_bytes = 4;
    ```
    
3. If the compiler is able to statically determine the array argument `x` of `sizeof(x)`, it will specify its array subtype as much as possible.

    In the case of an array of size 8 being initialised in the same scope, the compiler won't just specify "`int*`", but actually, "`int[8]`" as type for `sizeof()` to parse. Indeed, `int[1]` (size 4 B), `int[2]` (size 8 B) ... are all array subtypes, which `sizeof()` uses only if such a subtype is mentioned in the scope of the `sizeof()` call. For example:
    ```c
    int known[8] = {};
    int total_bytes = sizeof(known);  // => int total_bytes = sizeof(int[8]);   =>   int total_bytes = 32;
    ```
    
    
4. `sizeof()` not only doesn't use the value of its argument, but it doesn't even resolve its argument.

    This means we could, for example, call `sizeof(a[0])` for an empty array `a`, `sizeof(a[13])` for a 10-element array `a` etc ... 
