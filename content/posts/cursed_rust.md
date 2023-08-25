+++
title = "Cursed rust types"
date = "2023-08-21"
description = "Weird types you can use in rust"
draft = true
+++

While writing `epserde` we asked ourselves, can we consider the alginement of values
constant?


Print all the builtin targets of rustc (which btw are defined [here](https://github.com/rust-lang/rust/blob/b131febeb0e957628ab3d01f295e7c2333acf474/compiler/rustc_target/src/spec/mod.rs#L1270)):
```bash
$ rustc --print cfg --print target-list
```

Print the specific informations about a target:
```bash
$ rustc --print cfg --target x86_64-unknown-none
debug_assertions
overflow_checks
panic="abort"
target_abi=""
target_arch="x86_64"
target_endian="little"
target_env=""
target_feature="fxsr"
target_has_atomic
target_has_atomic="16"
target_has_atomic="32"
target_has_atomic="64"
target_has_atomic="8"
target_has_atomic="ptr"
target_has_atomic_equal_alignment="16"
target_has_atomic_equal_alignment="32"
target_has_atomic_equal_alignment="64"
target_has_atomic_equal_alignment="8"
target_has_atomic_equal_alignment="ptr"
target_has_atomic_load_store
target_has_atomic_load_store="16"
target_has_atomic_load_store="32"
target_has_atomic_load_store="64"
target_has_atomic_load_store="8"
target_has_atomic_load_store="ptr"
target_os="none"
target_pointer_width="64"
target_vendor="unknown"
```

We can see that the start of parsing of a target is ([HERE](
https://github.com/rust-lang/rust/blob/b131febeb0e957628ab3d01f295e7c2333acf474/compiler/rustc_target/src/spec/mod.rs#L2382)): 
```rust
/// Loads a target descriptor from a JSON object.
pub fn from_json(obj: Json) -> Result<(Target, TargetWarnings), String> {
    /// ....
    let mut base = Target {
        llvm_target: get_req_field("llvm-target")?.into(),
        pointer_width: get_req_field("target-pointer-width")?
            .parse::<u32>()
            .map_err(|_| "target-pointer-width must be an integer".to_string())?,
        data_layout: get_req_field("data-layout")?.into(),
        arch: get_req_field("arch")?.into(),
        options: Default::default(),
    };
    /// ....
} 
```

`parse_from_llvm_datalayout_string` which parses an [llvm data layout string](https://llvm.org/docs/LangRef.html#data-layout)

```rust
/// Parsed [Data layout](https://llvm.org/docs/LangRef.html#data-layout)
/// for a target, which contains everything needed to compute layouts.
#[derive(Debug, PartialEq, Eq)]
pub struct TargetDataLayout {
    pub endian: Endian,
    pub i1_align: AbiAndPrefAlign,
    pub i8_align: AbiAndPrefAlign,
    pub i16_align: AbiAndPrefAlign,
    pub i32_align: AbiAndPrefAlign,
    pub i64_align: AbiAndPrefAlign,
    pub i128_align: AbiAndPrefAlign,
    pub f32_align: AbiAndPrefAlign,
    pub f64_align: AbiAndPrefAlign,
    pub pointer_size: Size,
    pub pointer_align: AbiAndPrefAlign,
    pub aggregate_align: AbiAndPrefAlign,

    /// Alignments for vector types.
    pub vector_align: Vec<(Size, AbiAndPrefAlign)>,

    pub instruction_address_space: AddressSpace,

    /// Minimum size of #[repr(C)] enums (default c_int::BITS, usually 32)
    /// Note: This isn't in LLVM's data layout string, it is `short_enum`
    /// so the only valid spec for LLVM is c_int::BITS or 8
    pub c_enum_min_size: Integer,
}
```

So let's check if any of the bultin targets have a different `data-layout`:
```python
import json
import subprocess

stdout = subprocess.check_output("rustc -Z unstable-options --print all-target-specs-json ", shell=True).decode()
data = json.loads(stdout)

for arch, opts in data.items():
    data_layout = opts['data-layout']
    print(f'{arch}: {data_layout}')
```
which prints:
```
aarch64-apple-darwin: e-m:o-i64:64-i128:128-n32:64-S128
aarch64-apple-ios: e-m:o-i64:64-i128:128-n32:64-S128
aarch64-apple-ios-macabi: e-m:o-i64:64-i128:128-n32:64-S128
aarch64-apple-ios-sim: e-m:o-i64:64-i128:128-n32:64-S128
aarch64-apple-tvos: e-m:o-i64:64-i128:128-n32:64-S128
aarch64-apple-watchos-sim: e-m:o-i64:64-i128:128-n32:64-S128
aarch64-fuchsia: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-kmc-solid_asp3: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-linux-android: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-nintendo-switch-freestanding: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-pc-windows-gnullvm: e-m:w-p:64:64-i32:32-i64:64-i128:128-n32:64-S128
aarch64-pc-windows-msvc: e-m:w-p:64:64-i32:32-i64:64-i128:128-n32:64-S128
aarch64-unknown-freebsd: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-unknown-fuchsia: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-unknown-hermit: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-unknown-linux-gnu: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-unknown-linux-gnu_ilp32: e-m:e-p:32:32-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-unknown-linux-musl: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-unknown-linux-ohos: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-unknown-netbsd: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-unknown-none: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-unknown-none-softfloat: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-unknown-nto-qnx710: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-unknown-openbsd: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-unknown-redox: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64-unknown-uefi: e-m:w-p:64:64-i32:32-i64:64-i128:128-n32:64-S128
aarch64-uwp-windows-msvc: e-m:w-p:64:64-i32:32-i64:64-i128:128-n32:64-S128
aarch64-wrs-vxworks: e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64_be-unknown-linux-gnu: E-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64_be-unknown-linux-gnu_ilp32: E-m:e-p:32:32-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
aarch64_be-unknown-netbsd: E-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128
arm-linux-androideabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
arm-unknown-linux-gnueabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
arm-unknown-linux-gnueabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
arm-unknown-linux-musleabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
arm-unknown-linux-musleabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
arm64_32-apple-watchos: e-m:o-p:32:32-i64:64-i128:128-n32:64-S128
armeb-unknown-linux-gnueabi: E-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armebv7r-none-eabi: E-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armebv7r-none-eabihf: E-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv4t-none-eabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv4t-unknown-linux-gnueabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv5te-none-eabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv5te-unknown-linux-gnueabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv5te-unknown-linux-musleabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv5te-unknown-linux-uclibceabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv6-unknown-freebsd: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv6-unknown-netbsd-eabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv6k-nintendo-3ds: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7-apple-ios: e-m:o-p:32:32-Fi8-f64:32:64-v64:32:64-v128:32:128-a:0:32-n32-S32
armv7-linux-androideabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7-sony-vita-newlibeabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7-unknown-freebsd: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7-unknown-linux-gnueabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7-unknown-linux-gnueabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7-unknown-linux-musleabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7-unknown-linux-musleabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7-unknown-linux-ohos: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7-unknown-linux-uclibceabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7-unknown-linux-uclibceabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7-unknown-netbsd-eabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7-wrs-vxworks-eabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7a-kmc-solid_asp3-eabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7a-kmc-solid_asp3-eabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7a-none-eabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7a-none-eabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7k-apple-watchos: e-m:o-p:32:32-Fi8-i64:64-a:0:32-n32-S128
armv7r-none-eabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7r-none-eabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
armv7s-apple-ios: e-m:o-p:32:32-Fi8-f64:32:64-v64:32:64-v128:32:128-a:0:32-n32-S32
asmjs-unknown-emscripten: e-m:e-p:32:32-p10:8:8-p20:8:8-i64:64-f128:64-n32:64-S128-ni:1:10:20
avr-unknown-gnu-atmega328: e-P1-p:16:8-i8:8-i16:8-i32:8-i64:8-f32:8-f64:8-n8-a:8
bpfeb-unknown-none: E-m:e-p:64:64-i64:64-i128:128-n32:64-S128
bpfel-unknown-none: e-m:e-p:64:64-i64:64-i128:128-n32:64-S128
hexagon-unknown-linux-musl: e-m:e-p:32:32:32-a:0-n16:32-i64:64:64-i32:32:32-i16:16:16-i1:8:8-f32:32:32-f64:64:64-v32:32:32-v64:64:64-v512:512:512-v1024:1024:1024-v2048:2048:2048
i386-apple-ios: e-m:o-p:32:32-p270:32:32-p271:32:32-p272:64:64-f64:32:64-f80:128-n8:16:32-S128
i586-pc-nto-qnx700: e-m:e-p:32:32-p270:32:32-p271:32:32-p272:64:64-f64:32:64-f80:32-n8:16:32-S128
i586-pc-windows-msvc: e-m:x-p:32:32-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32-a:0:32-S32
i586-unknown-linux-gnu: e-m:e-p:32:32-p270:32:32-p271:32:32-p272:64:64-f64:32:64-f80:32-n8:16:32-S128
i586-unknown-linux-musl: e-m:e-p:32:32-p270:32:32-p271:32:32-p272:64:64-f64:32:64-f80:32-n8:16:32-S128
i686-apple-darwin: e-m:o-p:32:32-p270:32:32-p271:32:32-p272:64:64-f64:32:64-f80:128-n8:16:32-S128
i686-linux-android: e-m:e-p:32:32-p270:32:32-p271:32:32-p272:64:64-f64:32:64-f80:32-n8:16:32-S128
i686-pc-windows-gnu: e-m:x-p:32:32-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:32-n8:16:32-a:0:32-S32
i686-pc-windows-msvc: e-m:x-p:32:32-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32-a:0:32-S32
i686-unknown-freebsd: e-m:e-p:32:32-p270:32:32-p271:32:32-p272:64:64-f64:32:64-f80:32-n8:16:32-S128
i686-unknown-haiku: e-m:e-p:32:32-p270:32:32-p271:32:32-p272:64:64-f64:32:64-f80:32-n8:16:32-S128
i686-unknown-linux-gnu: e-m:e-p:32:32-p270:32:32-p271:32:32-p272:64:64-f64:32:64-f80:32-n8:16:32-S128
i686-unknown-linux-musl: e-m:e-p:32:32-p270:32:32-p271:32:32-p272:64:64-f64:32:64-f80:32-n8:16:32-S128
i686-unknown-netbsd: e-m:e-p:32:32-p270:32:32-p271:32:32-p272:64:64-f64:32:64-f80:32-n8:16:32-S128
i686-unknown-openbsd: e-m:e-p:32:32-p270:32:32-p271:32:32-p272:64:64-f64:32:64-f80:32-n8:16:32-S128
i686-unknown-uefi: e-m:x-p:32:32-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:32-n8:16:32-a:0:32-S32
i686-uwp-windows-gnu: e-m:x-p:32:32-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:32-n8:16:32-a:0:32-S32
i686-uwp-windows-msvc: e-m:x-p:32:32-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32-a:0:32-S32
i686-wrs-vxworks: e-m:e-p:32:32-p270:32:32-p271:32:32-p272:64:64-f64:32:64-f80:32-n8:16:32-S128
loongarch64-unknown-linux-gnu: e-m:e-p:64:64-i64:64-i128:128-n64-S128
loongarch64-unknown-none: e-m:e-p:64:64-i64:64-i128:128-n64-S128
loongarch64-unknown-none-softfloat: e-m:e-p:64:64-i64:64-i128:128-n64-S128
m68k-unknown-linux-gnu: E-m:e-p:32:16:32-i8:8:8-i16:16:16-i32:16:32-n8:16:32-a:0:16-S16
mips-unknown-linux-gnu: E-m:m-p:32:32-i8:8:32-i16:16:32-i64:64-n32-S64
mips-unknown-linux-musl: E-m:m-p:32:32-i8:8:32-i16:16:32-i64:64-n32-S64
mips-unknown-linux-uclibc: E-m:m-p:32:32-i8:8:32-i16:16:32-i64:64-n32-S64
mips64-openwrt-linux-musl: E-m:e-i8:8:32-i16:16:32-i64:64-n32:64-S128
mips64-unknown-linux-gnuabi64: E-m:e-i8:8:32-i16:16:32-i64:64-n32:64-S128
mips64-unknown-linux-muslabi64: E-m:e-i8:8:32-i16:16:32-i64:64-n32:64-S128
mips64el-unknown-linux-gnuabi64: e-m:e-i8:8:32-i16:16:32-i64:64-n32:64-S128
mips64el-unknown-linux-muslabi64: e-m:e-i8:8:32-i16:16:32-i64:64-n32:64-S128
mipsel-sony-psp: e-m:m-p:32:32-i8:8:32-i16:16:32-i64:64-n32-S64
mipsel-sony-psx: e-m:m-p:32:32-i8:8:32-i16:16:32-i64:64-n32-S64
mipsel-unknown-linux-gnu: e-m:m-p:32:32-i8:8:32-i16:16:32-i64:64-n32-S64
mipsel-unknown-linux-musl: e-m:m-p:32:32-i8:8:32-i16:16:32-i64:64-n32-S64
mipsel-unknown-linux-uclibc: e-m:m-p:32:32-i8:8:32-i16:16:32-i64:64-n32-S64
mipsel-unknown-none: e-m:m-p:32:32-i8:8:32-i16:16:32-i64:64-n32-S64
mipsisa32r6-unknown-linux-gnu: E-m:m-p:32:32-i8:8:32-i16:16:32-i64:64-n32-S64
mipsisa32r6el-unknown-linux-gnu: e-m:m-p:32:32-i8:8:32-i16:16:32-i64:64-n32-S64
mipsisa64r6-unknown-linux-gnuabi64: E-m:e-i8:8:32-i16:16:32-i64:64-n32:64-S128
mipsisa64r6el-unknown-linux-gnuabi64: e-m:e-i8:8:32-i16:16:32-i64:64-n32:64-S128
msp430-none-elf: e-m:e-p:16:16-i32:16-i64:16-f32:16-f64:16-a:8-n8:16-S16
nvptx64-nvidia-cuda: e-i64:64-i128:128-v16:16-v32:32-n16:32:64
powerpc-unknown-freebsd: E-m:e-p:32:32-i64:64-n32
powerpc-unknown-linux-gnu: E-m:e-p:32:32-i64:64-n32
powerpc-unknown-linux-gnuspe: E-m:e-p:32:32-i64:64-n32
powerpc-unknown-linux-musl: E-m:e-p:32:32-i64:64-n32
powerpc-unknown-netbsd: E-m:e-p:32:32-i64:64-n32
powerpc-unknown-openbsd: E-m:e-p:32:32-i64:64-n32
powerpc-wrs-vxworks: E-m:e-p:32:32-i64:64-n32
powerpc-wrs-vxworks-spe: E-m:e-p:32:32-i64:64-n32
powerpc64-ibm-aix: E-m:a-i64:64-n32:64-S128-v256:256:256-v512:512:512
powerpc64-unknown-freebsd: E-m:e-i64:64-n32:64
powerpc64-unknown-linux-gnu: E-m:e-i64:64-n32:64-S128-v256:256:256-v512:512:512
powerpc64-unknown-linux-musl: E-m:e-i64:64-n32:64-S128-v256:256:256-v512:512:512
powerpc64-unknown-openbsd: E-m:e-i64:64-n32:64
powerpc64-wrs-vxworks: E-m:e-i64:64-n32:64-S128-v256:256:256-v512:512:512
powerpc64le-unknown-freebsd: e-m:e-i64:64-n32:64
powerpc64le-unknown-linux-gnu: e-m:e-i64:64-n32:64-S128-v256:256:256-v512:512:512
powerpc64le-unknown-linux-musl: e-m:e-i64:64-n32:64-S128-v256:256:256-v512:512:512
riscv32gc-unknown-linux-gnu: e-m:e-p:32:32-i64:64-n32-S128
riscv32gc-unknown-linux-musl: e-m:e-p:32:32-i64:64-n32-S128
riscv32i-unknown-none-elf: e-m:e-p:32:32-i64:64-n32-S128
riscv32im-unknown-none-elf: e-m:e-p:32:32-i64:64-n32-S128
riscv32imac-esp-espidf: e-m:e-p:32:32-i64:64-n32-S128
riscv32imac-unknown-none-elf: e-m:e-p:32:32-i64:64-n32-S128
riscv32imac-unknown-xous-elf: e-m:e-p:32:32-i64:64-n32-S128
riscv32imc-esp-espidf: e-m:e-p:32:32-i64:64-n32-S128
riscv32imc-unknown-none-elf: e-m:e-p:32:32-i64:64-n32-S128
riscv64gc-unknown-freebsd: e-m:e-p:64:64-i64:64-i128:128-n32:64-S128
riscv64gc-unknown-fuchsia: e-m:e-p:64:64-i64:64-i128:128-n32:64-S128
riscv64gc-unknown-linux-gnu: e-m:e-p:64:64-i64:64-i128:128-n32:64-S128
riscv64gc-unknown-linux-musl: e-m:e-p:64:64-i64:64-i128:128-n32:64-S128
riscv64gc-unknown-netbsd: e-m:e-p:64:64-i64:64-i128:128-n32:64-S128
riscv64gc-unknown-none-elf: e-m:e-p:64:64-i64:64-i128:128-n32:64-S128
riscv64gc-unknown-openbsd: e-m:e-p:64:64-i64:64-i128:128-n32:64-S128
riscv64imac-unknown-none-elf: e-m:e-p:64:64-i64:64-i128:128-n32:64-S128
s390x-unknown-linux-gnu: E-m:e-i1:8:16-i8:8:16-i64:64-f128:64-v128:64-a:8:16-n32:64
s390x-unknown-linux-musl: E-m:e-i1:8:16-i8:8:16-i64:64-f128:64-v128:64-a:8:16-n32:64
sparc-unknown-linux-gnu: E-m:e-p:32:32-i64:64-f128:64-n32-S64
sparc64-unknown-linux-gnu: E-m:e-i64:64-n32:64-S128
sparc64-unknown-netbsd: E-m:e-i64:64-n32:64-S128
sparc64-unknown-openbsd: E-m:e-i64:64-n32:64-S128
sparcv9-sun-solaris: E-m:e-i64:64-n32:64-S128
thumbv4t-none-eabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
thumbv5te-none-eabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
thumbv6m-none-eabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
thumbv7a-pc-windows-msvc: e-m:w-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
thumbv7a-uwp-windows-msvc: e-m:w-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
thumbv7em-none-eabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
thumbv7em-none-eabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
thumbv7m-none-eabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
thumbv7neon-linux-androideabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
thumbv7neon-unknown-linux-gnueabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
thumbv7neon-unknown-linux-musleabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
thumbv8m.base-none-eabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
thumbv8m.main-none-eabi: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
thumbv8m.main-none-eabihf: e-m:e-p:32:32-Fi8-i64:64-v128:64:128-a:0:32-n32-S64
wasm32-unknown-emscripten: e-m:e-p:32:32-p10:8:8-p20:8:8-i64:64-f128:64-n32:64-S128-ni:1:10:20
wasm32-unknown-unknown: e-m:e-p:32:32-p10:8:8-p20:8:8-i64:64-n32:64-S128-ni:1:10:20
wasm32-wasi: e-m:e-p:32:32-p10:8:8-p20:8:8-i64:64-n32:64-S128-ni:1:10:20
wasm64-unknown-unknown: e-m:e-p:64:64-p10:8:8-p20:8:8-i64:64-n32:64-S128-ni:1:10:20
x86_64-apple-darwin: e-m:o-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-apple-ios: e-m:o-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-apple-ios-macabi: e-m:o-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-apple-tvos: e-m:o-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-apple-watchos-sim: e-m:o-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-fortanix-unknown-sgx: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-fuchsia: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-linux-android: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-pc-nto-qnx710: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-pc-solaris: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-pc-windows-gnu: e-m:w-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-pc-windows-gnullvm: e-m:w-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-pc-windows-msvc: e-m:w-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-sun-solaris: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-dragonfly: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-freebsd: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-fuchsia: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-haiku: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-hermit: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-illumos: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-l4re-uclibc: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-linux-gnu: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-linux-gnux32: e-m:e-p:32:32-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-linux-musl: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-netbsd: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-none: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-openbsd: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-redox: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-unknown-uefi: e-m:w-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-uwp-windows-gnu: e-m:w-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-uwp-windows-msvc: e-m:w-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64-wrs-vxworks: e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
x86_64h-apple-darwin: e-m:o-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128
```