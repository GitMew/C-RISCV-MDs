# CASS Notes --- RISC-V Programming --- Exception Handling & RARS System Calls

## Privilege Modes
- For security and virtualisation reasons, RISC-V processors can run in different modes of privilege. Some instructions are reserved for higher degrees of privilege, and if in the wrong mode, the processor will *trap* instruction execution and forward control of the situation to the OS.
    - All RISC-V support at least a single mode, being *machine mode*, which is allowed to issue any executable instruction.
    - Most processors support an additional mode with a slightly lower privilege degree, called *supervisor/system mode*. It cannot, in particular, alter vital processor hardware signals like the enabling of the floating-point unit. This is what we'd want the OS to be in.
    - Experimentally, a third, lower-privileged *user mode* has been suggested by RISC-V's `N` extension. RARS emulates a user-mode environment.

- Exceptions - problems halting execution, as discussed in the next section - are most often handled in supervisor mode, by branching to the OS.
    - Why not use machine mode, if that has more problem-resolving power? Alas, if an exception cannot handled by the supervisor, it's probably so bad machine mode can't either. This is why machine mode is better dubbed *boot mode*, and supervisor mode *operational mode*, if you will. That's why Patterson & Hennessy only mention supervisor mode.
    - An exception is slightly more than a branch, as it raises the privilege level. This in particular makes *system calls* possible (see final section).


## Exception ISA
- Sometimes, regular CPU operation is interrupted by a bad machine event taking place. Examples include:
    - Misaligned stack pointer;
    - Misaligned load address;
    - Undecodable instruction (e.g. unimplemented opcode);
    - Unprivileged call;
    - More useful halting, see system calls.
- RISC-V hardware is equipped for suspending CPU operation to catch exceptions and let them be handled properly, including dedicated registers and instructions.


### CSRs
- Control/Status Registers (CSRs) are one-off registers outside of the register file (`x0`-`x31`) and apart from the `pc` register. They hold information about peculiar events during execution (especially exceptions, in RISC-V).
    - As such, CSRs are *fundamentally not* accessible like the registers in the register file, because they do not live there. Specifically, they are not read during decoding, and they are not written during writeback.
    - For CSR access, special CSR instructions are implemented in hardware (see next sections).
- Each privilege mode has its own set of CSRs, prefixed by the first letter of the mode (`m` for machine, `s` for supervisor/system, `u` for user). They are as follows:

    | CSR (`x=m/s/u`) | Full name           | Usage |
    | ---       | ---                       | --- |
    | `xstatus` | Status                    | Various status bits informing about the CPU state |
    | `xcause`  | Cause                     | Cause code of exception |
    | `xepc`    | Exception Program Counter | Instruction address where exception occured |
    | `xtvec`   | Trap Vectored handler     | Address to exception handling procedure |
    | `xscratch`| Scratch                   | Free register for manipulation by the exception handler |


### CSR instructions
- As noted, reading from and writing to CSRs needs specialised instructions.
    - All such instructions are prefixed by `csr`. 
    - The processor implements 3 register-based instructions (and their 3 immediate-based counterparts, suffixed by `i`).

    | CSR instruction | Full name | Meaning |
    | --- | --- | --- |
    | `csrrs Rd, Csr, Rs` | Read-set   | Read `Csr` into `Rd`, and set `Csr` to `Csr OR Rs` |
    | `csrrw Rd, Csr, Rs` | Read-write | Read `Csr` into `Rd`, and set `Csr` to `Rs` |
    | `csrrc Rd, Csr, Rs` | Read-clear | Read `Csr` into `Rd`, and set `Csr` to `0` |
    | `mret|sret|uret` | Return from handler | Copy `xepc` into `pc` to resume execution |

- The RARS assembler offers additional meaningful pseudo-instructions that exploit the `x0` register internally:

    | CSR pseudo | Full name | Meaning |
    | --- | --- | --- |
    | `csrr Rd, Csr` | Read | Read `Csr` into `Rd` |
    | `csrw  Csr, Rs` | Write | Write `Csr` from `Rs` |
    | `csrwi Csr, Imm` | Write immediate | Write `Csr` from immediate `Imm` |
    - Notice the operand orders. They follow the same rule as the actual CSR instructions: data move from right to left. This differs from `sw`.

- Obviously, since CSRs are not accessible during decode nor writeback, they cannot function as operands/destinations: if an exception handler wants to manipulate values found in any of the CSRs, it'll need to load those into regular registers, which have connections to the ALU.
    - Sadly for the handler, non-immediate values must be moved to `xscratch` from a regular register, not a CSR. We'll deal with this conundrum in the next section.


### Custom exception handling
- `ustatus`'s first bit is also called `uie` (for *interrupt enable*). This bit needs to be set for custom exception handling.
    - In RARS, `uie` is `0` by default, hence it needs to be set.
    - After that, `utvec` can be set to the address of the first instruction of the exception handler.
    - Example setup:
        ```python
        csrsi 	uie, 1      # Enable custom interrupts in RARS
        la	    x5, handler
        csrw	utvec, x5	# Set User Trap VECtorised handler address
        ```

- The big conundrum of exception handlers is handler memory allocation. The handler likely wants to compute or retrieve at least one piece of data, or perhaps perform a system call (see next section). Yet, to perform any operation or load or argument pass ... you need a free register.
    - Assuming an exception can occur at any point in time, there is no guarantee that caller-callee conventions have been followed. Hence, all registers we normally assume to be safe to manipulate (`x5`-`x7`, `x10`-`x17` and `x28`-`x31`), **aren't safe**.
    - Furthermore, since stack pointer misalignment may have caused the exception, the stack is **not available** for storing operands to. 
    - Hence, this leaves two ways to allocate memory for the exception handler:
        1. Use the data segment (use `.data` to reserve `.space`) => fixed-size allocation
        2. Use dynamic memory (use an `ecall` to call `sbrk`) => ever-expandable allocation, though not freeable :(

    - Lastly, both need at least one free register (the first to store the address to the preallocated space in memory using `la ..., free_space_label_here`, the second to store the address to the created heap space as returned by `sbrk`). We have a free CSR `uscratch`, but ... 
        - Sadly, it can't be loaded to by `la` nor `sbrk`.
        - Furthermore, even if we wanted to use only one register in space, `uscratch` wouldn't suffice in the important use-case of loading data from informative CSRs (`ucause`, `uepc` ...).
        - Additionally, `uscratch` can't be used for passing arguments, nor arithmetic, nor branching.

- Doomsday scenario: a byte-aligned stack pointer when pushing a link register to the stack.
    - Not only does this cause a trap (`sp` should be doubleword-aligned), 
    - and not only does this make the procedure jump without having backed up its temporaries, 
    - but also, it makes the stack unusable to the handler. 

- The solution to the conundrum:
    - `uscratch` is really only good for storing a non-CSR register
    - `x10` is the ideal working space: it can be operated on, written to freely (like the address to fixed `.space`), passed as an argument, and used for return values (like the address of `sbrk` to "infinitely more" heap memory).
    - The solution: start the handler with `csrrw x0, uscratch, x10`, end it with `csrrw x10, uscratch, x0` and `uret`.

- A full exception handler:
    ```python
    .data
        handler_stash:  .space 4  # Space for spilling a register
        msg_exception1: .string "Exception with cause "
        msg_exception2: .string " occurred at address "
    .globl main
    .text
    handler:
        # 1. Stash one register to uscratch, so we can load in address(es) to our big stash
        csrr    x10, uscratch
        
        # 2. Load address into x10, and stash x17 (x10, neatly, is already stashed)
        la      x10, handler_stash
        sw      x17, 0(x10)
        
        # 3. Print ucause and uepc as part of a message
        la	    x10, msg_exception1
        li	    x17, 4	# PrintString(x10)
        ecall
        csrr 	x10, ucause
        li	    x17, 1  # PrintInt(x10)
        ecall
        la	    x10, msg_exception2
        li	    x17, 4	# PrintString(x10)
        ecall
        csrr 	x10, uepc
        li	    x17, 1  # PrintInt(x10)
        ecall
        
        # 4. Jump over line
        addi 	x10, x10, 4
        csrw 	uepc, x10
        
        # 5. Restore registers
        la      x10, handler_stash
        lw	    x17, 0(x10)
        csrr	x10, uscratch

        # 6. Return
        uret
        
    main:
        # Handler setup (enable custom handler, and set address to it)
        csrsi 	uie, 1      # Enable custom interrupts in RARS
        la	    x5, handler
        csrw	utvec, x5	# Set User Trap VECtorised handler address
        
        # Trap
        lw  	x5, 1       # Always traps (lw only allows word-aligned addresses, and 1 is byte-aligned)

        # Finish
        li      x17, 10     # Exit(0)
        ecall
    ```


## System Calls
- System calls are intentional exceptions. They're evoked by a special instruction (an environment call in RARS), and are used to interface with basic, useful OS functionality. Examples include:
    - Printing/reading integers, floats, characters, strings ... to/from console
    - Generating a random number
    - Playing a MIDI tone
    - Allocating heap memory
    - Exiting the program with a certain exit status

- RARS offers 42 environment calls, each referenced to by a unique code. The full documentation can be found [here](https://github.com/TheThirdOne/rars/wiki/Environment-Calls), and is very enlightening. Generally, environment calls work as follows:
    1. Load arguments into registers `x10` to `x16`;
    2. Load the system call code into `x17`;
    3. Execute the special instruction `ecall`.
- Example program:
    ```python
    li	    x17, 5   # ReadInt()
    ecall            # x10 = input

    mv 	    x5, x10  # ReadInt()   (note: x17 hasn't changed)
    ecall	         # x10 = input

    add 	x10, x5, x10
    li  	x17, 1   # PrintInt(x10)
    ecall

    li      x17, 10  # Exit(0)
    ecall
    ```
