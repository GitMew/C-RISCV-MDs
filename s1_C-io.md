# CASS Notes --- C Programming --- I/O

## I/O conventions for `main`
- The `main` function is used as an entry-point (i.e. where we start execution) when the C program is run after compilation.

- Whilst you can leave it be `void main() {}`, you probably want to allow arguments. There is a standard way, which accepts command-line arguments:

    - Input: `int argc` (argument count) and `char* argv[]` (argument values, i.e. an array of strings)

    - Output: return `0` (success) or `-1` (failure)

- Conventional MWE:
```c
int main(int argc, char* argv[]) {
    return 0;
}
```

## `gets(p)` and `puts(s)`
- The `<stdio.h>` library gives access to standard CLI input and output. You have to `#include` it.

- For regular I/O, `gets(pointer)` and `puts("your_text_here")` do just fine.
    - `gets`'s argument is the *address* to store the input string to, *not* just a regular variable. (That makes sense: what would you expect the internals of `gets` to look like if you passed a *value*? A reassignment of that value? No! Read the MD on pointers!)

## `scanf(s,p)` and `printf(s,v)`

- The `scanf` and `printf` functions both use *format strings*, which use special format codes as per below.

| Format string | Meaning |
| --- | --- |
| `%c` | character |
| `%s` | string |
| `%u` | unsigned (decimal) int | 
| `%d` | signed (decimal) int |
| `%lu` | long unsigned (decimal) int |
| `%li` | long signed (decimal) int |
| `%f` | float |
| `%lf` | double |
| `%o` | signed octal int |
| `%x` | signed hex int |

- Consider the following example:
    ```c
    #include <stdio.h>

    unsigned int n;

    // I
    printf("Enter number to take factorial of: ");
    scanf("%u", &n);

    // O
    printf("Your number: %d\n", n);  // Don't forget \n, or you'll have a bad time with printouts sticking together.
    printf("Your number times two: %ld\n", n * 2);
    ```
    - For both functions, the format string (see e.g. the `"%u"` for `scanf`) *converts* (not *requires*) the input/output. That means that if you enter -1 for a format string `"%u"` in `scanf`, it'll roll around to 4 294 967 295.


## `fgets()` and `fputs`
- For even more dynamic I/O, `fgets()` allows for a buffered stream (i.e. standard input, but also text files) read, and `fputs()` allows for a buffered stream write.
    - `fgets` takes a place to write streamed content to (a `char[]` usually), a buffer size to reserve in memory, and a stream to read from.
    - `fputs` takes a place to read content from (a `char[]` or a string literal, usually), and a stream to write to. 
- When using a file as I/O stream, you open the stream with `file = fopen("name.txt","mode")` and close them with `fclose(file)`.
    - The modes match those in Python's `open()` function (`r`, `w`, `w+` ...).
- Example:
    ```c
    #include <stdio.h>
    #define BUFFER 10 

    char bufferout[BUFFER];

    // Reading from stdin
    while (fgets(bufferout, BUFFER, stdin) != NULL) {};
    printf(bufferout);

    // Reading from a file
    file = fopen("file.txt", "r");
    while (fgets(bufferout, BUFFER, file) != NULL) {};
    printf(bufferout);
    fclose(file);

    // Writing to a file
    fp = fopen("file.txt", "w+");
    fputs("Hello world!", fp);
    fputs(" This gets concatenated!", fp);
    fclose(fp);
    ```