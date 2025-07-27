---
layout: post
title:  "Parse the coredump file - a practice of gdb "
subtitle: "use the GDB more than backtrace"
header-img: img/post-bg-coffee.jpeg
date:   2025-07-27 08:00:00
author: half cup coffee
catalog: true
tags:	
    - Linux
    - QNX
---

# Introduction

Debugging crashes can often feel like detective work, meticulously piecing together clues to understand what went wrong. When an application on a QNX system (or any Linux-like environment) crashes, it leaves behind a core dump – a snapshot of its memory at the moment of failure. This forensic artifact is gold for debugging, and GDB (the GNU Debugger) is our primary tool for mining its secrets.

Many developers dread core dumps, seeing them as opaque binary blobs. But the truth is, the initial steps of core dump analysis are surprisingly straightforward. The simplest and most immediate way to begin understanding a crash is by examining the backtrace. A backtrace (or stack trace) provides a chronological list of function calls that led up to the crash, showing you the exact execution path and, with proper debug symbols, the specific line of code where the program met its untimely end. It's like unwinding a tangled thread, revealing the sequence of events.

As we delve deeper, we'll discover that the most important skill for truly understanding a core dump, especially when source-level information is scarce, is the ability to interpret assembly language. While intimidating at first glance, disassembling the crash site allows us to peek directly into the machine's execution flow. It unveils the precise instructions the CPU was executing, the values in its registers, and how arguments were passed. This low-level insight is often the only way to pinpoint subtle memory errors, misaligned accesses, or incorrect function parameters that lead to catastrophic failures like segmentation faults. Through practical examples, we'll see how dissecting a few lines of assembly can transform a seemingly unsolvable crash into a clear bug report, providing the exact context needed for a fix.


# Some GDB commands

__Launch the GDB with core file :__

```java
gdb  APPLICATION APPLICATION.core
```

__Config the location for the GDB to search the libraries:__
```java
set solib-search-path PATH1:PATH2…  // (gdb) set solib-search-path /path/to/my_symbols_folder:/path/to/my_symbols_folder/lib
set sysroot /path/to/my_symbols_folder
or 
add-symbol-file /path/to/my_symbols_folder/mylib.so <address_where_it_was_loaded>
```

__List the shared library information, we can see what symbol files are not loaded:__
```java
info sharedlibrary
```

__Show the backtrace__
```java
bt
```

__List the CPU registers__
```java
info registers 

info registers x0 x1 x2
```


__Print the value of a CPU register__
```java

# print CPU register x0, x1, x2
print/x $x0
print/x $x1
print $x2

```

__inspect information about program variables__
```java
info variable A_VARIABLE
```

__Disassemble the machine code of the current function or a specific address range__
```java

#disassemble a function
disassemble FUNC_NAME

#disassemble a address. 
disassemble 0x51bfb97a98, +128
```

__List all the threads__

```java
info threads
```

__Examine memory at a given address, useful for inspecting raw data or corrupted memory__
```java
x /<count><format><size> <address>: 
    count: Number of units to display.
    format: x (hex), d (decimal), u (unsigned decimal), o (octal), t (binary), s (string), i (instruction), a (address).
    size: b (byte), h (halfword/2 bytes), w (word/4 bytes), g (giant word/8 bytes).

(gdb) x /20xw 0xdeadbeef   # Examine 20 words in hex starting at 0xdeadbeef
(gdb) x /s some_char_pointer # Examine a string at pointer
(gdb) x/10x 0x2870c55fc8 


dump memory source_mem_dump.bin 0x2870c55fc8 0x2870c55fc8 + 0x1c502e0

```

__To check if a file contains debug symbol or not :__
```java
readelf -S mylib.so | grep '\.debug_'
objdump -g mylib.so
```

__Check for Undefined Symbols (Dynamic Dependencies)  -- look for memcpy as a example__
Use readelf -Ws (or readelf --dyn-syms) to list dynamic symbols, and look for memcpy

```java
readelf -s /path/to/your_library.so | grep memcpy

UND (Undefined): If you see memcpy listed with UND in the Ndx column, it means your library calls memcpy, and it expects the dynamic linker to provide the implementation at runtime. This is the most common and definitive sign.

Num:    Value          Size Type    Bind   Vis      Ndx Name
...
123:    0000000000000000     0 FUNC    GLOBAL  DEFAULT  UND memcpy
```

No memcpy in UND list: If memcpy is not in the UND list, it means one of two things:
1. The library doesn't call memcpy at all.
2. The memcpy implementation was statically linked into the dynamic library itself

__Using objdump (Indirectly) to See Linked Libraries__


objdump -p your_executable_or_library.so | grep 'NEEDED\|RPATH\|RUNPATH'
    objdump -p (or --private-headers): Displays all file-format-specific information, including the .dynamic section.
    grep 'NEEDED\|RPATH\|RUNPATH': Filters the output to show lines indicating required libraries (NEEDED), and paths for runtime search (RPATH, RUNPATH).
    
```java
Example Output:

  NEEDED               libc.so.6
  NEEDED               libm.so.6
  RPATH                /opt/my_app/lib
```


# A practice

From backtrace the address 0x00000051beeb9308 is the crash place. So we disassemble the section of code like below:

```java
   0x00000051beeb92ec <+404>:   bl      0x51bee71340 <xxx_mmap@plt>
   0x00000051beeb92f0 <+408>:   cbz     x0, 0x51beeb9540 <MngrInit+1000>
   0x00000051beeb92f4 <+412>:   mov     w2, w19
   0x00000051beeb92f8 <+416>:   mov     x1, x0
   0x00000051beeb92fc <+420>:   adrp    x19, 0x51bfb97000 <xxx_animData+102304>
   0x00000051beeb9300 <+424>:   add     x0, x19, #0xa98
   0x00000051beeb9304 <+428>:   bl      0x51bee6c680 <memcpy@plt>
   0x00000051beeb9308 <+432>:   mov     x0, x26
   0x00000051beeb930c <+436>:   bl      0x51bee6b760 <xxx_fclose@plt>
   0x00000051beeb9310 <+440>:   add     x5, x22, #0xac8
   0x00000051beeb9314 <+444>:   add     x19, x19, #0xa98
   0x00000051beeb9318 <+448>:   mov     x0, x5
   0x00000051beeb931c <+452>:   str     x5, [sp, #96]
   0x00000051beeb9320 <+456>:   add     x25, x24, #0xf18
   0x00000051beeb9324 <+460>:   add     x26, x23, #0xaf8
   0x00000051beeb9328 <+464>:   bl      0x51bee6e400 <msgOutput@plt>
   0x00000051beeb932c <+468>:   add     x27, x21, #0xa28
```

__Get the address of xxx_animData:__
```java
(gdb) info variable sg_animData
All variables matching regular expression "xxx_animData":
 
Non-debugging symbols:
0x00000051bfb7e060  xxx_animData

```

__Check the input variable of memcpy__
```java
x0 is memcpy destination 
x1 is memcpy source
x2 is memcpy size
(gdb) info registers x0 x1 x2
x0             0x51bfb97a98        351108954776
x1             0x2870c55fc8        173690675144
x2             0x1c502e0           29688544

```

__memcpy destination:__
```java
x0 = 0x51bfb97000 + 0xa98 = 0x51bfb97a98  
```
 
0x51bfb97a98 is start of FileBuffer. which the size is 0x1c6f960.  it is larger than the copy size.  so the destination buffer is safe.

```java
0000000000ee5a98 0x1c6f960 OBJECT  LOCAL  DEFAULT   20 langFileBuffer
memcpy size:
x2 (Size): 0x1c502e0
```

__Try to access the source memory address:__
```java
(gdb) dump memory source_mem_dump.bin 0x2870c55fc8 0x2870c55fc8 + 0x1c502e0
Cannot access memory at address 0x2870c56000
 
The Source Memory Region is Invalid or Unreadable:  The memory region that memcpy was trying to read from (0x2870c55fc8 for 28.3 MB) is not entirely valid or readable.
```




