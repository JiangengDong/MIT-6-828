# MIT 6.828

This repository is my implementation for [MIT 6.828](https://pdos.csail.mit.edu/6.828/2018/schedule.html)'s homework. 

## Setup the environment 

I use the [ms-vscode-remote.remote-containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension of VSCode so that I can work inside a container where all the dependencies are configured without corrupting my host OS. My [Dockerfile](./.devcontainer/Dockerfile) and [configuration file](./.devcontainer/devcontainer.json) are provided. 

I mount the X11 Unix socket of the host onto the container so that I can make use of the host's X server. This means that running `make qemu` under the [jos](./jos) folder can create a window as if it is running on the host machine. However, I still prefer the `make qemu-nox` version. 

To anyone that want to use my docker configuration: If the GUI cannot work, first check if your host has an X server. If so, you may need to disable the access control of the X server on the host with `xhost +`. You can re-enable the control with `xhost -` later.

## Lab1: Booting a PC

### CPU reset 

During this phase, CPU is controlled by its inner circuits, so we cannot affect anything. After reset, CS=0xf000, IP=0xfff0, which points to the last 16 bytes in the first 1MB region. 

### BIOS

This is stored in a ROM or a flash hence hard to change, so it is usually provided by the manufacturer (in our lab, the qemu). It occupies the 64KB region from 0xf0000 to 0xfffff. The ROM that contains BIOS is "hard-wired", so its physical address is fixed. The last instruction, which is pointed to after CPU reset, is usually a jump instruction to some other position in BIOS. 

The function of BIOS is to check the hardware status and read the first sector (512 bytes) of the hard disk, called the *boot sector*, into *low memory*. Low memory, a 640KB region from 0x00000 to 0xA0000, used to be the main workspace in the 16-bit era. The content in the boot sector is *boot loader*. BIOS will pass control to boot loader with a jump instruction.

In our case, BIOS reads the boot loader into the region from 0x7c00 to 0x7dff, and jumps to 0x7c00.

### Boot loader

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

1. The 32-bit value 0x00000034 at address 0x1001c indicates the offset of the first program header from the beginning of this ELF binary.

1. The 16-bit value 0x0003 indicates the number of program headers in this ELF binary. 

1. The 32-bit value 0x0010000c indicates where the entry point of this ELF binary is.

The next step is to load according to the program headers. They are as follows. 

```
0x10034:        0x00000001      0x00001000      0xf0100000      0x00100000
0x10044:        0x00007120      0x00007120      0x00000005      0x00001000
0x10054:        0x00000001      0x00009000      0xf0108000      0x00108000
0x10064:        0x0000a948      0x0000a948      0x00000006      0x00001000
0x10074:        0x6474e551      0x00000000      0x00000000      0x00000000
0x10084:        0x00000000      0x00000000      0x00000007      0x00000010
```

Each program header takes 8 double words. The second double word is the offset of this program section from the beginning of the ELF binary. The forth double word is the physical address in memory that the program section should be loaded. The fifth double word is the length of the program section. These information tells us how to load program into the memory. In this case, the program is loaded to physical address 0x00100000, the first byte after the first 1MB region. 

The loaded program is as follows. It is exactly the same as obj/kern/kernel.asm.

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
    
### Kernel

Learning ...
