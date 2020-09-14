---
layout: post
title:  "Re: Brainfuck JIT compiler (Rust version)"
date:   2020-09-13 19:00:00 +0000
categories: blog
---

## Intro

The first version of the JIT compiler (see [previous post](https://pablojorge.github.io/blog/2020/07/27/bf-jit-compiler-in-cpp.html)) can be slightly improved in a couple of ways:

 * Instead of using `RDI` to hold the data pointer, we can use the `RSI` register, as that's the one already used in the `read` and `write` syscalls to point to the memory buffer. This way we don't have to explicitly setup that param, and we can skip preserving the original value of `RDI` (used to specify the `fd` in read/write)
 * Moving the pointer and the data pointed to can be done with immediate operands, there's no need to use an auxiliary register

Instead of doing those changes to the original compiler, or creating another C++ version, I decided to apply these improvements in a separate implementation, in Rust.

Just as the C++ version, at a high level, this is what we need:

- to parse the BF source into an "expressions" tree
- transform the tree into opcodes/operands in memory
- make the memory where the opcodes reside 'executable' and jump to it

The full source can be found [here](https://github.com/pablojorge/brainfuck/blob/master/rust/src/bin/jit.rs). Tested on MacOS Catalina, with Rust 1.48.0-nightly (2020-09-09).

## Transforming the BF source into an expression tree

In the original [Rust interpreter](https://github.com/pablojorge/brainfuck/blob/master/rust/src/lib.rs) we already have what we need. We can use the `tokenize`, `parse` and `optimize` functions to transform the BF source into an optimized `std::Vec<Expression>`.

For a simple "cat"-like program, that just copies input into output in and endless loop like `+[>,.<]`, this is the resulting tree:

```rust
[
    IncValue(1),
    Loop([
        MoveForward(1),
        InputValue,
        OutputValue,
        MoveBack(1)
    ])
]
```

## Generating opcodes

We'll make use of the [assembler crate](https://docs.rs/assembler/0.10.1/assembler/), which will assist in the management of the executable memory region and tracking of jumps.

As we said in the intro, we'll use `RSI` as the main memory pointer.

Data manipulation primitives are super simple (we use 32 bits for memory cells, so we have to move by increments of 4 bytes):

```rust
fn compile(expressions: &Vec<bf::Expression>, stream: &mut InstructionStream) {
    for expression in expressions {
        match expression {
            // 000000000000000c forward:
            //        c: 48 81 c6 80 00 00 00          addq    $128, %rsi
            &bf::Expression::MoveForward(n) => {
                stream.emit_bytes(b"\x48\x81\xc6");
                stream.emit_double_word((n*4).try_into().unwrap());
            },
            // 0000000000000013 backward:
            //       13: 48 81 ee 80 00 00 00          subq    $128, %rsi
            &bf::Expression::MoveBack(n) => {
                stream.emit_bytes(b"\x48\x81\xee");
                stream.emit_double_word((n*4).try_into().unwrap());
            },
            // 0000000000000000 increment:
            //        0: 81 06 80 00 00 00             addl    $128, (%rsi)
            &bf::Expression::IncValue(n) => {
                stream.emit_bytes(b"\x81\x06");
                stream.emit_double_word(n);
            },
            // 0000000000000006 decrement:
            //        6: 81 2e 80 00 00 00             subl    $128, (%rsi)
            &bf::Expression::DecValue(n) => {
                stream.emit_bytes(b"\x81\x2e");
                stream.emit_double_word(n);
            },
```

Input and output are now shorter, thanks to the usage of `RSI`. We only need to set the fd and size params, and the syscall number of course:

```rust
            // 000000000000001a read:
            //       1a: 48 c7 c0 03 00 00 02          movq    $33554435, %rax
            //       21: 48 c7 c7 00 00 00 00          movq    $0, %rdi
            //       28: 48 c7 c2 01 00 00 00          movq    $1, %rdx
            //       2f: 0f 05                         syscall
             bf::Expression::InputValue => {
                stream.emit_bytes(b"\x48\xc7\xc0");
                stream.emit_double_word(0x02000003);
                stream.emit_bytes(b"\x48\xc7\xc7\x00\x00\x00\x00");
                stream.emit_bytes(b"\x48\xc7\xc2\x01\x00\x00\x00");
                stream.emit_bytes(b"\x0f\x05");
             },
            // 0000000000000031 write:
            //       31: 48 c7 c0 04 00 00 02          movq    $33554436, %rax
            //       38: 48 c7 c7 01 00 00 00          movq    $1, %rdi
            //       3f: 48 c7 c2 01 00 00 00          movq    $1, %rdx
            //       46: 0f 05                         syscall
             bf::Expression::OutputValue => {
                stream.emit_bytes(b"\x48\xc7\xc0");
                stream.emit_double_word(0x02000004);
                stream.emit_bytes(b"\x48\xc7\xc7\x01\x00\x00\x00");
                stream.emit_bytes(b"\x48\xc7\xc2\x01\x00\x00\x00");
                stream.emit_bytes(b"\x0f\x05");
             },
```

Loops are a bit simpler thanks to the "labels" feature of the assembler crate. When the loop starts, we want to jump to the first instruction after it if the current memory cell is 0. If we enter the loop, on each iteration we jump back to the beginning of if the cell is != 0:

```rust
             bf::Expression::Loop(sub_exp) => {
                let loop_start = stream.create_label();
                let post_loop = stream.create_label();

                //   48: 83 3e 00                      cmpl    $0, (%rsi)
                //   4b: 0f 84 00 00 00 00             je  0 <loop_end>
                stream.emit_bytes(b"\x83\x3e\x00");
                stream.jz_Label_1(post_loop);       // --*
                stream.attach_label(loop_start);    // <-|-*
                                                    //   | |
                compile(sub_exp, stream);           //   | |
                                                    //   | |
                //   51: 83 3e 00  cmpl $0, (%rsi)  //   | |
                //   54: 0f 85 00 00 00 00  jne 0   //   | |
                stream.emit_bytes(b"\x83\x3e\x00"); //   | |
                stream.jnz_Label_1(loop_start);     // --|-*
                stream.attach_label(post_loop);     // <-*
            }
        }
    }
}
```

## Preparing for execution

Now that we know how many bytes each expression will take when transformed into opcodes, we can calculate how much we need to allocate to store the program. Additionally, we also need the total number of loops to reserve the space for all the jumps we'll make:

```rust
fn run(expressions: &Vec<bf::Expression>) {
    let stats = bf::stats(expressions);

    let mem_size = opcodes_size(&stats) 
                    + 1  // retq
                    + 8; // stream buffer

    let mut memory_map = ExecutableAnonymousMemoryMap::new(mem_size,
                                                           false,
                                                           false).unwrap();

    let mut instruction_stream = memory_map.instruction_stream(
        &InstructionStreamHints {
            number_of_labels: stats.loop_count * 2,
            number_of_8_bit_jumps: 0,
            number_of_32_bit_jumps: stats.loop_count * 2,
            number_of_emitted_labels: 0
        }
    );

    let function_pointer = instruction_stream.nullary_function_pointer::<()>();
```

We then generate all the opcodes in the `instruction_stream` (including the final `ret`), and use the `finish()` method to resolve the jumps and make memory executable (by doing the exact same call to `mmap()` as we do manually in the C++ version):

```rust
    compile(expressions, &mut instruction_stream);

    // 000000000000005b finish:
    //       5b: c3                            retq
    instruction_stream.emit_byte(0xc3);

    instruction_stream.finish();
```

## Executing

The final step is to setup a `u32` array as the working memory of the program, point it with the `RSI` register and make the processor jump to the generated code. Similarly to what we do in the C++ version:


```rust
    let memory = [0u32; 30000];

    unsafe {
        asm!(
            "movq {mem}, %rsi",
            mem = in (reg) &memory,
            options(att_syntax)
        );

        function_pointer();
    }
}
```

This is the generated code for the `+[>,.<]` program:

```
(lldb) disas -b -s $1 -c 40
    0x10019e000: 81 06 01 00 00 00     addl   $0x1, (%rsi)     ; +
    0x10019e006: 83 3e 00              cmpl   $0x0, (%rsi)     ; [
    0x10019e009: 0f 84 45 00 00 00     je     0x10019e054      
    0x10019e00f: 48 81 c6 04 00 00 00  addq   $0x4, %rsi       ; >
    0x10019e016: 48 c7 c0 03 00 00 02  movq   $0x2000003, %rax ; ,
    0x10019e01d: 48 c7 c7 00 00 00 00  movq   $0x0, %rdi
    0x10019e024: 48 c7 c2 01 00 00 00  movq   $0x1, %rdx
    0x10019e02b: 0f 05                 syscall
    0x10019e02d: 48 c7 c0 04 00 00 02  movq   $0x2000004, %rax ; .
    0x10019e034: 48 c7 c7 01 00 00 00  movq   $0x1, %rdi
    0x10019e03b: 48 c7 c2 01 00 00 00  movq   $0x1, %rdx
    0x10019e042: 0f 05                 syscall
    0x10019e044: 48 81 ee 04 00 00 00  subq   $0x4, %rsi       ; <
    0x10019e04b: 83 3e 00              cmpl   $0x0, (%rsi)     ; [
    0x10019e04e: 0f 85 bb ff ff ff     jne    0x10019e00f
    0x10019e054: c3                    retq
```

The _same_ program, generated with the C++ JIT version looks like this:

```
(lldb) disas -b -s $1 -c 40
    0x100500000: 48 c7 c0 01 00 00 00  movq   $0x1, %rax       ; +
    0x100500007: 01 07                 addl   %eax, (%rdi)
    0x100500009: 83 3f 00              cmpl   $0x0, (%rdi)     ; [
    0x10050000c: 0f 84 55 00 00 00     je     0x100500067
    0x100500012: 48 c7 c0 04 00 00 00  movq   $0x4, %rax       ; >
    0x100500019: 48 01 c7              addq   %rax, %rdi
    0x10050001c: 57                    pushq  %rdi             ; ,
    0x10050001d: 48 c7 c0 03 00 00 02  movq   $0x2000003, %rax
    0x100500024: 48 89 fe              movq   %rdi, %rsi
    0x100500027: 48 c7 c7 00 00 00 00  movq   $0x0, %rdi
    0x10050002e: 48 c7 c2 01 00 00 00  movq   $0x1, %rdx
    0x100500035: 0f 05                 syscall
    0x100500037: 5f                    popq   %rdi             ; .
    0x100500038: 57                    pushq  %rdi
    0x100500039: 48 c7 c0 04 00 00 02  movq   $0x2000004, %rax
    0x100500040: 48 89 fe              movq   %rdi, %rsi
    0x100500043: 48 c7 c7 01 00 00 00  movq   $0x1, %rdi
    0x10050004a: 48 c7 c2 01 00 00 00  movq   $0x1, %rdx
    0x100500051: 0f 05                 syscall
    0x100500053: 5f                    popq   %rdi
    0x100500054: 48 c7 c0 04 00 00 00  movq   $0x4, %rax       ; <
    0x10050005b: 48 29 c7              subq   %rax, %rdi
    0x10050005e: 83 3f 00              cmpl   $0x0, (%rdi)     ; [
    0x100500061: 0f 85 ab ff ff ff     jne    0x100500012
    0x100500067: c3                    retq
```

## Benchmark

A shorter code produces a slightly faster execution time, compared to the original JIT implementation:

|                | Primes up to 200| Mandelbrot|
| --------------:| ---------------:| ---------:|
|               C|           3872ms|    14850ms|
|            C++ |           2728ms|     5152ms|
|        C++-JIT |            647ms|     1173ms|
|            Rust|           4176ms|     7596ms|
|     **RustJIT**|            645ms|     1073ms|

Stay tuned for future updates!