# CASS Notes --- RISC V Programming --- Introduction To The RARS Environment

## 1. Installation
- Just download the [latest `.jar` release from GitHub](https://github.com/TheThirdOne/rars/releases). It's open-source, like the rest of the RISC-V paradigm. 

- Documentation is [here](https://github.com/TheThirdOne/rars/wiki).


## 2. Environment 101
### Clicking around
- All the way at the top, you'll find the menu bar, and below, the toolbar for creating, storing, loading, assembling and running programs.

- Under that, you'll find an *Edit* and *Execute* tab. These will be your working areas. The *Text Segment* and *Data Segment* tabs under *Execute* is where instructions and data will be shown after assembly, respectively.

- On the right-hand side, the RISC-V registers are displayed with stored values. More on the naming in the final section.

- At the bottom, the tabs *Messages* and *Run I/O* support general information (assembler success, execution exit status ...) and special input/output calls respectively. *Messages* is all you need when starting out; see the documentation for using *Run I/O*.


### Creating, assembling and running a RISC-V program
- Once you click on the blank paper icon (*New File*) in the toolbar up above, you'll find an empty `*.asm` file in the text editor under *Edit*.

- After having written some assembly, save the file, and click the wrench-screwdriver icon (*Assemble*). The editor now jumps to the *Execute* tab, where the instructions have been loaded into the *Text Segment*, any data in the *Data Segment* (see next section). The first instruction is highlighted.

- Using the toolbar's rightmost set of buttons, you can run the full program, or go through it step-by-step in forwards or backwards direction. During execution (and afterwards), the *Text Segment* tab will continuously highlight the instruction about to be executed, and the *Data Segment* and *Registers* tabs will have values updated according to data manipulations done by the program.


## 3. Assembler directives
- The RARS editor extends writing plain instructions with control over data memory. This is accessible via *assembler directives*: reserved keywords starting with `.` and preceding the actual instructions in the editor.

- To get a taste of usage, first, an example program as written in the editor is shown below; it instantiates a string and a data word in memory, and returns both the length of the string as well as whether it is smaller than the word.
```python
.data
    mystr:  .string  "kiekeboe"  # Reserve and fill words in memory to store char bytes and ending byte \0
    n:      .word    25
.globl main         # Make sure the program doesn't start at .data
.text
main:
    la 	x10, mystr  # Load string's address into memory (literally: the address to label "mystr")
    lw  x11, n      # Load simple word into memory
    jalr x1, fun
fun:
    mv	    x5,  x10      # Move argument to x5
    addi	x10, x0,  -1  # counter = -1
    start:
        addi	x10, x10, 1      # counter += 1
        addi	x7,  x5,  x10    # address = base + counter (no counter*8, since chars are bytes)
        lb	    x7,  0(x7)       # char = *address
        bne 	x7,  x0,  start  # if (char != \0): loop back
    end:
    slt     x11, x10, x11        # x11 = 1 if the string length is strictly smaller than the word, else 0
nomorefun:  # x10 holds the given string's length, x11 holds whether that length is smaller than the given word
```

- The most popular assembler directives for control include: 

| AsmDirective | Meaning |
| --------- | ------- |
| `.data` | Put the following into the memory's data segment (see below for syntaces). |
| `.text` | Put the following into the memory's text segment. |
| `.globl L` | Make label `L` globally discoverable. If label is `main`, start running the program at `main`'s address. **Note: for this latter functionality to work, turn on *Settings > Initialize Program Counter to global 'main' if defined*.** |

- For `.data` specifically, useful declarations are:

| AsmDirective | Example usage | Meaning |
| --- | --- | --- |
| `.space` | `s: .space 4` | Reserve 4 bytes of space in memory (further data cannot overwrite this) |
| `.string` | `mystr: .string "abcd"`| Reserve and fill words in memory to store char bytes and ending byte `\0` |
| `.word` | `n: .word 25` | Store a word in memory |
| `.word` | `arr: .word 25, 42, 69` | Store consecutive words in memory (an "array") |
| `.double` | `d: .double 399.99`| Store a double-precision floating point number in memory |
| `.double` | `darr: .double 399.99, 42.1`| Store consecutive double-precision floating point numbers in memory (an "array") |


- Note: RARS simulates the `rv32i` architecture, not 32-bit. That means we don't use `ld` and `sd`, but rather `lw` and `sw`. This explains why we use `.word` to instantiate data i.o. `.dword`. 
    - There is support for 64-bit floating point registers.
    - Technically, C programs making use of 64-bit variables (`long long`s) can be compiled in `rv32i`, but each such value will use two registers (where the uppermost register stores the uppermost bits, see next section). In memory, a `.dword` can always be reserved, but by the former, that'll be impractical.

## 4. Pseudo-expressions
- To start off, labels are written "*my_label:*" as in the data declarations, and are linked to the first non-blank line of assembly code after (or the end of the program if nothing comes after).

- The editor supports all kinds of pseudo-instructions, as those in the following table.

| Pseudo-op | Meaning |
|------|---------|
| `j L`   | Jump to label `L` (converted to `jal x0, 0(L)`) |
| `li x5, 69`   | Load immediate (converted to `addi x5, x0, 69`) |
| `la L`   | Load *address* to label `L` (particularly useful for data in the data segment) |
| `bnez x5, L` | Branch to label `L` if `x5` not equal to `zero` (converted to `bne x5, x0, (L)`) |

- Additionally, to aid in remembering the conventional use for registers, they can be referred to by informal names as per the below.

| Informal | Actual | Origin |
| --- | --- | --- |
| `zero` | `x0` | `x0` is hardwired to the value `0` |
| `ra` | `x1` | return address (link register) |
| `sp` | `x2` | stack pointer - see S3 |
| `gp` | `x3` | global pointer - see S4 |
| `tp` | `x4` | thread pointer |
| `t0-t2` & `t3-t6` | `x5-x7` & `x28-x31` | temporaries |
| `s0-s1` & `s2-s11` | `x8-x9` & `x18-x27` | saved |
| `a0-a7` | `x10-x17` | arguments and return values |
| `pc` | `pc` | program counter |
