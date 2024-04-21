+++
title = "Writing an x86 Compiler for Brainfuck"
+++

Brainfuck is a programming language which, as the name might suggest, was designed to fuck with your brain. Brainfuck programs consist of only 8 characters: `+-><[],.`. All other characters in a `.bf` file are ignored by the language. Here's what a simple "hello, world" program looks like in BF:

```brainfuck
++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.>---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.
```

BF has two pointers, one for instructions and one for data. The data pointer points somewhere inside a contiguous block of memory, and the instruction pointer points to the instruction currently being executed.

The BF operators have these semantics:

 - `>`: increment the data pointer by 1
 - `<`: decrement the data pointer by 1
 - `+`: increment the value pointed to by the data pointer by 1
 - `-`: decrement the value pointed to by the data pointer by 1
 - `.`: print the value pointed to by the data pointer
 - `,`: read a single byte from stdin and store it at the position pointed to by the data pointer
 - `[`: if the value pointed to by the data pointer is 0, then jump to the instruction after a matching `]`. otherwise continue
 - `]`: if the value pointed to by the data pointer is _not_ 0, then jump to the instruction after a matching `[`. otherwise continue

These 8 operators are technically enough to make BF turing complete, though doing even simple tasks involves a massive number of characters and it quickly becomes hard to reason about programs.

BF lacks types, variables, functions, and even idioms for doing things like multiplication, which makes it very hard to write. _But_ conversely, this makes it very easy to write an implementation. We can skip things like parsing and typechecking, and just go straight to compilation. 

## Writing a Brainfuck Interpreter

Interpreters are generally easier to write than compilers. You don't need to know assembly or any complex compiler algorithms — you just need to know the semantics of your operation and you can implement it in whatever language you want. We'll start by writing a simple BF interpreter to get a sense of BF's semantics, and also so that we have something to compare our compiler to.

The first thing we need to do for our interpreter is initialize the data and the instruction buffers. This is actually not that dissimilar to how your operating system loads executable files. 

Our data buffer is just an array of 8-bit bytes called "cells." In brainfuck we have to provide the language with at least 30,000 cells, though we could go much higher than that if we wanted to.

```rust
let mut data_buffer = vec![0_u8; 30_000];
```

For the instruction buffer, let's start with just hardcoding the instructions, and then later, if we want to, we can support things like reading from files.

```rust
// hello, world
let instruction_buffer = b"++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.>---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.";
```

We also need to initialize our data and instruction pointers.

```rust
let mut data_ptr = 0_usize;
let mut instruction_ptr = 0_usize;
```

The concept of an instruction pointer might sound similar if you already know a bit of assembly. In x86 this is the rip/eip/ip register.

Now that we have our buffers and pointers initialized, we can start writing our interpreter loop. Our program executes until the instruction pointer reaches the end of our instructions.

```rust
while instruction_ptr < instruction_buffer.len() {
    // ...
}
```

Then we can start implementing the functionality inside our loop,

```rust
match instruction_buffer[instruction_ptr] {
    b'>' => todo!(),
    b'<' => todo!(),
    b'+' => todo!(),
    b'-' => todo!(),
    b'.' => todo!(),
    b',' => todo!(),
    b'[' => todo!(),
    b']' => todo!(),
    _ => {},
}

// after executing each instruction, increment the instruction pointer to the next instruction
instruction_ptr += 1;
```

The first 4 operators are pretty simple.

```rust
b'>' => data_ptr += 1,
b'<' => data_ptr -= 1,
b'+' => data_buffer[data_ptr] += 1,
b'-' => data_buffer[data_ptr] -= 1,
```

In the case of `+` and `-`, we actually have a small issue. In rust, integer overflow is defined to panic in debug builds. Brainfuck is fine with overflow, and even expects it. To avoid crashes in debug mode, we need our addition and subtraction to be explicitly wrapping.

```rust
b'+' => data_buffer[data_ptr] = data_buffer[data_ptr].wrapping_add(1),
b'-' => data_buffer[data_ptr] = data_buffer[data_ptr].wrapping_sub(1),
```

Great. Now we can implement `.` and `,`.

```rust
b'.' => print!("{}", char::from(data_buffer[data_ptr])),
b',' => {
    use std::io::Read;
    let mut byte = [0];
    let mut stdin = std::io::stdin();
    // read single byte
    stdin.read_exact(&mut byte).unwrap();
    // drop rest of line
    stdin.read_line(&mut String::new()).unwrap();
    // store in memory
    data_buffer[data_ptr] = byte[0];
}
```

`.` just prints a single character to stdout based on the value pointed to by the data ptr. `,` looks a lot more complicated, but it's basically just doing the same thing in reverse. It reads a single byte from stdin and stores it in memory. The API for doing this in rust is a bit more complicated, because it's not nearly as common as printing.

The last two operators are `[` and `]`, which are used to implement a form of `goto`. These instructions allow you to jump to the matching instruction, which is actually very similar to how [control flow works in WebAssembly](https://developer.mozilla.org/en-US/docs/WebAssembly/Reference/Control_flow). There's lot of ways you can implement this, but we'll start with the simplest: loop over the instructions until we find a match.

```rust
b'[' => {
    // point to next instruction
    instruction_ptr += 1;

    // if the data isn't 0, execute the next instruction. otherwise we need to jump
    // to the matching `]` instruction
    if data_buffer[data_ptr] != 0 {
        continue;
    }

    // we need to find the closing brace matching this one, not just the first
    // closing brace we see
    let mut num_open_brackets = 0;

    while instruction_ptr < instruction_buffer.len() {
        match instruction_buffer[instruction_ptr] {
            b'[' => num_open_brackets += 1,
            b']' => {
                // if no open brackets, then we found our match
                if num_open_brackets == 0 {
                    instruction_ptr += 1;
                    break;
                // otherwise, "close" a bracket
                } else {
                    num_open_brackets -= 1;
                }
            }
            _ => {}
        }

        instruction_ptr += 1;
    }

    continue;
}
```

Here, we either execute the following instruction if `data_buffer[data_ptr]` is 0, otherwise we jump to the instruction after the matching `]`. The code for implementing `]` looks very similar, but we iterate in reverse.

Putting everything together:

```rust
fn main() {
    let mut data_buffer = vec![0_u8; 30_000];
    let instruction_buffer = b"++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.>---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.";

    let mut data_ptr = 0_usize;
    let mut instruction_ptr = 0_usize;

    while instruction_ptr < instruction_buffer.len() {
        match instruction_buffer[instruction_ptr] {
            b'>' => data_ptr += 1,
            b'<' => data_ptr -= 1,
            b'+' => data_buffer[data_ptr] = data_buffer[data_ptr].wrapping_add(1),
            b'-' => data_buffer[data_ptr] = data_buffer[data_ptr].wrapping_sub(1),
            b'.' => print!("{}", char::from(data_buffer[data_ptr])),
            b',' => {
                use std::io::Read;
                let mut byte = [0];
                let mut stdin = std::io::stdin();
                // read single byte
                stdin.read_exact(&mut byte).unwrap();
                // drop rest of line
                stdin.read_line(&mut String::new()).unwrap();
                // store in memory
                data_buffer[data_ptr] = byte[0];
            }
            b'[' => {
                // point to next instruction
                instruction_ptr += 1;

                // if the data is 0, execute the next instruction. otherwise we need to jump
                // to the matching `]` instruction
                if data_buffer[data_ptr] != 0 {
                    continue;
                }

                // we need to find the closing brace matching this one, not just the first
                // closing brace we see
                let mut num_open_brackets = 0;

                while instruction_ptr < instruction_buffer.len() {
                    match instruction_buffer[instruction_ptr] {
                        b'[' => num_open_brackets += 1,
                        b']' => {
                            // if no open brackets, then we found our match
                            if num_open_brackets == 0 {
                                instruction_ptr += 1;
                                break;
                            // otherwise, "close" a bracket
                            } else {
                                num_open_brackets -= 1;
                            }
                        }
                        _ => {}
                    }

                    instruction_ptr += 1;
                }

                continue;
            }
            b']' => {
                if data_buffer[data_ptr] == 0 {
                    instruction_ptr += 1;
                    continue;
                }

                let mut num_close_brackets = 0;

                instruction_ptr -= 1;

                while instruction_ptr < instruction_buffer.len() {
                    match instruction_buffer[instruction_ptr] {
                        b'[' => {
                            // if no close brackets, then we found our match
                            if num_close_brackets == 0 {
                                instruction_ptr += 1;
                                break;
                            // otherwise, "close" a bracket
                            } else {
                                num_close_brackets -= 1;
                            }
                        }
                        b']' => num_close_brackets += 1,
                        _ => {}
                    }

                    instruction_ptr -= 1;
                }

                continue;
            }

            _ => {}
        }

        instruction_ptr += 1;
    }
}
```

```sh
$ cargo r -q
Hello World!
```

It works! We successfully implemented a brainfuck interpreter.

But we have a problem. Our interpreter is super slow. If we try running [this program](https://github.com/pablojorge/brainfuck/blob/master/programs/primes.bf) to find all the primes under a given number, it takes way too long.

```sh
$ cargo b --release
$ ./target/release/bf
9
9


Primes up to: 2 3 5 7 11 13 17 19 23 29 31 37 41 43 47 53 59 61 67 71 73 79 83 89 97
```

On my machine it takes around 400ms to execute this program if I hard code the input. That's pretty bad!

## Optimizing `[` and `]`

Before doing anything more drastic, we can make our implementation of control flow a bit nicer by introducing an intermediate representation.

```rust
#[derive(Debug, Copy, Clone)]
enum Instruction {
    /// >
    AngleGt,
    /// <
    AngleLt,
    /// +
    Plus,
    /// -
    Minus,
    /// .
    Dot,
    /// ,
    Comma,
    /// [
    BracketOpen(usize),
    /// ]
    BracketClose(usize),
}
```

Then we can write a simple parser to convert the input to this representation.

```rust
fn parse_instructions(bytes: &[u8]) -> Vec<Instruction> {
    let mut instructions = Vec::new();

    let mut open_idxs = Vec::new();

    for b in bytes {
        match b {
            b'>' => instructions.push(Instruction::AngleGt),
            b'<' => instructions.push(Instruction::AngleLt),
            b'+' => instructions.push(Instruction::Plus),
            b'-' => instructions.push(Instruction::Minus),
            b'.' => instructions.push(Instruction::Dot),
            b',' => instructions.push(Instruction::Comma),
            b'[' => {
                let idx = instructions.len();
                open_idxs.push(idx);
                // placeholder value to be filled in when we find the matching `]`
                instructions.push(Instruction::BracketOpen(0));
            },
            b']' => {
                let idx = instructions.len();
                let matching_brace = open_idxs.pop().unwrap();
                instructions.push(Instruction::BracketClose(matching_brace));
                instructions[matching_brace] = Instruction::BracketOpen(idx);
            },
            _ => {},
        }
    }

    instructions
}
```

Our match statement looks roughly the same, but the implementation of `[` and `]` can become a lot simpler.

```rust
let instruction_buffer = parse_instructions(b"++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.>---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.");

// ...

match instruction_buffer[instruction_ptr] {
    // ...
    Instruction::BracketOpen(jmp) => {
        if data_buffer[data_ptr] != 0 {
            instruction_ptr += 1;
        } else {
            instruction_ptr = jmp + 1;
        }

        continue;
    }
    Instruction::BracketClose(jmp) => {
        if data_buffer[data_ptr] == 0 {
            instruction_ptr += 1;
        } else {
            instruction_ptr = jmp + 1;
        }

        continue;
    }
}
```

This is way simpler than traversing all the instructions to find the matching instruction each time. After making this change, our program to find all the primes under 99 takes about 200ms on my machine. 2x faster, but still not enough! We can do better.

## Writing a Compiler

A compiler is a program that takes a programming language and compiles it to some sort of assembly. Compilers for modern programming languages are huge and involve insane optimizations and complex type systems. We won't be worrying about any of that. Our first compiler is going to be as simple as possible.

What exactly are we compiling our program to?

Your CPU doesn't understand assembly. It understands machine code. Assembly is roughly the textual representation of machine code, but there are usually a few differences.

To convert assembly into machine code your CPU can understand, we need to use an assembler. Then, to create an executable your operating system can run, we need to use a linker.

### Writing Assembly

We're going to do something a bit hacky. There aren't any good libraries for going from assembly to executable in rust, so we're going to do string concatenation and then run [`nasm`](https://www.nasm.us/) on the result. [`iced_x86`](https://docs.rs/iced-x86/latest/iced_x86/) is a great library for assembling and disassembling x86, but has some issues with generating full binaries that make it unsuitable for our purposes here. `iced_x86` would be a great choice for a JIT compiler.

We'll initialize our assembly string like so,

```rust
let mut asm = String::new();
```

We need to start by setting some `nasm` boilerplate and creating our entrypoint. By convention the entrypoint to our program is named `_start` on Linux and `start` on Mac. I'm compiling this on Linux, so all the code examples below will use the Linux conventions, but I'll mention when there are differences for Mac.

```asm
%use masm

BITS 64

section .text

global _start

_start:
```

With that out of the way, how do we translate our interpreter to a compiler?

The first thing we want to think about is where to store the data pointer and the instruction pointer.

The instruction pointer is pretty easy — x86 already provides us a register called [`rip`](https://flint.cs.yale.edu/cs421/papers/x86-asm/asm.html#Control-Flow-Instructions:~:text=The%20x86%20processor%20maintains%20an%20instruction%20pointer%20(EIP)%20register%20that%20is%20a%2032%2Dbit%20value%20indicating%20the%20location%20in%20memory%20where%20the%20current%20instruction%20starts) which is exactly that. Plus, reusing the `rip` for moving between instructions lets us reuse `jmp` instructions and labels to do our control flow with `[` and `]`.

For the data pointer, we can store its value pretty much anywhere. Let's choose `rbx` since it's not really used by other constructs, like calling functions or syscalls.

The next question is, where do we store our data? In our interpreter, we allocate a block of 30,000 bytes in memory and initialize it all to 0. That memory never grows during our program so we'd actually be fine storing it in the data section of our binary. We can initialize that memory in our assembly like this:

```asm
section .data

data_buffer: db 30000 dup (0)
```

This declares a data section, and then initializes a buffer of 30,000 bytes to 0. We can reference `data_buffer` just like a label or a variable. The value of `data_buffer` is just a pointer to our initialized buffer of memory. For simplicity we'll declare this section in our boilerplate at the start, but it could also go anywhere in our program.

To start our program, we need to store the pointer to `data_buffer` in `rbx`. That's pretty simple: `mov rbx, data_buffer`.

So our actual prelude is:

```asm
%use masm

BITS 64

section .data

data_buffer: db 30000 dup (0)

section .text

global _start

_start:
    mov rbx, data_buffer

; the rest of our compiled code
```

Now we've translated our pointers and memory. It's time to start working on the instructions.

Like before, we loop through all our instructions.

```rust
fn main() {
    let instructions = parse_instructions(b"...");

    let mut asm = String::new();

    // ...

    for inst in instructions {
        match inst {
            // ...
        }
    }
}
```

And just like our interpreter, we can start filling out our match statement.

```rust
Instruction::AngleGt => asm += "add rbx, 1\n",
Instruction::AngleLt => asm += "sub rbx, 1\n",
```

`>` and `<` increment and decrement the data pointer respectively. Here, `rbx` holds our data pointer so we modify its value directly.

```rust
Instruction::Plus => asm += "add BYTE PTR [rbx], 1\n",
Instruction::Minus => asm += "sub BYTE PTR [rbx], 1\n",
```

`+` and `-` increment and decrement the value _pointed to_ by `rbx` respectively. The `[rbx]` in this syntax is a pointer dereference, so we're not modifying rbx directly, but rather the byte it's pointing to.

`,` and `.` are bit more complicated. They read from stdin and write to stdout respectively. To do that, we need to make syscalls. Syscalls are functions provided to us by the operating system. 

Syscalls, just like regular function calls in assembly, have specific calling conventions. We store the arguments in registers and execute the `syscall` instruction. To tell the operating system which syscall we want to make, we set the value in `rax` to a predefined number. For the `write` syscall, this number is [1 on Linux](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/) and `0x02000004` [on Mac](https://opensource.apple.com/source/xnu/xnu-1504.3.12/bsd/kern/syscalls.master). 

The [`write`](https://en.wikipedia.org/wiki/Write_(system_call)) syscall takes 3 arguments: the file handle, a pointer to the message to be written, and the number of bytes to write of that message. The Linux and Mac syscall ABI follows the ["diane's silk dress costs $89" mnemonic](https://csappbook.blogspot.com/2015/08/dianes-silk-dress-costs-89.html), meaning we pass in arguments in the order `rdi`, `rsi`, `rdx`, `rcx`, `r8`, and `r9`. Here we only have 3 arguments, so we just need to use the first 3 registers.

```asm
; select the write syscall on Linux
mov rax, 1
; file handle (1 for stdout)
mov rdi, 1
; pointer to message, which is our data buffer
mov rsi, rbx
; number of bytes to write, which is 1
mov edx, 1
syscall
```

This is our implementation of `.`, which prints to stdout. Next we have to do a [read](https://en.wikipedia.org/wiki/Read_(system_call)).

The API for read is pretty similar. It takes a file descriptor (0 for stdin), a pointer to somewhere in memory to write, and the number of bytes to read from stdin.

```asm
; select the read syscall on Linux
mov rax, 0
; file handle (0 for stdin)
mov rdi, 0
; pointer to message, which is our data buffer
mov rsi, rbx
; number of bytes to read, which is 1
mov edx, 1
syscall
```

Finally we have to compile our control flow instructions, `[` and `]`.

We're going to compile our loops to look something like this:

```asm
loop_start:
; condition
jmp loop_end
; loop code
jmp loop_start
loop_end:
; other instructions
```

We have to give each loop label a unique identifier, otherwise their labels would conflict with each other. We can do this pretty simply by keeping a strictly increasing integer which we increment every time we encounter a `[`. To match our `]` to `[`, we can keep a stack, similar to our optimization for matching braces in our interpreter.

```rust
let mut label_count = 0;
let mut labels = Vec::new();
```

And then in our match statement:

```rust
Instruction::BracketOpen(..) => {
    let loop_idx = label_count;
    label_count += 1;

    asm += &format!("
        cmp BYTE PTR [rbx], 0
        je loop{loop_idx}_close
        loop{loop_idx}_open:
    ");

    labels.push(loop_idx);
}
```

In our closing brace code we need to pop off the `labels` stack and jump to the start.

```rust
Instruction::BracketClose(jmp) => {
    let loop_idx = labels.pop().unwrap();

    asm += &format!("
        cmp BYTE PTR [rbx], 0
        jne loop{loop_idx}_open
        loop{loop_idx}_close:
    ");
}
```

And that's it! Now we just have to make our program exit gracefully. It can do that by calling the `exit` syscall with an error code of 0.

```asm
; select the exit syscall on Linux
mov rax, 60
; set exit code in rdi to 0
xor rdi, rdi
syscall
```

That's everything we need to write a compiler for brainfuck. Let's try printing out `asm` and seeing what the codegen looks like.

```sh
$ cargo r
```

```asm
%use masm

BITS 64

section .data

data_buffer: db 30000 dup (0)

section .text

global _start

_start:
mov rbx, data_buffer
add BYTE PTR [rbx], 1
add BYTE PTR [rbx], 1
add BYTE PTR [rbx], 1
add BYTE PTR [rbx], 1
add BYTE PTR [rbx], 1
add BYTE PTR [rbx], 1
add BYTE PTR [rbx], 1
add BYTE PTR [rbx], 1
cmp BYTE PTR [rbx], 0
je loop0_close
loop0_open:
add rbx, 1

; ... 

mov rax, 60
xor rdi, rdi
syscall
```

That looks pretty good to me! We can pipe that to a file and try assembling it with nasm.

```sh
$ cargo r > hello-world.s
$ nasm -felf64 hello-world.s
$ ld hello-world.o 
$ ./a.out
Hello World!
```

It works!

Here's the full code for this section:

```rust
fn main() {
    let instructions = parse_instructions(b"++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.>---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.");

    let mut asm = String::new();

    // nasm setup
    asm += "\
%use masm

BITS 64

section .data

data_buffer: db 30000 dup (0)

section .text

global _start

_start:
mov rbx, data_buffer
";

    let mut label_count = 0;
    let mut labels = Vec::new();

    for inst in instructions {
        match inst {
            Instruction::AngleGt => asm += "add rbx, 1\n",
            Instruction::AngleLt => asm += "sub rbx, 1\n",
            Instruction::Plus => asm += "add BYTE PTR [rbx], 1\n",
            Instruction::Minus => asm += "sub BYTE PTR [rbx], 1\n",
            Instruction::Dot => {
                asm += "\
                    mov rax, 1\n\
                    mov rdi, 1\n\
                    mov rsi, rbx\n\
                    mov edx, 1\n\
                    syscall\n\
                ";
            }
            Instruction::Comma => {
                asm += "\
                    mov rax, 0\n\
                    mov rdi, 0\n\
                    mov rsi, rbx\n\
                    mov edx, 1\n\
                    syscall\n\
                "
            }
            Instruction::BracketOpen(..) => {
                let loop_idx = label_count;
                label_count += 1;

                asm += &format!(
                    "\
                    cmp BYTE PTR [rbx], 0\n\
                    je loop{loop_idx}_close\n\
                    loop{loop_idx}_open:\n\
                "
                );

                labels.push(loop_idx);
            }
            Instruction::BracketClose(..) => {
                let loop_idx = labels.pop().unwrap();

                asm += &format!(
                    "\
                    cmp BYTE PTR [rbx], 0\n\
                    jne loop{loop_idx}_open\n\
                    loop{loop_idx}_close:\n\
                "
                );
            }
        }
    }

    // prologue
    asm += "\
        mov rax, 60\n\
        xor rdi, rdi\n\
        syscall\n\
    ";

    println!("{asm}");
}
```

### Compiler Optimizations

This is pretty good, but we can do better. For a simple hello world we generate over 200 instructions. 

Our assembly is pretty suboptimal, so we have plenty of room to improve. Take this sample from our assembly:

```asm
add BYTE PTR [rbx], 1
add BYTE PTR [rbx], 1
add BYTE PTR [rbx], 1
add BYTE PTR [rbx], 1
add BYTE PTR [rbx], 1
add BYTE PTR [rbx], 1
```

We have strings of `+++++` all over our hello world code, to increment the memory to a given character value. These repeated additions by 1 could be simplified into a single addition by a larger number.

To start optimizing, we need to change our intermediate representation a bit. Instead of separating out `<`/`>` and `+`/`-`, we can combine them in our intermediate representation. `<` and `-` are just adds by negative numbers.

```rust
#[derive(Debug, Copy, Clone)]
enum Instruction {
    /// > and <
    PointerAdd(isize),
    /// + and -
    ValueAdd(isize),
    /// .
    Dot,
    /// ,
    Comma,
    /// [
    BracketOpen,
    /// ]
    BracketClose,
}
```

In addition to combining our addition operators, we've added support for arbitrary numbers to `+`, `-`, `>`, and `<`. This way we can represent `++` as `Plus(2)`. We also removed the offset code from `[` and `]`, since that's handled by our compiler now.

We can change our parser to support this new representation.

```rust
fn parse_instructions(bytes: &[u8]) -> Vec<Instruction> {
    let mut instructions = Vec::new();

    for b in bytes {
        match b {
            b'>' | b'<' => {
                let v = match b {
                    b'>' => 1,
                    b'<' => -1,
                    _ => unreachable!(),
                };

                if let Some(Instruction::PointerAdd(n2)) = instructions.last_mut() {
                    *n2 += v;
                } else {
                    instructions.push(Instruction::PointerAdd(v))
                }
            }
            b'+' | b'-' => {
                let v = match b {
                    b'+' => 1,
                    b'-' => -1,
                    _ => unreachable!(),
                };

                if let Some(Instruction::ValueAdd(n2)) = instructions.last_mut() {
                    *n2 += v;
                } else {
                    instructions.push(Instruction::ValueAdd(v))
                }
            }
            b'.' => instructions.push(Instruction::Dot),
            b',' => instructions.push(Instruction::Comma),
            b'[' => instructions.push(Instruction::BracketOpen),
            b']' => instructions.push(Instruction::BracketClose),
            _ => {}
        }
    }

    instructions
}
```

This already makes our assembly a lot nicer. 

```asm
_start:
mov rbx, data_buffer
add BYTE PTR [rbx], 8
cmp BYTE PTR [rbx], 0
je loop0_close
loop0_open:
add rbx, 1
add BYTE PTR [rbx], 4
cmp BYTE PTR [rbx], 0
je loop1_close
loop1_open:
add rbx, 1
add BYTE PTR [rbx], 2
add rbx, 1
add BYTE PTR [rbx], 3
```

There's a couple other low hanging fruit. In our syscalls to read and write, we always set `edx` to 1, so instead of doing that before each syscall, let's just do it once at the start of our program.

```asm
section .text

global _start

_start:
mov rbx, data_buffer
mov edx, 1
```

That shaves off about 10 instructions from our hello world program.

We can make a similar optimization after realizing that we always set `rsi` to the value of `rbx` before doing a syscall. Why don't we just use `rsi` instead of `rbx` everywhere, and then we can make instructions like `mov rsi, rbx` superfluous. This shaves off another 10 instructions.

```asm
loop2_close:
add rsi, -1
add BYTE PTR [rsi], -1
cmp BYTE PTR [rsi], 0
jne loop0_open
loop0_close:
add rsi, 2
mov rax, 1
mov rdi, 1
syscall
add rsi, 1
add BYTE PTR [rsi], -3
mov rax, 1
mov rdi, 1
syscall
```

There's still plenty of optimization to be done — for example, removing the redundant `mov rax, 1` and `mov rdi, 1` between sequential write syscalls and trivial things like removing empty loops or additions by 0 — but I think what we have so far is sufficient to get the idea.

If we compare our first interpreter to our optimizing compiler on [this program which finds all the primes under 255](https://www.reddit.com/r/brainfuck/comments/847vl0/prime_number_generator_in_brainfuck/)[^1], our compiled program prints all the primes in 13 seconds. The interpreter ran for several minutes without printing anything before I killed it.

Here's our final code, if you'd like to modify it to generate ARM assembly or add more optimizations.

```rust
#[derive(Debug, Copy, Clone)]
enum Instruction {
    /// > and <
    PointerAdd(isize),
    /// + and -
    ValueAdd(isize),
    /// .
    Dot,
    /// ,
    Comma,
    /// [
    BracketOpen,
    /// ]
    BracketClose,
}

fn parse_instructions(bytes: &[u8]) -> Vec<Instruction> {
    let mut instructions = Vec::new();

    for b in bytes {
        match b {
            b'>' | b'<' => {
                let v = match b {
                    b'>' => 1,
                    b'<' => -1,
                    _ => unreachable!(),
                };

                if let Some(Instruction::PointerAdd(n2)) = instructions.last_mut() {
                    *n2 += v;
                } else {
                    instructions.push(Instruction::PointerAdd(v))
                }
            }
            b'+' | b'-' => {
                let v = match b {
                    b'+' => 1,
                    b'-' => -1,
                    _ => unreachable!(),
                };

                if let Some(Instruction::ValueAdd(n2)) = instructions.last_mut() {
                    *n2 += v;
                } else {
                    instructions.push(Instruction::ValueAdd(v))
                }
            }
            b'.' => instructions.push(Instruction::Dot),
            b',' => instructions.push(Instruction::Comma),
            b'[' => instructions.push(Instruction::BracketOpen),
            b']' => instructions.push(Instruction::BracketClose),
            _ => {}
        }
    }

    instructions
}

fn main() {
    let instructions = parse_instructions(b"
        ++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.>---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.
    ");

    let mut asm = String::new();

    // nasm setup
    asm += "\
%use masm

BITS 64

section .data

data_buffer: db 30000 dup (0)

section .text

global _start

_start:
mov rsi, data_buffer
mov edx, 1
";

    let mut label_count = 0;
    let mut labels = Vec::new();

    for inst in instructions {
        match inst {
            Instruction::PointerAdd(n) => asm += &format!("add rsi, {n}\n"),
            Instruction::ValueAdd(n) => asm += &format!("add BYTE PTR [rsi], {n}\n"),
            Instruction::Dot => {
                asm += "\
                    mov rax, 1\n\
                    mov rdi, 1\n\
                    syscall\n\
                ";
            }
            Instruction::Comma => {
                asm += "\
                    mov rax, 0\n\
                    mov rdi, 0\n\
                    syscall\n\
                "
            }
            Instruction::BracketOpen => {
                let loop_idx = label_count;
                label_count += 1;

                asm += &format!(
                    "\
                    cmp BYTE PTR [rsi], 0\n\
                    je loop{loop_idx}_close\n\
                    loop{loop_idx}_open:\n\
                "
                );

                labels.push(loop_idx);
            }
            Instruction::BracketClose => {
                let loop_idx = labels.pop().unwrap();

                asm += &format!(
                    "\
                    cmp BYTE PTR [rsi], 0\n\
                    jne loop{loop_idx}_open\n\
                    loop{loop_idx}_close:\n\
                "
                );
            }
        }
    }

    // prologue
    asm += "\
        mov rax, 60\n\
        xor rdi, rdi\n\
        syscall\n\
    ";

    println!("{asm}");
}
```

[^1]: the code linked only finds the primes up to 30, but can be easily modified to find up to 255