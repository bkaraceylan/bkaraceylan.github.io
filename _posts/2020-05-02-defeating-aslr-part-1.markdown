---
layout: post
title: "Defeating ASLR Part I"
date: 2020-05-02 01:06:00 +0300
categories: hack binary-exploitation
---
[Second part](/hack/binary-exploitation/2020/05/02/defeating-aslr-part-2.html) of this series contains the exploitation process.
<br>
This is the first post of a two part series. In this post we're going to learn a bit about virtual memory, dynamic linking, position indepentend code, and ASLR protection. All of these topics are very specific and low-level on their own but we're only going to scratch the surface and learn just enough to comprehend what we're doing in the second part of this series where we'll take advantage of a memory corruption vulnerability to exploit an ELF binary with defeating ASLR protection. We are going to use the binary from the hackthebox machine Ellingson. A big thanks to the creator of this lovely box.


# WTF is ASLR?

Address Layout Space Randomization is a protection technique to prevent the exploitation of vulnerabilities related to memory corruption. It does so by randomizing the virtual memory mapping of the components of a binary such as the address of stack, heap, and libraries etc. The idea is that if the attacker doesn't know the address of a crafted payload in the memory or the addresses of certain leverageable functions such as system, he's gonna have a tough time pwning.

## Virtual Memory

When you launch a process, the operating system doesn't allocate space right on the physical memory (RAM). Instead the OS allocates chunks of memory called *pages* and while the pages of prioritized processes reside on the RAM, pages of processes with less priority are stored on the slow hard drive. OS switches the pages between RAM and HDD as needed.
<br>
Each process has it's own virtual address space and doesn't know anything about the address space of other processes. This creates a layer of security and allows easier programming since all a process needs to know are it's own virtual address space and the OS will translate these virtual addresses to physical addresses on the RAM.
<br>
Below is the virtual memory mapping of an example program. Here you can see the addresses of the different sections of the program, the address belonging to libc and dynamic linker, and finally the stack.
<br>


```bash
Start              End                Offset             Perm Path
0x0000000000400000 0x0000000000401000 0x0000000000000000 r-- /root/Tuts/hbox/ellingson/exploits/garbage
0x0000000000401000 0x0000000000402000 0x0000000000001000 r-x /root/Tuts/hbox/ellingson/exploits/garbage
0x0000000000402000 0x0000000000403000 0x0000000000002000 r-- /root/Tuts/hbox/ellingson/exploits/garbage
0x0000000000403000 0x0000000000404000 0x0000000000002000 r-- /root/Tuts/hbox/ellingson/exploits/garbage
0x0000000000404000 0x0000000000405000 0x0000000000003000 rw- /root/Tuts/hbox/ellingson/exploits/garbage
0x00007ffff7ddf000 0x00007ffff7e04000 0x0000000000000000 r-- /usr/lib/x86_64-linux-gnu/libc-2.30.so
0x00007ffff7e04000 0x00007ffff7f4e000 0x0000000000025000 r-x /usr/lib/x86_64-linux-gnu/libc-2.30.so
0x00007ffff7f4e000 0x00007ffff7f98000 0x000000000016f000 r-- /usr/lib/x86_64-linux-gnu/libc-2.30.so
0x00007ffff7f98000 0x00007ffff7f9b000 0x00000000001b8000 r-- /usr/lib/x86_64-linux-gnu/libc-2.30.so
0x00007ffff7f9b000 0x00007ffff7f9e000 0x00000000001bb000 rw- /usr/lib/x86_64-linux-gnu/libc-2.30.so
0x00007ffff7f9e000 0x00007ffff7fa4000 0x0000000000000000 rw- 
0x00007ffff7fd0000 0x00007ffff7fd3000 0x0000000000000000 r-- [vvar]
0x00007ffff7fd3000 0x00007ffff7fd4000 0x0000000000000000 r-x [vdso]
0x00007ffff7fd4000 0x00007ffff7fd5000 0x0000000000000000 r-- /usr/lib/x86_64-linux-gnu/ld-2.30.so
0x00007ffff7fd5000 0x00007ffff7ff3000 0x0000000000001000 r-x /usr/lib/x86_64-linux-gnu/ld-2.30.so
0x00007ffff7ff3000 0x00007ffff7ffb000 0x000000000001f000 r-- /usr/lib/x86_64-linux-gnu/ld-2.30.so
0x00007ffff7ffc000 0x00007ffff7ffd000 0x0000000000027000 r-- /usr/lib/x86_64-linux-gnu/ld-2.30.so
0x00007ffff7ffd000 0x00007ffff7ffe000 0x0000000000028000 rw- /usr/lib/x86_64-linux-gnu/ld-2.30.so
0x00007ffff7ffe000 0x00007ffff7fff000 0x0000000000000000 rw- 
0x00007ffffffde000 0x00007ffffffff000 0x0000000000000000 rw- [stack]
``` 

## What does ASLR has to do with virtual memory?

ASLR works by randomizing the virtual address space. Every time the program is run, the memory addresses of the libraries and the stack shown in the above virtual memory map will change. So, if you are executing a ret-2-libc or ROP gadget now you do not know the address of the libc, or if you are trying to execute a shellcode in memory now you do not know the address of the stack. You can run ldd multiple times to observe this behaviour.
<br>
```terminal
root@kali:~/Tuts/hbox/ellingson/exploits# ldd garbage 
        linux-vdso.so.1 (0x00007fff58fa3000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f0bb7f4f000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f0bb8140000)
root@kali:~/Tuts/hbox/ellingson/exploits# ldd garbage 
        linux-vdso.so.1 (0x00007ffe37d20000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fe96848f000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fe968680000)
```
<br>
See how the address of the libraries change each time? You can disable ASLR and try the commands again.
<br>
```terminal
root@kali:~/Tuts/hbox/ellingson/exploits# echo 0 | tee /proc/sys/kernel/randomize_va_space
0
root@kali:~/Tuts/hbox/ellingson/exploits# ldd garbage 
        linux-vdso.so.1 (0x00007ffff7fd3000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ffff7ddf000)
        /lib64/ld-linux-x86-64.so.2 (0x00007ffff7fd4000)
root@kali:~/Tuts/hbox/ellingson/exploits# ldd garbage 
        linux-vdso.so.1 (0x00007ffff7fd3000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ffff7ddf000)
        /lib64/ld-linux-x86-64.so.2 (0x00007ffff7fd4000)
root@kali:~/Tuts/hbox/ellingson/exploits# 
```
<br>
Now the libraries are loaded on the exact same memory address every time. Don't forget the enable ASLR back again.
<br>
```terminal
echo 2 | tee /proc/sys/kernel/randomize_va_space
```
<br>
## How bad is it?

It does make the life of an attacker harder, but it is not the end of the world. There are multiple ways around it such as relative addressing, bruteforcing etc. but we're going to talk about the address leaking technique. For that we need to learn a bit about GOT and PLT.

## What the hell is GOT and PLT?

So there are certain similar procedures that nearly every program have in common such as printing things to stdout or getting inputs from stdin. It would be very inefficent if every programmer had to write their own functions for such common procedures. Thus came the libraries. Libraries are pieces of code that can be shared between programs. One such example library is the famous libc.so.6 in GNU/Linux systems. If you want to print something to stdout in your C program you can use the *printf* function. This printf function resides in libc library and is accessible by every C binary. Now, there are two ways a program knows the address of a function in a library: *static linking* and *dynamic linking*. Statically linked binaries contain all of the code in the libraries that it uses in the final executable. Consider having hundreds of statically linked C programs running on a linux system and all of them use libc. This means each of the programs will have a copy of libc and load it in the memory. This is a waste of storage and memory. In dynamic linking only minimal information about the libraries are included in the final executable. Libraries are loaded at load time or runtime and the function addresses are resolved. If a copy of a library is in the memory it is shared by all applications that use the same library. You can use the below command to see which processes a particular library is linked to.
<br>
```terminal
lsof /lib/x86_64-linux-gnu/libc.so.6 

...
bash       3651       root mem    REG    8,1  1831600 1838471 /usr/lib/x86_64-linux-gnu/libc-2.30.so
bash       4341       root mem    REG    8,1  1831600 1838471 /usr/lib/x86_64-linux-gnu/libc-2.30.so
bash       5367       root mem    REG    8,1  1831600 1838471 /usr/lib/x86_64-linux-gnu/libc-2.30.so
vim        5399       root mem    REG    8,1  1831600 1838471 /usr/lib/x86_64-linux-gnu/libc-2.30.so
Web\x20Co  5425       root mem    REG    8,1  1831600 1838471 /usr/lib/x86_64-linux-gnu/libc-2.30.so
bash      10700       root mem    REG    8,1  1831600 1838471 /usr/lib/x86_64-linux-gnu/libc-2.30.so
lsof      10834       root mem    REG    8,1  1831600 1838471 /usr/lib/x86_64-linux-gnu/libc-2.30.so
lsof      10835       root mem    REG    8,1  1831600 1838471 /usr/lib/x86_64-linux-gnu/libc-2.30.so
``` 
<br>
Function addresses in a shared library can't be hardcoded during compile. Because any update on the shared library that changes the addresses of functions would break all of the programs that use it. And the address of the libraries would have to be static and things like ASLR wouldn't be possible. Resolving of the addresses of shared libraries during runtime is done by the linker (ld-linux.so) and this process is called *relocation*.

## Relocation

Relocations are basicaly dummy entries in executables that are later filled at link time or at runtime. Below are the relocations of an executable. Getting deep into relocations is out of scope. What you need to know is below table tells that the address of the symbol in an entry will be resolved and patched into the specified offset.
<br>
```terminal
root@kali:~/Tuts/hbox/ellingson# readelf --relocs garbage 

Relocation section '.rela.dyn' at offset 0x6a0 contains 3 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000403ff0  000b00000006 R_X86_64_GLOB_DAT 0000000000000000 __libc_start_main@GLIBC_2.2.5 + 0
000000403ff8  000f00000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
0000004040d0  001700000005 R_X86_64_COPY     00000000004040d0 stdin@GLIBC_2.2.5 + 0

Relocation section '.rela.plt' at offset 0x6e8 contains 20 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000404018  000100000007 R_X86_64_JUMP_SLO 0000000000000000 putchar@GLIBC_2.2.5 + 0
000000404020  000200000007 R_X86_64_JUMP_SLO 0000000000000000 strcpy@GLIBC_2.2.5 + 0
000000404028  000300000007 R_X86_64_JUMP_SLO 0000000000000000 puts@GLIBC_2.2.5 + 0
000000404030  000400000007 R_X86_64_JUMP_SLO 0000000000000000 fclose@GLIBC_2.2.5 + 0
000000404038  000500000007 R_X86_64_JUMP_SLO 0000000000000000 getpwuid@GLIBC_2.2.5 + 0
000000404040  000600000007 R_X86_64_JUMP_SLO 0000000000000000 getuid@GLIBC_2.2.5 + 0
000000404048  000700000007 R_X86_64_JUMP_SLO 0000000000000000 printf@GLIBC_2.2.5 + 0
000000404050  000800000007 R_X86_64_JUMP_SLO 0000000000000000 rewind@GLIBC_2.2.5 + 0
000000404058  000900000007 R_X86_64_JUMP_SLO 0000000000000000 fgetc@GLIBC_2.2.5 + 0
000000404060  000a00000007 R_X86_64_JUMP_SLO 0000000000000000 read@GLIBC_2.2.5 + 0
000000404068  000c00000007 R_X86_64_JUMP_SLO 0000000000000000 fgets@GLIBC_2.2.5 + 0
000000404070  000d00000007 R_X86_64_JUMP_SLO 0000000000000000 strcmp@GLIBC_2.2.5 + 0
000000404078  000e00000007 R_X86_64_JUMP_SLO 0000000000000000 getchar@GLIBC_2.2.5 + 0
000000404080  001000000007 R_X86_64_JUMP_SLO 0000000000000000 gets@GLIBC_2.2.5 + 0
000000404088  001100000007 R_X86_64_JUMP_SLO 0000000000000000 syslog@GLIBC_2.2.5 + 0
000000404090  001200000007 R_X86_64_JUMP_SLO 0000000000000000 access@GLIBC_2.2.5 + 0
000000404098  001300000007 R_X86_64_JUMP_SLO 0000000000000000 fopen@GLIBC_2.2.5 + 0
0000004040a0  001400000007 R_X86_64_JUMP_SLO 0000000000000000 __isoc99_scanf@GLIBC_2.7 + 0
0000004040a8  001500000007 R_X86_64_JUMP_SLO 0000000000000000 strcat@GLIBC_2.2.5 + 0
0000004040b0  001600000007 R_X86_64_JUMP_SLO 0000000000000000 exit@GLIBC_2.2.5 + 0
```
<br>
ELF files are made up of different sections. Sections such as .text, .data, .rodata, and .bss may be familiar to you. There are also sections used during relocations, namely: .got, .plt, .got.plt, .plt.got. 
<br>
* **.got (Global Offset Table):** This section is where the linker puts the addresses of resolved global variables. You can see in the previous relocation table that relocations in .rela.dyn section that are of type GLOB_DAT and COPY point to offsets in the .got section of the executable.
* **.plt (Procedure Linkage Table):** This section contains stubs of codes that either jump to the right address or call the linker to resolve the address.
* **.got.plt: (GOT for PLT):** This section either contains the right address of the resolved function or points back to .plt to trigger the lookup to resolve the address. Entries in the .rela.plt of type JUMP_SLOT points to offsets in .got.plt.
<br>
<br>
What you need to know is that when a function is first called (ie. putchar) **(1)** it jumps to an address in .plt (putchar@plt) **(2)** which jumps to an address saved in GOT for PLT (.got.plt) **(3)** which at this time does not contain the right address and points back to the next instruction in .plt **(4)** which will actually do the lookup and patch the address in .got.plt with the right address of the function (putchar@GLIBC) and call it. Next time the function is called .got.plt entry in the third step contains the right address and the lookup is not triggered. There are many tutorials online that take you through this steps using a debugger or you can dissassemble the code and follow the addresses.
<br>
<br>
All of this make it possible to have position independent code (PIC) which means code without using absolute addresses. This in turn makes it possible for text sections of shared libraries to be loaded into the memory once and then mapped to virtual address spaces of processes.
<br>
<br>
Armed with all of this knowledge, we're going to battle ASLR protection in the [next post](/hack/binary-exploitation/2020/05/02/defeating-aslr-part-2.html).
