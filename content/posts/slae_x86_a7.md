---
title: "SLAE x86 - Assignment 0x7"
date: 2021-10-23T05:05:06Z
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

# Custom Shellcode Crypter
## Objectives
- Create a custom crypter
- Free to use any existing encryption schema
- Use any programming language

## Crypter
Another process to conceal a shellcode by means of encrypting with different schema. *"Encryption is the foundation of modern security."* as [snowden mentioned](https://twitter.com/Snowden/status/1451205615838183426). In case of exploit development, it is another technique such as [Encoding](../slae_x86_a4) & [Polymorphism](../slae_x86_a6) from previous assignments to make detection harder.

## Advanced Encryption Standard (AES)
The AES is originally Rijndael cipher, the winner of the NIST's AES selection, is widely used. It can be implemented with 128, 192, or 256 bits of key.
The algorithm consist of several steps: sub bytes, shift rows, mix columns, & add round key. More details of the process here: https://formaestudio.com/rijndaelinspector/archivos/Rijndael_Animation_v4_eng-html5.html.

Fortunately, AES can be implemented using the [`OpenSSL`](https://github.com/ruby/openssl) gem in Ruby. With the [`Cipher`](https://ruby.github.io/openssl/OpenSSL/Cipher.html) class, AES with 256-bit key can be intantiated by:

```ruby {linenos=false}
OpenSSL::Cipher.new('AES-256-CBC')
```

Encryption and decryption are possible by passing the method `.encrypt` and `.decrypt` into the instance as decribed in the [documentation](https://ruby.github.io/openssl/OpenSSL/Cipher.html#class-OpenSSL::Cipher-label-Choosing+either+encryption+or+decryption+mode). Also, a key is passed using the [`.key`](https://ruby.github.io/openssl/OpenSSL/Cipher.html#class-OpenSSL::Cipher-label-Choosing+a+key) method. But before passing the key, it can be processed further to make it 256-bit. OpenSSL's [`Digest`](https://ruby.github.io/openssl/OpenSSL/Digest.html) class is capable of creating a 256-bit hash from an input data by using its [`.digest`](https://ruby.github.io/openssl/OpenSSL/Digest.html#method-c-digest-label-Examples) method.

### Ruby Script
Below Ruby script used the mentioned functionalities [above](#advanced-encryption-standard-aes). Comments are added to explain the functionality of the codes used.
The script have three options in processing the shellcode: encrypt only, decode only, or decode & execute.

```ruby
#!/usr/bin/env ruby
# Author: Jason Villaluna

require 'optparse'
require "openssl"

def run
  parse_args
  validate_inputs
  process_shellcode
rescue OptionParser::MissingArgument
  puts "\e[33m[!] Missing argument:\e[0m"
  @parser.parse %w[--help]
rescue StandardError => e
  puts "\e[31m[-]\e[0m Encountered: \e[31m'#{e.class}' #{e}\e[0m"
  exit(1)
end

# Simple check to validate cli arguments
def validate_inputs
  messages = [
    "Please provide both shellcode and key",
    "Please choose one from encrypt, decrypt_only, and decrypt_run"
  ]
  messages.shift if [@shellcode, @key].all? { |arg| arg }
  messages.pop if ['encrypt', 'decrypt_only', 'decrypt_run'].include?(@execute)
  return if messages.empty?

  messages.each { |message| puts "\e[33m[!] #{message}\e[0m" }
  exit(1)
end

# Choose which method to execute depending on the user's options
def process_shellcode
  case @execute
  when /^encrypt$/i
    encrypt_shellcode
  when /^decrypt_only$/i
    decrypt_only
  when /^decrypt_run$/i
    decrypt_run
  end
end

# Encrypt the shellcode with AES-256-CBC
def encrypt_shellcode
  encrypted_shellcode = aes(@key, @shellcode) { |aes| aes.encrypt }
  puts "\e[32m[+] Encrypted successfully:\e[0m "\
    "'#{encrypted_shellcode.to_s.unpack('C*').map { |c| '\x%02x' % c }.join}'"
end

# Decrypt the encrypted shellcode
def decrypt_shellcode
  shellcode = [@shellcode.split('\\x').join].pack('H*')
  aes(@key, shellcode) { |aes| aes.decrypt }
end

# Displays the decrypted shellcode in stdout
def decrypt_only
  decrypted_shellcode = decrypt_shellcode
  puts "\e[32m[+] Decrypted successfully:\e[0m '#{decrypted_shellcode}'"
end

# Decrypt the shellcode, compile using 'shell.c', and then run
def decrypt_run
  @decrypted_shellcode = decrypt_shellcode
  replace_shellcode
  compile_binary
  run_binary
end

# AES-256-CBC encryption and decryption
# depending on what code is pass in the block
def aes(key, data)
  aes = OpenSSL::Cipher.new('AES-256-CBC')
  yield(aes)
  aes.key = OpenSSL::Digest.digest("SHA256", key)
  aes.update(data) + aes.final
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
    c_code.gsub!(/\"\S+\"\;/, "\"#{@decrypted_shellcode}\"\;")
  end
end

# Compile the shellcode into binary with the help of the shellcode.c program
def compile_binary
  `gcc -fno-stack-protector -m32 -z execstack shellcode.c -o shellcode 2> /dev/null`
  puts "\n\e[32m[+]\e[0m Successfully generated \e[32mshellcode\e[0m "
end

# Execute the shellcode binary
def run_binary
  system('./shellcode')
end

def parse_args
  @parser = OptionParser.new do |opts|
    opts.banner = "Usage: #{$0} -s [shellcode] -k [key] -[e|d|r]"
    opts.separator ""
    opts.separator "Options:"

    opts.on("--shellcode", "-s SHELLCODE", "Input Shellcode") do |value|
      @shellcode = value
    end

    opts.on("--key", "-k KEY", "Pass key") do |value|
      @key = value
    end

    opts.on(
      "--execute", "-e [encrypt|decrypt_only|decrypt_run]",
      "\n\t\tencrypt: displays encrypted shellcode"\
      "\n\t\tdecrypt_only: displays decrypted shellcode:"\
      "\n\t\tdecryp_run: decrypt, compile, and then run the shellcode"
           ) do |value|
      @execute = value&.downcase
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
end
```

#### Script usage and testing
![](aes.gif)

---
This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: [http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)

Student ID: SLAE - 1558

---
[`Link to Github repository`](https://github.com/jsnv-dev/slae_x86_assignments/tree/main/A7) | [`Link to index page`](../slae_x86)
