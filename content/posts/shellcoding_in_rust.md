+++
title = "Writing Shellcodes in Rust"
description = "Brief tutorial on how to write memory safe shellcodes using Rust."
date = 2022-07-09
+++

**TLDR** You can clone and modify [this example repository](https://github.com/zommiommy/rust_shellcoding).

Add the rust target to compile for plain x86_64:
```bash
rustup target add x86_64-unknown-none
```
Create a new binary crate:
```bash
cargo new --bin shellcode
```
Add to your `Cargo.toml` the following configurations for the release profile to make the shellcode as small as possible.
```toml
[profile.release]
panic = "abort"             # don't unwind the stack on panic
opt-level = "z"             # optimize for size
debug=0                     # no debug info
debug-assertions = false    #
overflow-checks = false     # optional
strip="symbols"             # remove symbols
```

The `main.rs` for our shellcode looks like this:
```rust
#![no_std]
#![no_main]
#![feature(core_intrinsics)]

const N: u64 = 100;

// our shellcode
#[no_mangle]
pub fn _start() {
    let mut buffer = [0; 1024];
    let mut sum = 0;
    for i in 0..N {
        let start = rdtsc();
        buffer[i as usize] = 0;
        let end = rdtsc();
        sum += end - start;
    }
    print!(sum / N);
}

/// Handle the panics, by default abort
#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    core::intrinsics::abort()
}

// simple macro to call the write syscall with an integer
macro_rules! print {
    ($value:expr) => {
        unsafe{
            core::arch::asm!(
                "push {input}",  // put the value to print on stack 
                "mov rdx, 8",   // len = 8
                "mov rsi, rsp", //char *buf = rsp
                "mov rdi, 1",   // fd = stdout
                "mov rax, 1",   // write
                "syscall",

                input = in(reg) $value,
                out("rdx") _,
                out("rsi") _,
                out("rdi") _,
                out("rax") _,
            );
        }
    };
}

// safe wrapper, this might fails only if we are not on x86_64 
pub fn rdtsc() -> u64 {
    unsafe{core::arch::x86_64::_rdtsc()}
}
```

Finally, we can compile the shellcode with:
```bash
RUSTFLAGS="-C relocation-model=pie" cargo build --release --target="x86_64-unknown-none"
```
If you want your shellcode to exploit AVX and other instructions you can specify your target CPU with `-C target-cpu=native`.

This will create a file `./target/x86_64-unknown-none/release/shellcode` which `.text` segment contains the shellcode.



```bash
objdump -D ./target/x86_64-unknown-none/release/shellcode -M intel -j .text

./target/x86_64-unknown-none/release/shellcode:     file format elf64-x86-64

Disassembly of section .text:

000000000000120d <.text>:
    120d:       6a 64                   push   0x64
    120f:       5f                      pop    rdi
    1210:       31 c9                   xor    ecx,ecx
    1212:       48 83 ef 01             sub    rdi,0x1
    1216:       72 1d                   jb     0x1235
    1218:       0f 31                   rdtsc  
    121a:       48 89 d6                mov    rsi,rdx
    121d:       48 c1 e6 20             shl    rsi,0x20
    1221:       48 09 c6                or     rsi,rax
    1224:       0f 31                   rdtsc  
    1226:       48 c1 e2 20             shl    rdx,0x20
    122a:       48 09 c2                or     rdx,rax
    122d:       48 29 f1                sub    rcx,rsi
    1230:       48 01 d1                add    rcx,rdx
    1233:       eb dd                   jmp    0x1212
    1235:       50                      push   rax
    1236:       6a 64                   push   0x64
    1238:       5e                      pop    rsi
    1239:       48 89 c8                mov    rax,rcx
    123c:       31 d2                   xor    edx,edx
    123e:       48 f7 f6                div    rsi
    1241:       48 89 c1                mov    rcx,rax
    1244:       51                      push   rcx
    1245:       48 c7 c2 08 00 00 00    mov    rdx,0x8
    124c:       48 89 e6                mov    rsi,rsp
    124f:       48 c7 c7 01 00 00 00    mov    rdi,0x1
    1256:       48 c7 c0 01 00 00 00    mov    rax,0x1
    125d:       0f 05                   syscall 
    125f:       58                      pop    rax
    1260:       c3                      ret    
```
To dump the shellcode to a binary file use `objcopy` to dump the whole text segment
```bash
objcopy -O binary ./target/x86_64-unknown-none/release/shellcode shellcode.bin -j .text
```

**Caveats**: Since we are dumping the `.text` segment, we can't use static or global variables because these would go in the sections
`.bss`, `.rodata`, and `.data` which we won't load.

The dumped shellcode can be examined with:
```bash
objdump -D -b binary -mi386 -Mx86-64 -Mintel ./shellcode.bin
./shellcode.bin:     file format binary


Disassembly of section .data:

00000000 <.data>:
   0:   6a 64                   push   0x64
   2:   5f                      pop    rdi
   3:   31 c9                   xor    ecx,ecx
   5:   48 83 ef 01             sub    rdi,0x1
   9:   72 1d                   jb     0x28
   b:   0f 31                   rdtsc  
   d:   48 89 d6                mov    rsi,rdx
  10:   48 c1 e6 20             shl    rsi,0x20
  14:   48 09 c6                or     rsi,rax
  17:   0f 31                   rdtsc  
  19:   48 c1 e2 20             shl    rdx,0x20
  1d:   48 09 c2                or     rdx,rax
  20:   48 29 f1                sub    rcx,rsi
  23:   48 01 d1                add    rcx,rdx
  26:   eb dd                   jmp    0x5
  28:   50                      push   rax
  29:   6a 64                   push   0x64
  2b:   5e                      pop    rsi
  2c:   48 89 c8                mov    rax,rcx
  2f:   31 d2                   xor    edx,edx
  31:   48 f7 f6                div    rsi
  34:   48 89 c1                mov    rcx,rax
  37:   51                      push   rcx
  38:   48 c7 c2 08 00 00 00    mov    rdx,0x8
  3f:   48 89 e6                mov    rsi,rsp
  42:   48 c7 c7 01 00 00 00    mov    rdi,0x1
  49:   48 c7 c0 01 00 00 00    mov    rax,0x1
  50:   0f 05                   syscall 
  52:   58                      pop    rax
  53:   c3                      ret    
  ```

If the code editor keeps warning about duplciated panic handler, just create the file `.cargo/config` with:
```toml
[build]
target = "x86_64-unknown-none"
```