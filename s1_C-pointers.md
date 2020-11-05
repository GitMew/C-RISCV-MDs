# CASS Notes --- C Programming --- Pointers and Dynamic Data

## Pointers
- As a warm-up, consider the following peculiarity: all numbers have a size and a value. Strangely, those don't have to match: a number's value can indicate the size of another number.
    - E.g.: the number `111` in binary has a size of 3 and a value of 7. It represents, for example, the size of `1000101`, but not its own size.
- Similarly, each piece of data stored in memory has an address and a value. A value can represent the address to another value, even if it's not its own address. We call a value which has as its meaning the address of another value, a *pointer* or *reference*.
    - Practically, pointers are always integers, because addresses are. However, use of pointers is so common that they have their own C types, which have some special functionality as we will see.
    - Again, remember: pointers are just data of which the values have a special meaning. They, like other data, have an address themselves: indeed, a pointer pointing to a pointer exists.

- Consider the following C code (for `printf` and `scanf` syntax, see the MD regarding them):

```c
#include <stdio.h>

int n;
printf("Enter number: ");
scanf("%d", &n);

printf("=== int variable ===\n");
printf("Address: %d\n", &n);  // The & operator means "reference this" or "get address of". Every variable has an address.
printf(  "Value: %d\n",  n);
printf(   "Size: %d\n", sizeof(int));

int* p = &n;  // A starred type in declaration - like int* - denotes a pointer.

printf("=== int* variable ===\n");
printf(    "Address: %d\n", &p);
printf(  "Value (p): %d\n",  p);
printf( "Value (*p): %d\n", *p);  // The * operator means "dereference", or "expand" or "access". It retrieves the value a pointer points to.
printf(       "Size: %d\n", sizeof(int*));
```
- Mnemonic for pointer operators: **a**mpersan**d** returns **ad**dress, **a**steris**k** returns **ac**cess.

- A few notes on the `*` operator:
    - It is not to be confused with the star in pointer types.
    - It is unique to pointers. You can't dereference a non-pointer variable; theoretically, you could treat any variable's value as an address and dereference that, but the compiler says that's a very bad idea.
    - It actually does more than retrieve a value. Retrieving a value is what a function call does, e.g. `retrieveValue(pointy_thing)`. But **a dereference is not a function call**. The correct way to look at it, is that it **diverts memory access** of the pointer's own value to access of the value the pointer points at, for **both reads and writes**. It's as if the pointer dodges the access, and sacrifices the pointee instead. Indeed, the following code prints the same value twice:
        ```c
        #include <stdio.h>

        int n = 420;
        int* p = &n;
        
        // Dereferenced read:
        printf("n plus sixty-nine: %d\n", *p + 69);

        // Dereferenced write:
        *p = 420 + 69
        printf("n right now: %d\n", n);
        ```
    - Secretly, all variables are dereferenced pointers.
        - To pre-emptively clear up any doubts you might have when thinking about this: when passing a variable to a function, you pass its value, but you forget about its address. (Ignore this note if you're not thinking about it.)

- When we print pointers (so, addresses), it's conventional to cast the pointer to `(void*)`, which means "a pointer of no particular type". It makes no real difference for the print, but it's good practice.

    - Pointer type has impact on pointer arithmetic. E.g., for `(int*)` pointers, `p++` is compiled to `p += 4` to account for the size of an `int` in memory (4 bytes), whilst for `(char*)` pointers, `p++` merely becomes `p += 1` (since chars are 1-byte numbers).


## Arrays
### On array declaration syntax
    
- In Java, we declare (and initialise) arrays as
    ```Java
    int[s] my_array = {...}
    ```
    to be read as "create a reference to the integer array object of size `s` called `my_array` with values `...`". 

- In C, we declare
    ```C
    int my_array[s] = {...}
    ```
    to be read as "create a reference to an integer `my_array` in memory, and now that you're at it, reserve `s` of those locations starting there, with values `...`".

    - It's important to note that the latter isn't fully accurate. `my_array` is not a reference (a pointer) to an integer (so, an `int*`). Why not? Exactly because pointer type affects pointer arithmetic. The `int[s]` type is indeed like `int*` when read, but **unlike** `int*`, it does **not** allow incrementation. Hence, if you're going to call an integer array a pointer, don't call it an *integer pointer* (`int*`), but an *integer array pointer* (`int[s]`).

### On arrays as return types for functions

In Java, `int[] myFunction(){...}` is everyday business. Alas, C simply does not allow to directly return arrays as return values for functions. Here are three arguments as to why:
1. The function would have to specify the size of the array it'd return, which it cannot do dynamically.

2. Arrays are created in the procedure's stack frame; when the procedure returns, the freshly created array virtually disappears (it might still exist under the stack pointer.

3. Arrays are, arguably, abstract data types. They don't jive with low-level programming, which focuses on as little abstraction as possible without writing straight-up assembly.

In-place functions (e.g. `void multArray(int arr[]){...}` would of course work.

## Strings
- Just like arrays kind of look like pointers -- but not quite -- strings kind of look like arrays -- but not quite.
```c
char* my_str1 = "abcd";  // Immutable string
char my_str2[4];         // Mutable string, hence it's initialisable later on
strcpy(my_str2, "abcd");  // "String copy"
```

## Structs
- Imagine the following: you want to consistently group the same amount of values in memory, but they don't all have the same type, so an array doesn't work to keep them together. A `struct` is your friend!
```c
struct MyVeryFirstStructure {
    float   f;
    double  d;
    long    l;
    char    str[4];
    char    c;
};

struct MyVeryFirstStructure s1;  // Nothing initialised, but struct is now declared, and its fields can be assigned
s1.float = 0.0f
```