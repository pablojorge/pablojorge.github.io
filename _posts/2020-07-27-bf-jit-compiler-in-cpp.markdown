---
layout: post
title:  "Brainfuck JIT compiler in C++"
date:   2020-07-27 19:00:00 +0000
categories: blog
---

## Intro

As part of my [collection of Brainfuck interpreters](https://github.com/pablojorge/brainfuck) (which includes, among others, an interpreter written in [pure ASM](https://github.com/pablojorge/brainfuck/blob/master/asm), and others written in [C++](https://github.com/pablojorge/brainfuck/tree/master/cpp)), I wanted to create one able to generate machine code on the fly, and execute it. Some sort of "handmade" JIT compiler, without relying on any library to achieve it.

[Brainfuck](https://en.wikipedia.org/wiki/Brainfuck) is a simple language that contains just a few operators: to read/write a single byte, move the data pointer, increment/decrement and loop. It's simple enough to easily create machine code equivalents for each, glue them together and execute the final result.

The goal is to convert any Brainfuck program such as the following one into executable x86_64 code, in runtime, in C++:

```
+++ [> ++++ <-]
```

(This "multiplies" 3 by 4, and produces a "0" in the first "cell" and a "12" in the second one)

Following the choice made for the C++ version, the memory will be composed of 32 bits cells.

At a high level, this is what's needed:

- parse the BF source and obtain the "expressions" tree
- transform the tree into opcodes/operands in memory
- make the memory where the opcodes reside 'executable' (otherwise, attempting to execute the code will fail, as regular programs _shouldn't_ be executing code outside of the code sections of the process)
- jump to the generated code

The full source is [here](https://github.com/pablojorge/brainfuck/blob/master/cpp/brainfuck-jit.cpp). It works on MacOS, and was tested with Clang 11.0.3 and GCC 10.1.0.

## Parsing the BF source

For this part, we can just reuse the existing [C++ parser](https://github.com/pablojorge/brainfuck/blob/master/cpp/brainfuck.h), which includes an extra benefit: "compressing" contiguous identical operators.

In the case of the sample BF program (`+++ [> ++++ <-]`), the resulting tree looks like this:

```
[
    Increment(3),
    loop [
        Forward(1),
        Increment(4),
        Backward(1),
        Decrement(1)
    ]
]
```

This simple optimization avoids executing the same operation N times, executing it just once using N as an 'operand'.

In a style inspired by the "Visitor" pattern, we'll traverse the tree and emit opcodes for each.

## Generating x86_64 opcodes

All the state we need to keep is the pointer to the current position in the main "memory" that BF programs use. The memory will be a `uint32_t` buffer, and we'll use the RDI register for this purpose.

This means that for each BF operator, we need to generate opcodes that update it/use it accordingly.

As a reference, this is the C-equivalent for each BF operator, where `ptr` is the pointer to the program memory -what we'll keep in RDI- (_adapted from the [BF to C translator](https://github.com/pablojorge/brainfuck/blob/master/haskell/bf2c.hs)_)

```
    '>' = "++ptr;"            # forward
    '<' = "--ptr;"            # backward
    '+' = "++(*ptr);"         # increment
    '-' = "--(*ptr);"         # decrement
    '.' = "putchar(*ptr);"    # input
    ',' = "*ptr = getchar();" # output
    '[' = "while(*ptr) {"     # loop start
    ']' = "}"                 # loop end
```

### Moving the pointer forward/backward

These are the simplest ones: we need to increment/decrement the RDI register by N. We can use the RAX register as auxiliary:

```
  movq    N, %rax
  addq    %rax, %rdi # or subq
```

Notice we're using full quad-words, since we're working with pointers and of course memory addressing utilizes all 64 bits.

The equivalent opcodes (for `addq`) are:

```
      48 c7 c0 01 00 00 00         	movq	$1, %rax
      48 01 c7                     	addq	%rax, %rdi
```

In this case we're setting 1 to RAX (notice this is x86, so all values all little endian)

We can generate the opcodes on the fly like this:

```c++
    virtual void visit(const Forward& fwd) {
        buffer_.writes((uint8_t*)"\x48\xc7\xc0", 3);
        buffer_.writel(fwd.offset()*4);
        buffer_.writes((uint8_t*)"\x48\x01\xc7", 3);
    }
```

`buffer_`'s class provides methods to write single bytes, long words (32 bits) or arbitrary-length byte sequences.

Also notice we move the pointer N places by 4, because each cell is 4-bytes long.

### Increment/decrement the data pointed-to

These are similar to the previous ones, but dereferencing the pointer. This is what we want to do:

```
  movq    N, %rax
  addl    %eax, (%rdi) # or subl
```

Here, since we're operating on the _data_, the pointer is still 64 bits, but the operand is 32 (EAX). Opcodes + generation:

```c++
    virtual void visit(const Increment& inc) {
        // 0000000000000000 increment:
        //        0: 48 c7 c0 01 00 00 00          movq    $1, %rax
        //        7: 01 07                         addl    %eax, (%rdi)
        buffer_.writes((uint8_t*)"\x48\xc7\xc0", 3);
        buffer_.writel(inc.offset());
        buffer_.writes((uint8_t*)"\x01\x07", 2);        
    }
```

### Input/output

Moving the pointer around and modifying data in memory are fine, but we need to interact with the outside world as well!

Now it starts to get interesting, as we'll need to execute syscalls to read/write from/to stdin/stdout. This is in fact easier than using the libc for example, as we'll talk directly with the kernel.

Using the original [asm interpreter](https://github.com/pablojorge/brainfuck/blob/master/asm/brainfuck.s) as reference, this is how the `read`/`write` syscalls can be invoked in MacOS:

```
 # ssize_t read(int fildes, void *buf, size_t nbyte);
 #              rdi         rsi        rdx
 movq $0, %rdi
 movq $1, %rsi
 movq $2, %rdx
 movq $$0x2000003, %rax # SYS_read (3)
 syscall

 # ssize_t write(int fildes, const void *buf, size_t nbyte);
 #               rdi         rsi              rdx
 movq $0, %rdi
 movq $1, %rsi
 movq $2, %rdx
 movq $$0x2000004, %rax # SYS_open (4)
 syscall
```

The file descriptor goes into RDI, the input/output buffer goes in RSI, the count in RDX and finally the syscall descriptor in RAX. Then we execute the `syscall` instruction (what was done with `int 0x80` in the past)

The problem is that we need to use RDI, but we use it to hold the data pointer. We can easily solve this by pushing RDI to the stack and restoring it after we're done.

To write a single byte to stdout, this is what we need: (`read` is similar, only the syscall descriptor -`0x2000003`- and the fd -`0, stdout`- change):

```
  pushq   %rdi              # save RDI
  movq    $0x02000004, %rax # SYS_write
  movq    %rdi, %rsi        # set RDI as the 'buf'
  movq    $1, %rdi          # fd: stdout
  movq    $1, %rdx          # len: 1
  syscall                   # execute the call
  popq    %rdi              # restore RDI
```

Opcodes + generation:

```c++
    virtual void visit(const Output&) {
        // 0000000000000042 write:
        //       42: 57                            pushq   %rdi
        //       43: 48 c7 c0 04 00 00 02          movq    $33554436, %rax
        //       4a: 48 89 fe                      movq    %rdi, %rsi
        //       4d: 48 c7 c7 01 00 00 00          movq    $1, %rdi
        //       54: 48 c7 c2 01 00 00 00          movq    $1, %rdx
        //       5b: 0f 05                         syscall
        //       5d: 5f                            popq    %rdi

        buffer_.writeb(0x57);
        buffer_.writes((uint8_t*)"\x48\xc7\xc0", 3);
        buffer_.writel(0x02000004);
        buffer_.writes((uint8_t*)"\x48\x89\xfe", 3);
        buffer_.writes((uint8_t*)"\x48\xc7\xc7\x01\x00\x00\x00", 7);
        buffer_.writes((uint8_t*)"\x48\xc7\xc2\x01\x00\x00\x00", 7);
        buffer_.writes((uint8_t*)"\x0f\x05", 2);
        buffer_.writeb(0x5f);
    }
```

### Loops

Finally, loops brings a bit more of a challenge. Continuing with the same example, we need to replicate these semantics:

```c
1. +++  = *ptr+=3;
2. [    = while(*ptr) {
3. >    =   ptr++;
4. ++++ =   *ptr+=4;
5. <    =   ptr--;
6. -    =   *ptr-=1;
7. ]    = }
8.      = (END)
```

In line 2 we want to enter the loop only if the cell pointed by `ptr` is > 0, otherwise we want to skip it and jump to line 8.

In line 7, we have two options:
 - unconditionally jump back to 2 (which, in case the condition is not true anymore we'll perform two jumps: 7->2, 2->8)
 - check the condition and continue in case it's not true, or jump back to 3 (to enter the loop again) in case it is

Option 2 is a bit more efficient, since it avoids some jumps. To compare the pointed data and jump, we need to:

```
loop_start:
  cmpl    $0, (%rdi)
  je      N          # if (%rdi) == 0, jump out of the loop

loop_end:
  cmpl    $0, (%rdi)
  jne     N          # if (%rdi) != 0, jump back to the 1st instruction
```

The challenge is to set N in each case, the appropriate jump distance. Mainly for the first jump, as we don't know where the loop end will be.

The technique is to put a placeholder, push the current position to a stack, continue with the generation until we reach the end of the loop, calculate the distances and fill both jumps.

```
1. +++  = *ptr+=3;      = movq 3, rax; addl eax, (rdi);
2. [    = while(*ptr) { = cmp 0, (rdi); je [TBD1];
3. >    =   ptr++;      = movq 4, rax; addq rax, rdi;
4. ++++ =   *ptr+=4;    = movq 4, rax; addl eax, (rdi);
5. <    =   ptr--;      = movq 4, rax; subq rax, rdi;
6. -    =   *ptr-=1;    = movq 1, rax; subl eax, (rdi);
7. ]    = }             = cmp 0, (rdi); jne [TBD2];
8.      = (END)         = ret
```

We want `[TBD1]` to become '8' (or, in fact, the distance between 8 and the end of 2, since jumps in x86_64 can only be relative, not absolute), and we want `[TBD2]` to be the distance between the end of 7 and 3.

This is the full solution:

```c++
    virtual void visit(const Loop& loop) {
        // 000000000000005e loop_start:
        //       5e: 83 3f 00                      cmpl    $0, (%rdi)
        //       61: 0f 84 00 00 00 00             je  0
        buffer_.writes((uint8_t*)"\x83\x3f\x00", 3);
        buffer_.writes((uint8_t*)"\x0f\x84", 2);
        buffer_.writel(0); // reserve 4 bytes

        // Save current position:
        uint8_t *after_loop_start = buffer_.get_ptr();

        // Recurse into subexpressions:
        for(const auto &child: loop.children()) {
            child->accept(*this);
        }

        // 0000000000000067 loop_end:
        //       67: 83 3f 00                      cmpl    $0, (%rdi)
        //       6a: 0f 85 00 00 00 00             jne 0
        buffer_.writes((uint8_t*)"\x83\x3f\x00", 3);
        buffer_.writes((uint8_t*)"\x0f\x85", 2);
        // Calculate how much to jump back (consider the 4 bytes
        // of the operand itself):
        uint32_t jump_back = after_loop_start - buffer_.get_ptr() - 4;
        // Append the distance to the jump:
        buffer_.writel(jump_back);

        // Now calculate how much to jump forward in case we want
        // to skip the loop, to fill the pending jump:
        uint8_t *after_loop_end = buffer_.get_ptr();
        uint32_t jump_fwd = after_loop_end - after_loop_start;

        // Go back to the original position (-4), fill the pending
        // jump distance and get back to the the current pos:
        buffer_.set_ptr(after_loop_start - 4);
        buffer_.writel(jump_fwd);
        buffer_.set_ptr(after_loop_end);
    }
```

## Marking the memory as executable

For security reasons, dynamic memory is not executable by default. That would prevent our awesome generated code from running! To avoid this problem, we need to mark the pages where the code resides as executable.

We can either allocate some pages and mark them as executable right away, or even better, allocate them and update them later, after the code has been generated. This a good practice, as it guarantees that the pages are not _writable_ and _executable_ at the same time at any moment.

Since we can't mark _any_ position as executable, we need to mark full pages, we still need to use `mmap()` to allocate, since the memory allocated will be on a page boundary.

Here we initialize the buffer that will contain the generated code in read/write mode:

```c++
    ExecutableBuffer(size_t size) {
        size_ = size;
        buf_ = (uint8_t*) mmap(
            0,
            size_,
            PROT_READ | PROT_WRITE,
            MAP_PRIVATE | MAP_ANONYMOUS,
            -1,
            0
        );
        if (buf_ == MAP_FAILED) {
            perror("mmap");
            exit(1);
        }
        ptr_ = buf_;
        memset(ptr_, 0x00, size);
    }
```

And when we're done, we switch to read/exec:

```c++
    void make_executable() {
        if (mprotect(buf_, size_, PROT_READ | PROT_EXEC) == -1) {
            perror("mprotect");
            exit(1);
        }
    }
```

## Jump to the generated code

Once we have all generated code in a memory region suitable for execution, we need to jump from our C++ program to it. The first step is to initialize the RDI register, so that it points to a valid `uint32_t` buffer. We can do that with a bit of inline assembly. Immediately after that, we need to treat the pointer to the generated code as a pointer to a function, and invoke it. That's it. That will make the processor start executing the opcodes we generated on the fly:

```c++
    void run() {
    	// define buffer
        uint32_t memory[30000];

        // set its address in RDI
        asm("movq %0, %%rdi\n":
            : "r"(&memory)
            : "%rdi");

        // enter our code!
        ((void (*)()) program)();
    }
```

Using LLDB to inspect the memory and the generated code:

Memory before execution:

```
$ lldb -s lldb-commands.txt ./brainfuck-jit -- test.bf
[...]
(lldb) b JITProgram::run
Breakpoint 1: where = brainfuck-jit`main + 1142 [inlined] JITProgram::run() at brainfuck-jit.cpp:288, address = 0x00000001000021f6
(lldb) r
[...]
(lldb) p &memory
(uint32_t (*)[30000]) $0 = 0x00007ffeefbe2230
(lldb) x $0
0x7ffeefbe2230: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
0x7ffeefbe2240: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
```

Generated code:

```
(lldb) p buf_.buf_
(uint8_t *) $1 = 0x0000000100500000 "H��\x03"
(lldb) x -c 80 $1
0x100500000: 48 c7 c0 03 00 00 00 01 07 83 3f 00 0f 84 2f 00  H��.......?.../.
0x100500010: 00 00 48 c7 c0 04 00 00 00 48 01 c7 48 c7 c0 04  ..H��....H.�H��.
0x100500020: 00 00 00 01 07 48 c7 c0 04 00 00 00 48 29 c7 48  .....H��....H)�H
0x100500030: c7 c0 01 00 00 00 29 07 83 3f 00 0f 85 d1 ff ff  ��....)..?...���
0x100500040: ff c3 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ��..............
(lldb) disas -b -s $1 -c 40
    0x100500000: 48 c7 c0 03 00 00 00  movq   $0x3, %rax
    0x100500007: 01 07                 addl   %eax, (%rdi)
    0x100500009: 83 3f 00              cmpl   $0x0, (%rdi)
    0x10050000c: 0f 84 2f 00 00 00     je     0x100500041
    0x100500012: 48 c7 c0 04 00 00 00  movq   $0x4, %rax
    0x100500019: 48 01 c7              addq   %rax, %rdi
    0x10050001c: 48 c7 c0 04 00 00 00  movq   $0x4, %rax
    0x100500023: 01 07                 addl   %eax, (%rdi)
    0x100500025: 48 c7 c0 04 00 00 00  movq   $0x4, %rax
    0x10050002c: 48 29 c7              subq   %rax, %rdi
    0x10050002f: 48 c7 c0 01 00 00 00  movq   $0x1, %rax
    0x100500036: 29 07                 subl   %eax, (%rdi)
    0x100500038: 83 3f 00              cmpl   $0x0, (%rdi)
    0x10050003b: 0f 85 d1 ff ff ff     jne    0x100500012
    0x100500041: c3                    retq
```

Memory AFTER execution:

```
(lldb) x $0
0x7ffeefbe2230: 00 00 00 00 0c 00 00 00 00 00 00 00 00 00 00 00  ................
0x7ffeefbe2240: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
```

The first 4 bytes (the 1st cell) are `00 00 00 00`, and the next 4 (2nd cell) are `0c 00 00 00`, `0x0c` in little endian (12), just as expected.

## Benchmark

Heavy BF programs like the Mandelbrot Set generator, or the prime numbers generator run much faster compared to the optimized interpreters:

|             | Primes up to 200| Mandelbrot Set|
| -----------:| ---------------:| -------------:|
|            C|           3878ms|        14619ms|
|         Rust|           4123ms|         7270ms|
|         C++ |           2827ms|         5272ms|
| **C++-JIT** |      _**644ms**_|   _**1157ms**_|
