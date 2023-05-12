+++
title = "Optimizing HyperLogLog"
description = "Different tecniques I used to speed up our implementation of HyperLogLog Counters"
date = 2023-05-12
+++

# HyperLogLog

[HyperLogLog counters](https://en.wikipedia.org/wiki/HyperLogLog) are a smart way to **estimate** the number of **uniques** 
elements seen in a stream. They exploit the fact that the number \(N\) of leading 
(or trailing) zeros (or ones), on a random number follows a geometric 
distribution of parameter \\(p = \frac{1}{2}\\), i.e.:

\\[P(N = n) \propto \frac{1}{2^{n}}\\]

The jist of the HyperLogLog counters is to use an hash function on the values and 
keep track of the maximum number of leading zeros. 
To be more resiliant to outliers, multiple buckets are used.
The first few bits are used to pick in which bucket each hashed value belongs to. 
In order to compute the caradinality estimate, we take the [**harmonic mean**](https://en.wikipedia.org/wiki/Harmonic_mean)
of the value of each bucket:

\\[|C| \approx \left( \sum_i \frac{1}{2^{x_i}} \right)^{-1}\\]

In this post I'll explore the different implementations I experimented with
to optimize this simple but fundamental operation.

### Naive implementation
This is the simplest way to compute it.
```rust
let mut total: f32 = 0.0;
for value in values {
    total += 1.0 / 2.0.powf(value as f32);
}
```
The assembly inside the loop looks like this:
```asm
movzx  edi, byte ptr [r14 + r15],    ; Load `value` from `values`
vmovss xmm0, dword ptr [rip + .TWO]  ; Load 2.0
call   r12                           ; Call ldexpf -> xmm0 = 2.0^values
vmovss xmm1, dword ptr [rip + .ONE]  ; Load 1.0
vdivss xmm0, xmm1, xmm0              ; 1.0 / 2.0^value
vadds  xmm2, xmm1, xmm0              ; total +=
```

### Avoiding the call to ldexpf
Having a call in the function adds a lot of overhead so we can avoid it this way:
```rust
let mut total: f32 = 0.0;
for value in values {
    total += 1.0 / ((1 << value) as f32);
}
```
The assembly inside the loop looks like this:
```asm
movzx r8d, byte ptr [rax] ; Load `value` from `values`
mov   edx, 1              ; 1
shlx  r8d, edx, e8d       ; 1 << value ;  3 cycles
vcvtsi2ss xmm2, xmm3, r8d ; as f32     ;  8 cycles
vdivss xmm2, xmm1, xmm2   ; 1.0 /      ; 18 cycles
vaddss  xmm0, xmm0, xmm2  ; total +=   ;  3 cycles
```

### Avoiding Divisions
One way to avoid division is to pre-compute all the values and look them. Assuming that the table in in the L1 Cache we can avoid the division and the conversion to f32, trading 18 + 8 = 26 cycles for a memory read which is about 10 cycles.
```rust
let mut total: f32 = 0.0;
for value in values {
    total += RECIPROCALS_TABLE[value as usize];
}
```
The assembly inside the loop looks like this:
```asm
movzx r8d, byte ptr [rax]  ; Load `value` from `values`
movd xmm2, ptr [r12 + r8d] ; RECIPROCALS_TABLE ; ~10 cycles 
vaddss  xmm0, xmm0, xmm2   ; total +=          ;   3 cycles
```

### IEEE 754 tricks
The [IEEE 754 floats](https://en.wikipedia.org/wiki/IEEE_754), we use and love everyday, represents values as:
![](https://upload.wikimedia.org/wikipedia/commons/thumb/d/d2/Float_example.svg/1024px-Float_example.svg.png)

where a value is decoded as:
\\[\text{value} = \text{sign}() \cdot (1 + \text{fraction}) \cdot 2^{\text{exponent} - 127}\\]

So with a bit of bit trickery we can directly build the value with just a sum and a shift:

```rust
let mut total: f32 = 0.0;
for value in values {
    let hack = (127 + value as u32) << 23;
    total += f32::from_ne_bytes(hack.to_ne_bytes());
}
```
The assembly inside the loop looks like this:
```asm
movzx edx, byte ptr [rax] ; Load `value` from `values`
shl   edx, 23             ; << 23         ; 1 cycle
add ecx, 1065353216       ; + (127 << 23) ; 1 cycle
movd xmm1, ecx            ; as f32        ; 3 cycles
vaddss xmm0, xmm0, xmm1   ; total +=      ; 3 cycles
```

### Fixed point for precision and profit
If we assume that the values are less or equal than 32, we can use an u64 word as a [fixed point precision](https://en.wikipedia.org/wiki/Fixed-point_arithmetic) value with 32 decimal bits.
This allows us to compute the exact result with no loss of resolution and further speed up the code.

```rust
let mut counter: u64 = 0;
for value in values {
    counter += (1_u64 << 32) >> value;
}
let exp = counter.leading_zeros() + 1;
counter <<= exp;
counter >>= 64 - 23;
let res = (((127 + 32 - exp) as u32) << 23) | (counter as u32);
let estimate = f32::from_ne_bytes(res.to_ne_bytes());
```
The assembly inside the loop looks like this:
```asm
movzx esi, byte ptr [rax]   ; Load `value` from `values`
shrx rsi, rcx, rsi          ; <<         ; 3 cycles
add rdx, rsi                ; counter += ; 1 cycle
```

## Benchmarks
Computing the sum of 10'000 elements (between 0 and 32) on my `Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz`

| Impl | ns / element | speedup |
|--- |---- |---- | 
| naive       | 9.2354 | / |
| no call     | 1.4025 | 6.58 |
| table       | 1.1763 | 7.85 |
| IEE 754     | 1.1586 | 7.97 |
| Fixed Point | 0.5693 | 16.22|

And compiling with `RUSTFLAGS="-C target-cpu=native"` we get:

| Impl | ns / element | speedup |
|--- |---- |---- | 
| naive       | 8.3358 | / |
| no call     | 1.2977 | 6.42 |
| table       | 1.1572 | 7.20 |
| IEE 754     | 1.1303 | 7.37 |
| Fixed Point | 0.1351 | 61.7 |

## Reference
- [Code for benchmarks](https://gist.github.com/zommiommy/f6a42d8c7c59826f45aa1c0cef687c1a)
- [Code on Godbolt](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DEArgoKkl9ZATwDKjdAGFUtEywYgAzADZSjgBk8BkwAOXcAI0xiEEkADlIAB1QFQjsGFzcPbz9k1NsBIJDwliiY%2BMtMawKGIQImYgJM909fSur0uoaCIrDI6NiE827m7Lbhxt6SssGASktUE2Jkdg4AUg0AQUSTCIBqKgY9gDcuCGOxE0wFED21gCYfNYBWACETOJeAEVm9gFo1l5sAcvPc7gB2V4bTZ7WF7egEPYsEyIoj1Wi3Kigu5eL57DQAOg0gKhWzhBxIJ0umD2wSpbmuENJMPJcLRYju9yhuL2XCJewA9Ht7kSAPpYkXJADuVAgACoLgy9kwFCD7rMSdDyWtwV8tWzUOjoTq9VtjVsdvtDid7udqTdOU83h9vr8AUC1Uz9bCEUiUXt2RjPYC8YTiV5meT%2BMR6VdaUdFVdVTrI6zYYHOdy8XyNIK9lAuDinICnHsFdTfiq1RqI96IaaWenDWJjbrzdtdgcE147QyHQ8ne9Ps8fv9AcCJV6yXDfcjUc2g5OQ/iiZrp7Do7GaXTE4yU3XyRmHlm9gAlbBOACSylPAHknJsAkJRQAVTavALYF6vctKqtmPAAC9MG%2BNdG3rOtA1bBtoUtLsTkkXsk1uAdvxdEc3XHYNITrWd/UDTFsWXMMwKjSld3jLdkxw9dWV9BAmGQABrHE8QgLh7nBTMywo/9QV%2BEsS2FLxSLTAMF0zZcJRAEAqGIVAWFFEJRQiABPAhrggBjmIJIglMwFT1M02Ya1TWETUghdoPbODrWOZ4kOuFDHjQ4dR3dCciJo8C8MRNATEEAY9hMHxJFY/FRI3cjqUo3dqLM1l/MCmNj2XdjRRCsLBK8Ut%2BJxL8PV/K5Iog2jfUwVREnCpKNOIAl6CYfAjFFYD5IUCABK5XkSpq6Ii2yvEKsSHrFmS/Lx2XUL/mEkrfWIPceSgdjOO47E/j2IbK1VEw8uy0t7i8ATwVLCBepjPj1RK6TZPkxTlLUjT2vmhRdNQfTDMejqNTNNszS2NAGHMM8L2vO8HyfV930/ZzBwlb5wtQt46xzUg60JZ5Udowl7gxtGCQ43GsaJHwccx8Cwy8AmyfJMMuGeEnCfJokNHBOIqbxjQNC8ABODQGepuEww0LhueeSnSY5znufBHx6YlonOY0eI4nuNn5aZxX7kkSQpD59WaeZ4X7nuFmufZ2ifHxrnnjphnMD%2BDRGfJLwiRt8FnlZgn7cdgXYT5Z4cbiVnebth2nbhcECRJ7meckMWvYdvw6xdtnJB8aXHjl55vaT2i%2BV58EvHiHxQTV7PE99vZuYJMWfHBSRKdCvXy5ZyvJAJGW4i8cE6ZVs2ce98FK5FLw4h1uIPe5rhhazwfK75EWTe51XReeFmy7nutngJbmfCV0LJHBd2eb5rxvYSOsRW5uINAO43R5l0XxfP%2BeCUkXmuC4HxP5jseZfpl%2BdZI5x0djfZ4u9l6j1Vt7bmlcXbghxvcXmSCx7fynpIGBr84gk1Cjbd%2BB0NCi3BJgus1dKaghxvXb%2BzxD4T3ttPNuUd6bf1VoXFmhcY70I0MPAkpcvB8wbrLUe6cS5cNfl/HWYt7hcBVunIubMxFbwJH3GWe8p5FzrjrehXAeFTy5l3JW9NC7dyVto1%2BccbZTwDqrXwPgJ73DMUAgknFwEe2/uPIujx6H3DgVHbu0s4g33BIQrgPcz5/A4lgmR19uaSGtnHCestvGV2roQ9%2Bh9hZHyQarBxESvCMJofXQua9AmhVClweh%2BTL7OMLoXOuNCG53z5pU8RvhsFdyyV/L%2BXMWlKLsQ3ce2CNA3xvrkqQPCx7SOCbE/hcS4kVIidIZGb9pHDMKXTE2JtTGLMrpHaeNDOJeDccLTmYzw6whds8AOYS66BM5mvBZdNX5HxLvTLusTTlr3Lk8uscRlFs1ZjraWnMCZjNzuBdu2thnL0IRPTmOD6HgvJCKY2Ss05K2XmvchiLxHT3vrfNOONbbfKRXCbeVzP50w4uLD2Xd6FD2qUfK5Ht6a2xXlIelr8H5HyDkE1OcdiERIZRbHeo84gx15ifBB0CIkX1oi7NOsTFWWNrhy2Vzyi4IM1eAzuLN6FyvAn8uuvhQn%2BMCXEo59DYF1nbl4Npwy7Hc1iSbGRVqeFfzHkrIu78e5Szdcs4ZkhNmhK1hPNe2t/VmhHCSDg8wMQcGeLwTwHAtCkFQJwU8ZhEQKEWMsGkDxKa8AIJoWN8wmIgBtjvZ4KstY0P4TfWJGN42SF4CwCQnNSDJtTemjgvAbjcOLSm2NpA4CwCQGgFgiQ6DRHIJQCdU76AxAYHgYACACC0FUqQLAxw8ArAAGp4EwFKW8iRGCcB4HwOgtUbgQAiCW0gERggNFUue3gj7mDEFUreCI2hMA2FfaQCdbBBC3gYBu%2B9WAWCGGAOIIdW68DzRsHgY4Tk4MVT/SiVYF7ggaXjam2geAIjEGfS4LA96CDEDwG27gw6qAGGAAoA9R6T1npozIQQIgxDsHGfwQQigVDqDg7oHRBgjAgFMOYfQhGbiQHmKgRINQbgcD%2BKWeTBA/j0BQ7QEMXh/hOHqMQYAmB1PIB2CGBQTFVIGCYpgPtVQ/01AcAwZwrgWjeHyYEYIfRSgDC1kPPIaQBCjFaPkgLNQpj9BiH59oDnOgGaaK5sY%2BSrCxYEF0SYXnpi%2BcPpYeLwX3O5e6BFnzUXD7zBzUsFYEg42cETZ2%2B9Pa9gSYIMgPYy7V3rtUvmXAhBKQFq4LMItJb5haRAIsAgOwLAUFOgpBd0RQisFWM11r7W10bt4JgfARBKPoD0Lx4QohxA8dkPxtQ97hOkClMRxIw39C1aTQ1zgt4USTb2KgKgTWs0rZXWtrrEAXCTunSlA6A2htDpMqQct8QiQ6yZXHdOItHZNs4C20gbbp7cK7bwHtfaQADuGzVjg9wHtwZx6QQdWgIcoeIKkewkggA%3D%3D%3D)