# CASS Notes --- RISC-V Programming --- Stack Use and Word Alignment

## Stack Mechanics
- The stack is a variable-size space in memory that starts at the *top* of the program memory. Indeed, it must therefore grow **downwards**, even though data still grow upwards when stored "onto" (i.e. hung from the tail of) the stack.

- Adding (pushing) and removing (popping) data from the stack only happens to and from its end. The address of the stack's end is stored in the *stack pointer* register `sp` (formally: `x2`).

    - RISC-V doesn't offer a `pop()` nor `push()` instruction. We do this manually: for example, to add 4 bytes stored in the `x5` register to the stack, one would call 
    
        ```python
        addi sp, sp, -4
        sw   x5, 0(sp)
        ```

    - Common misconception: because HLLs tend to return the value at the end of the stack when calling `pop()`, people think that's the only way to *read* from the stack. This is incorrect: you can "`peek()`" the stack relative to the stack pointer (e.g. `+4(sp)`). (Imagine if you couldn't, and you stored an array on the stack!)

    - The stack pointer points to the last filled byte on the stack. **Never assume any procedure has been courteous enough to leave the stack pointer free for use.** This is illustrated in the memory figure below. (Note: the `0`s and `1`s represent bytes. Also, "*sp*" denotes that that byte's address is stored in `sp`, not that `sp` is stored at the denoted location.)

        |       |       |       |       |
        | ---   | ---   | ---   | ---   |
        |   1   |   0   |   1   |   1   |
        |   1   |   1   |   0   |   0   |
        |   0   |   1   |   1   |   1   |
        |   1   |   1   |   0   |   1   |
        |   1   |   1   |   1   |   *sp*   |
        |   -   |   -   |   -   |   -   |
        |   -   |   -   |   -   |   -   |
        |   ...   |  ...   |   ...   |   ...   |

- Some good practice w.r.t. the stack:

    - In RISC-V, the stack pointer has an alignment restriction of 2 doublewords (two blocks of 8 bytes). For assembly programming exercises, agreeing to a 1-word (4-byte) alignment restriction instead makes way life easier.

    - There are multiple ways of phrasing "lower stack pointer and write data to the new address" as a chain of instructions. However, some ways are better than others. Below are some alternatives for doing exactly this.

        ```python
        # 1. This is what you should do: reserve stack room (correct protocol) and do it once (efficient)
        addi sp, sp, -16
        sw x5, 12(sp)
        sw x5, 8(sp)
        sw x5, 4(sp)
        sw x5, 0(sp)
        addi sp, sp, +16  # At the end of a procedure, free the stack space again.

        # 2. This is wasteful use of the ALU (and blocks multi-issuing!), but still correct protocol.
        addi sp, sp, -4
        sw x5, 0(sp)
        addi sp, sp, -4
        sw x5, 0(sp)
        addi sp, sp, -4
        sw x5, 0(sp)
        addi sp, sp, -4
        sw x5, 0(sp)
        addi sp, sp, +16

        # 3. This is incorrect protocol; by not updating sp, a called procedure wouldn't know about the new data on the stack!
        sw x5, -4(sp)
        sw x5, -8(sp)
        sw x5, -12(sp)
        sw x5, -16(sp)
        ```

    - When someone saves the value of `sp` at the start of a procedure (probably for later use in that procedure), we call that value "the procedure's *frame pointer*".
    
        - RISC-V hardware doesn't do this for you automatically, although there is the convention among programmers to store the frame pointer to `s0` (formally: `x8`), because that's a reserved register and is hence safe from being overwritten by nested procedure calls. Of course, exactly because `s0`/`x8` is a reserved register, you'll need to first save whatever value it's already holding to the stack - think for a moment how you'd implement all of this, because there is a quirky Catch-22 you'll need to work around.
        
        - Frame pointers can be of help for the following: some complex procedures need to dynamically add more space to the stack. This poses an annoying problem: imagine if we had a value stored at `0(sp)`, and then an extra word was pushed to the stack. Now, our original value should be referred to by `+4(sp)`; hence, since the frame pointer is static throughout the same procedure, referring to stack varianbles relative to `s0`/`x8` is better: in the example, the original value is at `0(sp)` and then at `+4(sp)` yet always stays at `0(x8)`.


## On the Topic of Word Sizes: C Integers in RV32 vs. RV64

- Any whole number type in C can be suffixed by "`int`": `long long == long long int`, and `short == short int`, and ...

- Whole number types in C have a debatable amount of bits. The convention for RISC-V's 32-bit and 64-bit architectures is simply this:

    | Type | Bytes | Recognisable positive 2's limit |
    | --- | --- | --- |
    | char | 1 byte (8 bits) | 127 |
    | short | 2 bytes (16 bits) | 32 767 |
    | int |  4 bytes (32 bits) | 2 147 483 647 |
    | long long | 8 bytes (64 bits) | 9.223e18 |
    | long/void* | <***width of integer registers***> | -|
 	
    That is: in `rv32`, a `long` or an address is 4 bytes (32 bits), whilst in `rv64`, both are 8 bytes (64 bits).

- Since the `rv32` architecture has 32-bit (4-byte) registers, a C `long long` doesn't fit in one. To still host 64-bit variables and values on `rv32`, such a value's bits are split over two consecutive registers. For function return values, the split happens as `x11|x10`. (64-bit function arguments are split similarly, or spilled to stack.)

    - Memory is just one long string of bytes, and hence storing a split return value is straight-forward:
        ```python
        sw x11, 4(x5)  # Store upper bits of result
        sw x10, 0(x5)  # Store lower bits of result
        ```
