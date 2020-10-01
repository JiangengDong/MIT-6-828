# Lab1: Booting a PC

## CPU reset 

During this phase, CPU is controlled by its inner circuits, so we cannot affect anything. After reset, `CS=0xf000`, `IP=0xfff0`, which points to the last 16 bytes in the first 1MB region. 

## BIOS

This is stored in a ROM or a flash hence hard to change, so it is usually provided by the manufacturer (in our lab, the qemu). It occupies the 64KB region from `0xf0000` to `0xfffff`. The ROM that contains BIOS is "hard-wired", so its physical address is fixed. The last instruction, which is pointed to after CPU reset, is usually a jump instruction to some other position in BIOS. 

The function of BIOS is to check the hardware status and read the first sector (512 bytes) of the hard disk, called the *boot sector*, into *low memory*. Low memory, a 640KB region from 0x00000 to 0xA0000, used to be the main workspace in the 16-bit era. The content in the boot sector is *boot loader*. BIOS will pass control to boot loader with a jump instruction.

In our case, BIOS reads the boot loader into the region from `0x7c00` to `0x7dff`, and jumps to `0x7c00`.

## Boot loader

It is the first part that we get involved in. The boot loader has two main purposes:

1. Change to 32-bit protected mode. All the phases above are under 16-bit real mode, so the memory is limited.

1. Read the kernel into memory, which is our most essential work in this lab. 

The kernel is ELF binary, which has a header that describes the layout of the file. Hence, we first read the header into memory, and then load the other parts of the file according to the header. The kernel is usually stored in the second sector on the hard disk, but it can vary for different OS, so the boot loader and the kernel should always be compatible.

The ELF header looks like this, 

```
0x10000:        0x464c457f      0x00010101      0x00000000      0x00000000
0x10010:        0x00030002      0x00000001      0x0010000c      0x00000034
0x10020:        0x00014328      0x00000000      0x00200034      0x00280003
0x10030:        0x0008000b      
```

There are three importent values in this header. 

1. The 32-bit value `0x00000034` at address 0x1001c indicates the offset of the first program header from the beginning of this ELF binary.

1. The 16-bit value `0x0003` indicates the number of program headers in this ELF binary. 

1. The 32-bit value `0x0010000c` indicates where the entry point of this ELF binary is.

The next step is to read the program headers. They are as follows. 

```
0x10034:        0x00000001      0x00001000      0xf0100000      0x00100000
0x10044:        0x00007120      0x00007120      0x00000005      0x00001000
0x10054:        0x00000001      0x00009000      0xf0108000      0x00108000
0x10064:        0x0000a948      0x0000a948      0x00000006      0x00001000
0x10074:        0x6474e551      0x00000000      0x00000000      0x00000000
0x10084:        0x00000000      0x00000000      0x00000007      0x00000010
```

Each program header takes 8 doublewords. The second doubleword is the offset of this program section from the beginning of the ELF binary. The forth doubleword is the physical address in memory that the program section should be loaded. The fifth doubleword is the length of the program section. These information tells us how to load program into the memory. In this case, the program is loaded to physical address 0x00100000, the first byte after the first 1MB region. 

The loaded program is as follows. It is exactly the same as [obj/kern/kernel.asm](../jos/obj/kern/kernel.asm).

```
0x10000c:    movw   $0x1234,0x472
0x100015:    mov    $0x110000,%eax
0x10001a:    mov    %eax,%cr3
0x10001d:    mov    %cr0,%eax
0x100020:    or     $0x80010001,%eax
0x100025:    mov    %eax,%cr0
0x100028:    mov    $0xf010002f,%eax
0x10002d:    jmp    *%eax
0x10002f:    mov    $0x0,%ebp
0x100034:    mov    $0xf0110000,%esp
0x100039:    call   0x100094
0x10003e:    jmp    0x10003e
0x100040:    push   %ebp
0x100041:    mov    %esp,%ebp
0x100043:    push   %ebx
0x100044:    sub    $0xc,%esp
```
    
## Kernel

The first thing the kernel does is to enable virtual memory. The reason we need virtual memory is that the link address and load address can be different. During the linking stage, the linker calculates most of the addresses under the assumption that the beginning of the ELF binary is at a fixed address, say 0xf010000. This is known as the "Linking address". However, when we actually load the ELF binary, the beginning of the ELF binary may vary, say 0x00100000, as is indicated by the second and third doublewords in the ELF header above. Now that the addresses in the binary are fixed and hard to change, we need virtual memory hardware to map any addresses referred to in the binary to the physical memory. The virtual memory will be discussed in the class later, so I don't know how the page table is defined yet. 

### Formatted printing

(I don't understand why we need to finish this task when these texts are written. Maybe it can familiarize us with the hardware? I hope I can understand it later. Anyway, let's start. )

(After I finish this part, the purpose of such design is clear. It is aimed to teach us how the hardware are abstracted in an operating system. For example, the screen, serial, etc. are stdout, while the keyboard is stdin. )

#### Background knowledge

The first thing I need to figure out is variable arguments. I didn't write too much code in C, so in fact, I don't know this feature until now. It is defined in the file [inc/stdarg.h](../jos/inc/stdarg.h), using the builtin function of GCC. The usage is as follows. 

```C
void fn(int first_arg, int second_arg, ...) {
    int temp;

    va_list ap; // a struct for the variable arguments stack
    va_start(ap, second_arg); // initialize ap with the last fixed argument. After initalization, ap will point to the address after second_arg
    temp = va_arg(ap, int); // interprete the address ap points to as int
    va_end(ap); // some clean up
}
```

`printf` is highly dependent on the feature. 

#### The structure of related code

Let's first look at [inc/stdio.h](../jos/inc/stdio.h), which reveals the outline of the code. 

```C
// lib/console.c
void	cputchar(int c);
int	getchar(void);
int	iscons(int fd);

// lib/printfmt.c
void	printfmt(void (*putch)(int, void*), void *putdat, const char *fmt, ...);
void	vprintfmt(void (*putch)(int, void*), void *putdat, const char *fmt, va_list);
int	snprintf(char *str, int size, const char *fmt, ...);
int	vsnprintf(char *str, int size, const char *fmt, va_list);

// lib/printf.c
int	cprintf(const char *fmt, ...);
int	vcprintf(const char *fmt, va_list);

// lib/fprintf.c
int	printf(const char *fmt, ...);
int	fprintf(int fd, const char *fmt, ...);
int	vfprintf(int fd, const char *fmt, va_list);

// lib/readline.c
char*	readline(const char *prompt);
```

First, we can ignore the functions `printfmt`, `snprintf`, `cprintf` and `fprintf`. They are wrappers for functions with the same name but prepended with a "v". 

Second, there are only character operations in [console.c](../jos/kern/console.c), so it is for the most low-level operations related to hardware. 

After that, let's look at [printfmt.c](../jos/lib/printfmt.c). The `vprintfmt` is the common operation for `vsnprintf` (print to a string buffer), `cprintf` (print to the console), `fprintf` (print to a file). It parses the `fmt`, fills the fields with data in va_list, and writes the result into `putdat` using the function `putch`. Hence there is nothing to say about function `vsnprinf`, for it is only a wrapper. The same is true for [printf.c](../jos/kern/printf.c). 

The function `readline` copys the contents before the first "\n" in `prompt` into a static global char array `buf` of length 1024. Two things are interesing about readline: 

1. The `buf` is a static global array for the [readline.c](../jos/lib/readline.c) module, but it is not decorated by a `extern` keyword. This means: first, the other module cannot use the `buf` symbol directly; second, `readline` always returns the same pointer. 

1. I used to take echoing and backspace for granted. However, these actions are provided by the kernel. Once a visible character is entered, `readline` will call the `cputchar` function, so that we can check we have pressed. If a "\b" is received, the cursor (a pointer to an address in the `buf`) moves one step back. This is reasonable, but it still surprises me to some extent, because I never thought about this before.

I cannot find the [fprintf.c](), so I guess it is meant to be written by myself. 

### The stack

By the GCC convention, the stack is in fact a "stack of stack frames": each function has a stack frame that contains the local variables. We should pay attention to two registers here: `%ebp` and `%esp`, which points to the bottom and top of the current stack frame.
```
        +------------+   |
        | arg 2      |   \
        +------------+    >- previous function's stack frame
        | arg 1      |   /
        +------------+   |
        | ret %eip   |   /
        +============+   
%ebp->  | saved %ebp |   \
        +------------+   |
        |            |   |
        |   local    |   \
        | variables, |    >- current function's stack frame
        |    etc.    |   /
        |            |   |
%esp->  |            |   |
        +------------+   /
```

Let's take a look at an example. Supposing `func1` calls `func2` and `func2` calls `func3`, the stack will look like follows. 

* (high address)
* local variables of `func1` (**L1**)
* last argument of `func2`
* ...
* first argument of `func2`
* return address for `func2` (the next instruction in `func1` after calling `func1`)
    * stack bottom of `func1` (**L2**)
    * local variables of `func2`
    * last argument of `func3`
    * ...
    * first argument of `func3`
    * return address for `func3` (the next instruction in `func2` after calling `func3`)
        * stack bottom of `func2` (**L3**)
        * local variables of `func3`
* (low address)

During the program execution, `esp` moves sequentially, increasing or decreasing 4 at a time, while `ebp` jumps among **L1**, **L2** and **L3**. The movement of `ebp` during the whole process is as follows. 

1. At first, `ebp` points to **L1**. 
1. Once entering `func2`, `ebp` points to **L2**, whose content is the address of **L1**. 
1. Once entering `func3`, `ebp` points to **L3**, whose content is the address of **L2**. 
1. Once exiting `func3`, `ebp` returns to **L2** by reading the content of **L3**.
1. Once exiting `func2`, `ebp` returns to **L1** by reading the content of **L2**.

To sum up, if we ignore local variables, the stack is designed to recover the instruction and data address of the caller when exiting callee. If we take local variables into consideration, the stack is also designed to abandon them quickly by pointing `esp` to `ebp`.

We all know that the modern computer are based on the Princeton architecture, where the instruction and data are both stored in the same memory space. In practice, however, the instruction and data are stored in different sections in the memory. That's why we need the GCC calling convention: it provides a way to recover both instructions and data.

Some other tips:

1. %eax (and %edx, if return type is 64-bit) contains return value (or trash if function is void).
1. Various x86 instructions, such as call, are "hard-wired" to use the stack pointer register.
1. The ebp (base pointer) register, in contrast, is associated with the stack primarily by software convention.

### Backtrace

It is easy after I understand the stack. Several tips: 

1. The initial value of ebp is 0, which indicates where the backtrace should stop.

1. As I mentioned above, there are three program headers at the beginning of the ELF file. It may be confusing if we list the section headers with the command `objdump -h obj/kern/kernel`. There are 7 sections, why? Because the text, rodata, stab and stabstr sections are loaded as a whole. 

    ```
    Sections:
    Idx Name          Size      VMA       LMA       File off  Algn
      0 .text         00001831  f0100000  00100000  00001000  2**4
                      CONTENTS, ALLOC, LOAD, READONLY, CODE
      1 .rodata       0000079c  f0101840  00101840  00002840  2**5
                      CONTENTS, ALLOC, LOAD, READONLY, DATA
      2 .stab         000038e9  f0101fdc  00101fdc  00002fdc  2**2
                      CONTENTS, ALLOC, LOAD, READONLY, DATA
      3 .stabstr      000018fc  f01058c5  001058c5  000068c5  2**0
                      CONTENTS, ALLOC, LOAD, READONLY, DATA
      4 .data         0000a300  f0108000  00108000  00009000  2**12
                      CONTENTS, ALLOC, LOAD, DATA
      5 .bss          00000648  f0112300  00112300  00013300  2**5
                      CONTENTS, ALLOC, LOAD, DATA
      6 .comment      00000035  00000000  00000000  00013948  2**0
                      CONTENTS, READONLY
    ```
