# GCC-Inline-Assembly-HOWTO
## GCC Assembly Syntax
1. Source-Destination Ordering

   The direction of the operands in AT&T syntax is opposite to that of Intel. In Intel syntax the first operand is the destination, and the second operand is the source whereas in AT&T syntax the first operand is the source and the second operand is the destination. ie,

   "Op-code src dst" in AT&T syntax.
2. Register Naming.

   Register names are prefixed by % ie, if eax is to be used, write %eax.

## Basic Inline
The format of basic inline assembly is very much straight forward. Its basic form is
```
asm("assembly code");
```
Example.
```
asm("movl %ecx %eax"); /* moves the contents of ecx to eax */
```

## Extended Asm.
Anyway the basic format is:
```
asm (assembler template
    : output operands              /* optional */
    : input operands               /* optional */
    : list of clobbered registers) /* optional */
```
The assembler template consists of assembly instructions.
```
int a=10, b;
asm ("movl %1, %%eax";
      movl %%eax, %0;"
      :"=r"(b)        /* output */
      :"r"(a)         /* input */
      :"%eax"         /* clobbered */
      );
```
Here what we did is we made the value of 'b' equal to that of 'a' using assembly instructions. Some points of interest are:
- "b" is the output operand, referred to by %0 and "a" is the input operand, referred to by %1.
- "r" is a constraint on the operands. We'll see constraints in detail later. For the time being, "r" says to GCC to use any register for storing the operands. output operand constraint should have a constraint modifier "=". And this modifier says that it is the output operand and is write-only.
- There are two %'s prefixed to the register name. This helps GCC to distinguish between the operands and registers. operands have a single % as prefix.
- The clobbered register %eax after the third colon tells GCC that the value of %eax is to be modified inside "asm", so GCC won't use this register to store any other value.

When the execution of "asm" is complete, "b" will reflect the updated value, as it is specified as an operand. In other words, the change made to "b" inside "asm" is supposed to be reflected outside the "asm".
### Operands
In the assemble template, each operand is referenced by numbers. Numbering is done as follows. If there are a total of n operands (both input and output inclusive), then the first output operand is numbered 0, continuing in increasing order, and the last input operand is numbered n-1.

### Clobber list
Some instructions clobber some hardware registers. We have to list those registers in the clobber-list, ie the field after the third ':' in the asm function. This is to inform gcc that we will use and modify them ourselves.

## More about constraints
### Commonly used constraints
There are a number of constraints of which only a few are used frequently. We'll have a look at those constraints.

1. **Register operand constraint**

   When operands are specified using this constraint, they get stored in General Purpose Registers(GPR). Take the following example:
   ```
   asm ("movl %%eax, %0\n" :"=r"(myval));
   ```
   Here the variable myval is kept in a register, the value in register `eax` is copied onto that register, and the value of `myval` is updated into the memory from this register.

2. **Memory operand constraint**

   When the operands are in the memory, any operations performed on them will occur directly in the memory location, as opposed to register constraints, which first store the value in a register to be modified and then write it back to the memory location. Memory constraints can be used most efficiently in cases where a C variable needs to be updated inside an "asm" and you really don't want to use a register to hold its value. For example, the value of idtr is stored in the memory location loc:
   ```
   asm("sidt %0\n" : :"m"(loc));
   ```

### Constraint Modifiers
1. "=": Means that this operand is write-only for this instruction; the previous value is discarded and replaced by output data.

## Some Useful Recipes

1. In Linux, system calls are implemented using GCC inline assembly. Let us look how a system call is implemented. All the system calls are written as macros (linux/unistd.h). For example, a system call with three arguments is defined as a macro as shown below:

   ```
   #define _syscall3(type,name,type1,arg1,type2,arg2,type3,arg3) \
   type name(type1 arg1,type2 arg2,type3 arg3) \
   { \
   long __res; \
   __asm__ volatile (  "int $0x80" \
                     : "=a" (__res) \
                     : "0" (__NR_##name),"b" ((long)(arg1)),"c" ((long)(arg2)), \
                       "d" ((long)(arg3))); \
   __syscall_return(type,__res); \
   ```
   Whenever a system call with three arguments is made, the macro shown above is used to make the call. The syscall number is placed in eax, then each parameters in ebx, ecx, edx. And finally "int 0x80" is the instruction which makes the system call work. The return value can be collected from eax.

