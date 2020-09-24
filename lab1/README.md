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

Second, there are only character operations in [console.c](../jos/kern/console.c), so it is for the most low-level operations that related to hardware. 

After that, let's look at [printfmt.c](../jos/lib/printfmt.c). The `vprintfmt` is the common operation for `vsnprintf` (print to a string buffer), `cprintf` (print to the console), `fprintf` (print to a file). It parses the `fmt`, fills the fields with data in va_list, and writes the result into `putdat` using the function `putch`. Hence there is nothing to say about function `vsnprinf`, for it is only a wrapper. The same is true for [printf.c](../jos/kern/printf.c). 

The function `readline` copys the contents before the first "\n" in `prompt` into a static global char array `buf` of length 1024. Two things are interesing about readline: 

1. The `buf` is a static global array for the [readline.c](../jos/lib/readline.c) module, but it is not decorated by a `extern` keyword. This means: first, the other module cannot use the `buf` symbol directly; second, `readline` always returns the same pointer. 

1. I used to take echoing and backspace for granted. However, these actions are provided by the kernel. Once a visible character is entered, `readline` will call the `cputchar` function, so that we can check we have pressed. If a "\b" is received, the cursor (a pointer to an address in the `buf`) moves one step back. This is reasonable, but it still surprises me to some extent, because I never thought about this before.

I cannot find the [fprintf.c](), so I guess it is meant to be written by myself. 

### The stack

Learning... I need knowledge of GCC calling convention to understand this part.

