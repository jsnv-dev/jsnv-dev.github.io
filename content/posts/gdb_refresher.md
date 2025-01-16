---
title: GDB Refresher
date: 2025-01-16T00:00:00Z
draft: false
tags:
  - assembly
  - re
categories:
  - notes
keywords:
  - assembly
  - re
  - learning
---

# GDB Refresher

This cheatsheet provides a concise summary of the key GDB commands and concepts discussed in [`Debugging Refresher`](https://www.youtube.com/watch?v=r185fCzdw8Y&list=PL-ymxv0nOtqqQzEncNuE6jetlJAiBUda-) tutorial. It covers basic commands, memory examination, variable manipulation, disassembly, breakpoint management, scripting, configuration, and tips for using GDB. 

## Basic Commands
- `gdb <program>`: Start GDB with a program
- `run [args]`: Run the program (with optional arguments)
- `break <function/line>`: Set a breakpoint
- `continue` or `c`: Continue execution
- `next` or `n`: Step over
- `step` or `s`: Step into
- `finish`: Run until the current function returns
- `print <expr>` or `p <expr>`: Print value of expression
- `display <expr>`: Display expression value after each step
- `info break`: List breakpoints
- `delete <breakpoint-num>`: Delete a breakpoint
- `quit` or `q`: Exit GDB

## Examining Memory and Registers
- `info registers` or `info reg`: Show register values
- `print $<register>`: Print specific register value (e.g., `print $rax`)
- `x/<format> <address>`: Examine memory
  - Formats: `x` (hex), `d` (decimal), `u` (unsigned decimal), `o` (octal), `t` (binary), `a` (address), `i` (instruction), `c` (char), `s` (string)
  - Example: `x/10i $rip` (examine 10 instructions at current instruction pointer)

## Variables and Memory Manipulation
- `set <variable> = <value>`: Set a variable's value
- `set $<register> = <value>`: Set a register's value
- `set {<type>}<address> = <value>`: Set memory at address

## Disassembly
- `disassemble <function>`: Disassemble a function
- `disassemble <address>`: Disassemble at an address

## Breakpoint Commands
- `commands <breakpoint-num>`: Specify commands to run when breakpoint is hit
- `silent`: Make breakpoint silent (use within `commands`)
- `end`: End list of commands for a breakpoint

## Scripting
- Create a file with GDB commands (e.g., `myscript.gdb`)
- Run with: `gdb -x myscript.gdb <program>`

## Configuration
- Create `~/.gdbinit` for persistent configurations
- `set disassembly-flavor intel`: Set to Intel syntax (add to `.gdbinit` for persistence)

## Plugins
- GEF (GDB Enhanced Features): Provides richer output and additional commands
- Enable in `.gdbinit` with: `source /path/to/gef.py`

## Tips
- Use `$_` to reference the last value printed
- Variables printed are stored as `$1`, `$2`, etc., for later reference
- Use `set pagination off` to disable paging for long outputs

## Reference
- Debugging Refresher - Robert - GDB Demo - 2022.09.16: https://www.youtube.com/watch?v=r185fCzdw8Y&list=PL-ymxv0nOtqqQzEncNuE6jetlJAiBUda-