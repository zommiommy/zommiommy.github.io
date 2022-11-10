+++
title = "M0lecon 2021 Rev Writeups"
date = "2021-11-10"
description = "Write up of the reverse engineering challenge of M0lecon 2021"
+++

**TLDR**
- Reverse the code
- Re-implment it in Rust
- Bruteforce
- ???
- Profit

# Automatic Rejection Machine
This is a classical flag checker but its an Aarch64 ELF.
Since qemu is a pain in the ass and I have no usable device to run the executable, we just reversed the binary and re-implemented it in Rust.

The binary basically takes the flag and apply some operations on it, and then it checks that the result matches some magic values. To solve it we just brute-force the value using the number of magic values matched as a """""side-channel"""".
```rust
use std::collections::HashMap;

const SBOX: [i16; 30] = [
    0x0, 0x2, 0xB, 0x6, 0x4, 0x5, 0x3, 0x7, 0x8, 0x9, 0xA, 0x1, 0xC, 0x16, 0x18, 0xF, 0x11, 0x10, 0x12, 0x13, 0x17, 0x14, 0xD, 0x15, 0xE, 0x1D, 0x1C, 0x1B, 0x1A, 0x19
];

fn shuffle_flag(mut flag: Vec<u8>) -> Vec<u8> {
    let mut sbox = SBOX.to_vec();
    for i in 0..0x1E {
        let mut j = i;
        while sbox[j] > -1 {
            flag.swap(i, sbox[j] as usize);
            let tmp = sbox[j];
            sbox[j] -= 0x1E;
            j = tmp as usize;
        }
    }
    flag
}

const WTF: [u64; 6] = [
    0x9e3779b917181920,
    0x9e3779b912881288,
    0xdeadbeeffeedbeef,
    0x1badb002facecafe,
    0xFEEDFACE08920892,
    0xCAFEFEED12401240,
];

fn f1(flag: Vec<u8>) -> Vec<u64> {
    let mut wtf = WTF.to_vec();
    let mut res = Vec::new();

    for i in 0..(flag.len() - 2) {
        wtf[0] = 
            (flag[i + 2] as u64) << 0x10 |
            (flag[i + 1] as u64) << 8 |
            flag[i] as u64 |
            0xaabbccdd11000000;
        let (a, b) = f2(&wtf);
        res.push(a);
        res.push(b);
    }

    res
}

fn f2(wtf: &Vec<u64>) -> (u64, u64) {
    let mut res1 = wtf[0];
    let mut res2 = wtf[1];
    let mut c = wtf[2..].to_vec();

    for i in 0..0x10 {
        c[i & 3] = 
            c[0] + c[1] + (
                c[2] + c[3] ^ c[0] << (c[2 & 0x3F])
            );
        let tmp = c[i & 3];
        res1 += (tmp + res2) * 0x200 ^ tmp - res2 ^ (tmp + res2) >> 0xe;
        res2 += (tmp + res1) * 0x200 ^ tmp - res1 ^ (tmp + res1) >> 0xe;
    }

    (res1, res2)
}

const MAGIC: [u64; 60] = [
    0xb4d8846071ac9ee5,
    0x1e1ff00814e134fe,
    0x6b198e7941b7002e,
    0xbc6fa839efe36443,
    0xc3c71ad9a664b6c3,
    0x5692a2f09c98d986,
    0xf084a1a59cd01e68,
    0xbc52e78a7e4df2df,
    0xda219d93290b91a8,
    0x5703d0286fa5d32f,
    0x6274b1b118da82b2,
    0xa746ebfb0954ebbc,
    0x5f6df7bd4f1967a2,
    0x16d5b5bdee98cf8e,
    0x52e8b6df7e62e39a,
    0x99f9455fb0c8d933,
    0x5ffd82d53af933d,
    0xff9084a16ff0141c,
    0xe17c5f0781d52f9b,
    0x1a0f4431548e51d1,
    0xf2e8573d8f0f01dd,
    0x250039177f4def91,
    0x8851491ecbc7af7c,
    0xad427c6695b91d24,
    0x5e0071d97d98d094,
    0x264dda52b0c37b03,
    0xa5811271d6d7c428,
    0xe0133fc719f34136,
    0xe508ace2412b2633,
    0x74321a3e9face34c,
    0xff5b8a59e8ebf70b,
    0x76275a516f88c986,
    0x1604d76f74599cc4,
    0xf744bcd8f2016f58,
    0xa0b6a7a0239e4ea7,
    0xf1efc57f15cb9ab4,
    0xb0d1ad4fb4ed946a,
    0x81ca31324d48e689,
    0xe6a9979c51869f49,
    0xa666637ee4bc2457,
    0x6475b6ab4884b93c,
    0x5c033b1207da898f,
    0xb66dc7e0dec3443e,
    0xe4899c99cfa0235c,
    0x3b7fd8d4d0dcaf6b,
    0xb1a4690db34a7a7c,
    0x8041d2607129adab,
    0xa6a1294a99894f1a,
    0xdde37a1c4524b831,
    0x3bc8d81de355b65c,
    0x6c61ab15a63ad91e,
    0x8fa4e37f4a3c7a39,
    0x268b598404e773af,
    0x74f4f040ae13f867,
    0x4df78e91fd682404,
    0xabe1fc425a9a671a,
    0x1bb06615c8a31dd5,
    0x9f56e9aef2fa5d55,
    0x239dcf030b3ce09b,
    0x24556a34b61ca99,
];

const INV_SBOX: [usize; 30] = [0, 11, 1, 6, 4, 5, 3, 7, 8, 9, 10, 2, 12, 22, 24, 15, 17, 16, 18, 19, 21, 23, 13, 20, 14, 29, 28, 27, 26, 25];


fn main() {
    // start from a plausible input this is important because the f1 function
    // depends on the next 3 chars in the flag (once shuffled) so we need the `p` and the `t`
    // to be already right otherwise the bruteforcing won't work.
    let mut input = b"ptm{EFGHIJKLMNOPQRSTUVWXYZabc}".to_vec();
    
    // for each carachter
    for i in 1..30 {
        let mut tests = HashMap::new();
        
        // bruteforce the next carachter
        for c in "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!\"#$%&\\'()*+,-./:;<=>?@[\\]^_`{|}~".bytes() {
            // set the current carachter 
            input[INV_SBOX[i]] = c;
            
            // simlate the binary
            let clone_input = input.clone();
            let flag = shuffle_flag(input.clone());
            let flag = f1(flag);
            
            // count how many magic values the current input matches
            let mut count = 0;
            for i in 0..56 {
                if flag[i] == MAGIC[i] {
                    count += 1;
                }
            }
            
            // save it into the possible solutions map
            tests.insert(clone_input, count);
        }
        
        // extract the input which matches the most magics
        let (new_input, count) = tests.iter().max_by(|(_flag1, count1), (_flag2, count2)| count1.cmp(count2)).unwrap();
        
        // print it and use it for the next iteration
        println!("{} {}", String::from_utf8(new_input.clone()).unwrap(), count);
        input = new_input.clone();
        
    }
}
```
Which give the result:
```
$ time cargo run --release
ptm{EFGHIJKuMNOPQRSTUVWXYZabc} 2
ptm{EFGHIJKuMNOPQRSTUVWXYZabc} 2
ptm{EF0HIJKuMNOPQRSTUVWXYZabc} 4
ptm{5F0HIJKuMNOPQRSTUVWXYZabc} 6
ptm{5m0HIJKuMNOPQRSTUVWXYZabc} 10
ptm{5m0HIJKuMNOPQRSTUVWXYZabc} 10
ptm{5m0lIJKuMNOPQRSTUVWXYZabc} 12
ptm{5m0l_JKuMNOPQRSTUVWXYZabc} 14
ptm{5m0l_cKuMNOPQRSTUVWXYZabc} 16
ptm{5m0l_chuMNOPQRSTUVWXYZabc} 20
ptm{5m0l_chuMNOPQRSTUVWXYZabc} 20
ptm{5m0l_chunNOPQRSTUVWXYZabc} 22
ptm{5m0l_chunNOPQRSTUV3XYZabc} 24
ptm{5m0l_chunNOPQRSTUV3XuZabc} 26
ptm{5m0l_chunNO_QRSTUV3XuZabc} 28
ptm{5m0l_chunNO_QmSTUV3XuZabc} 30
ptm{5m0l_chunNO_5mSTUV3XuZabc} 32
ptm{5m0l_chunNO_5m0TUV3XuZabc} 34
ptm{5m0l_chunNO_5m0lUV3XuZabc} 36
ptm{5m0l_chunNO_5m0lU53XuZabc} 38
ptm{5m0l_chunNO_5m0lU53cuZabc} 40
ptm{5m0l_chunkO_5m0lU53cuZabc} 42
ptm{5m0l_chunkO_5m0l_53cuZabc} 44
ptm{5m0l_chunk5_5m0l_53cuZabc} 48
ptm{5m0l_chunk5_5m0l_53cuZabc} 48
ptm{5m0l_chunk5_5m0l_53cuZaby} 50
ptm{5m0l_chunk5_5m0l_53cuZa7y} 52
ptm{5m0l_chunk5_5m0l_53cuZ17y} 54
ptm{5m0l_chunk5_5m0l_53cur17y} 56
cargo run --release  0.07s user 0.01s system 98% cpu 0.073 total
```

# Parallel-The-M0le
This programs is a flag-hasher that spawns 14 threadss and apply operations to the flag (in a random order), it print the results, then it re-apply these operation in a slightly different order to a new copy of the flag and print it.

In particular if at the first step it executes the functions:
```
result1 = f1 f2 f3 f4 f5 f6 f7 f8 f9 f10 f11 f12 f13 f14
```
then the second step it execute:
```
result2 = f1 f2 f3 f4 f5 f6 f14 f13 f12 f11 f10 f9 f8 f7
```

Since the functions are all injective, we can write the inverse and bruteforce the sequence of 7 functions that applied in a reverse order on result1 and result2 produces the same result.

Once we found this ""collision"" we can bruteforce the last 7 functions until we find a string that starts with `ptm{`.
```rust
// All the functions reversed from the binary, and their inverse.

fn func1(mut flag: Vec<u8>) -> Vec<u8> {
    let mut buffer = vec![0, 0, 0, 0];
    let mut i  = 0;
    while i < flag.len() {
        for j in 0..4 {
            buffer[j] = flag[j + i];
        }

        buffer.swap(3, 0);
        buffer.swap(1, 0);

        for j in 0..4 {
            flag[j + i] = buffer[j];
        }
        i += 4;
    }
    flag
}

fn func1_inv(mut flag: Vec<u8>) -> Vec<u8> {
    let mut buffer = vec![0, 0, 0, 0];
    let mut i  = 0;
    while i < flag.len() {
        for j in 0..4 {
            buffer[j] = flag[j + i];
        }

        buffer.swap(1, 0);
        buffer.swap(3, 0);

        for j in 0..4 {
            flag[j + i] = buffer[j];
        }
        i += 4;
    }
    flag
}

fn func2(mut flag: Vec<u8>) -> Vec<u8> {
    let l = flag.len();
    for i in 0..(l / 2) {
        flag[i] ^= flag[l - i - 1];
    }
    flag
}

fn func2_inv(mut flag: Vec<u8>) -> Vec<u8> {
    func2(flag)
}

fn func3(mut flag: Vec<u8>) -> Vec<u8> {
    for i in 0..16 {
        let t = i & 7;
        flag[i] = flag[i] << t | flag[i] >> (8 - t);
    }
    flag
}

fn func3_inv(mut flag: Vec<u8>) -> Vec<u8> {
    func3(flag)
}


fn func4(mut flag: Vec<u8>) -> Vec<u8> {
    let l = flag.len();
    for i in 0..(l / 2) {
        flag[l - i - 1] ^= flag[i];
    }
    flag
}

fn func4_inv(mut flag: Vec<u8>) -> Vec<u8> {
    func4(flag)
}

fn func5(mut flag: Vec<u8>) -> Vec<u8> {
    let buffer = b"{reverse_fake_flag}ptm".to_vec();
    for i in 0..16 {
        flag[i] ^= buffer[i];
    }
    flag
}

fn func5_inv(mut flag: Vec<u8>) -> Vec<u8> {
    func5(flag)
}

fn func6(mut flag: Vec<u8>) -> Vec<u8> {
    for i in 0..16 {
        flag[i] = !flag[i];
    }
    flag
}

fn func6_inv(mut flag: Vec<u8>) -> Vec<u8> {
    func6(flag)
}

fn func7(mut flag: Vec<u8>) -> Vec<u8> {
    for i in 0..16 {
        let mut val = flag[i];
        let mut j = 7;
        let mut tmp = val >> 1;
        while tmp != 0 {
            val = val << 1 | tmp & 1;
            tmp = tmp >> 1;
            j -= 1;
        }
        flag[i] = val << (j & 0x1F);
    }
    flag
}

fn func7_inv(mut flag: Vec<u8>) -> Vec<u8> {
    func7(flag)
}

fn func8(mut flag: Vec<u8>) -> Vec<u8> {
    flag.reverse();
    flag
}

fn func8_inv(mut flag: Vec<u8>) -> Vec<u8> {
    func8(flag)
}

fn func9(mut flag: Vec<u8>) -> Vec<u8> {
    for i in 0..16 {
        flag[i] += 42;
    }
    flag
}

fn func9_inv(mut flag: Vec<u8>) -> Vec<u8> {
    for i in 0..16 {
        flag[i] -= 42;
    }
    flag
}

fn func10(mut flag: Vec<u8>) -> Vec<u8> {
    for i in 0..16 {
        flag[i] = flag[i] >> 4 & 0xf | flag[i] << 4;
    }
    flag
}

fn func10_inv(mut flag: Vec<u8>) -> Vec<u8> {
    func10(flag)
}


fn func11(mut flag: Vec<u8>) -> Vec<u8> {
    for i in 0..16 {
        flag[i] = (
                ((flag[i] & 1) << 4) 
                | flag[i] * 2 & 0x80
                | ((flag[i] & 0x10) << 2)
                | ((flag[i] & 0x4) << 3)
            ) >> 4
            | ((flag[i] & 0x2) << 3) 
            | flag[i] & 0x80
            | flag[i] * 2 & 0x40
            | ((flag[i] & 0x8) << 2);
    }
    flag
}

// we are lazy so to invert the Func11 we just save every input-output pair
const FUNC11MAP: [u8; 256] = [0, 1, 4, 5, 16, 17, 20, 21, 64, 65, 68, 69, 80, 81, 84, 85, 2, 3, 6, 7, 18, 19, 22, 23, 66, 67, 70, 71, 82, 83, 86, 87, 8, 9, 12, 13, 24, 25, 28, 29, 72, 73, 76, 77, 88, 89, 92, 93, 10, 11, 14, 15, 26, 27, 30, 31, 74, 75, 78, 79, 90, 91, 94, 95, 32, 33, 36, 37, 48, 49, 52, 53, 96, 97, 100, 101, 112, 113, 116, 117, 34, 35, 38, 39, 50, 51, 54, 55, 98, 99, 102, 103, 114, 115, 118, 119, 40, 41, 44, 45, 56, 57, 60, 61, 104, 105, 108, 109, 120, 121, 124, 125, 42, 43, 46, 47, 58, 59, 62, 63, 106, 107, 110, 111, 122, 123, 126, 127, 128, 129, 132, 133, 144, 145, 148, 149, 192, 193, 196, 197, 208, 209, 212, 213, 130, 131, 134, 135, 146, 147, 150, 151, 194, 195, 198, 199, 210, 211, 214, 215, 136, 137, 140, 141, 152, 153, 156, 157, 200, 201, 204, 205, 216, 217, 220, 221, 138, 139, 142, 143, 154, 155, 158, 159, 202, 203, 206, 207, 218, 219, 222, 223, 160, 161, 164, 165, 176, 177, 180, 181, 224, 225, 228, 229, 240, 241, 244, 245, 162, 163, 166, 167, 178, 179, 182, 183, 226, 227, 230, 231, 242, 243, 246, 247, 168, 169, 172, 173, 184, 185, 188, 189, 232, 233, 236, 237, 248, 249, 252, 253, 170, 171, 174, 175, 186, 187, 190, 191, 234, 235, 238, 239, 250, 251, 254, 255];

fn func11_inv(mut flag: Vec<u8>) -> Vec<u8> {
    for i in 0..16 {
        flag[i] = FUNC11MAP[flag[i] as usize];
    }
    flag
}

fn func12(mut flag: Vec<u8>) -> Vec<u8> {
    for i in 0..16 {
        flag[i] = !(flag[i] ^ i as u8);
    }
    flag
}

fn func12_inv(mut flag: Vec<u8>) -> Vec<u8> {
    func12(flag)
}

fn func13(mut flag: Vec<u8>) -> Vec<u8> {
    for i in 0..16 {
        flag[i] += i as u8;
    }
    flag
}

fn func13_inv(mut flag: Vec<u8>) -> Vec<u8> {
    for i in 0..16 {
        flag[i] -= i as u8;
    }
    flag
}

fn func14(mut flag: Vec<u8>) -> Vec<u8> {
    for i in 0..16 {
        if (flag[i] < b'A') || (b'Z' < flag[i]) {
            if (b'`' < flag[i]) && (flag[i] < b'{') {
                flag[i] = flag[i] - 0x20;
            }
        } else {
            flag[i] = flag[i] + 0x20;
        }
    }
    flag
}

fn func14_inv(mut flag: Vec<u8>) -> Vec<u8> {
    func14(flag)
}


const INV_FUNCTIONS: [fn(Vec<u8>) -> Vec<u8>; 14] = [
    func1_inv,
    func2_inv,
    func3_inv,
    func4_inv,
    func5_inv,
    func6_inv,
    func7_inv,
    func8_inv,
    func9_inv,
    func10_inv,
    func11_inv,
    func12_inv,
    func13_inv,
    func14_inv
];
// the two results that are used to bruteforce the flag
const RESULT1: [u8; 16] = [
    42,
    173,
    46,
    90,
    73,
    251,
    45,
    154,
    219,
    144,
    141,
    208,
    14,
    180,
    140,
    138,
];
const RESULT2: [u8; 16] = [
    102,
    7,
    171,
    97,
    159,
    117,
    176,
    39,
    47,
    60,
    30,
    179,
    63,
    233,
    237,
    175,
];

// find the sequence of seven functions that in opposite order yields the same output using result1 and result2 as inputs
fn brute_step_1() -> Vec<u8> {
    for i1 in 0..14 {
        for i2 in 0..14 {
            for i3 in 0..14 {
                for i4 in 0..14 {
                    for i5 in 0..14 {
                        for i6 in 0..14 {
                            for i7 in 0..14 {
                                let mut x: u64 = 0;
                                x |= 1 << i1;
                                x |= 1 << i2;
                                x |= 1 << i3;
                                x |= 1 << i4;
                                x |= 1 << i5;
                                x |= 1 << i6;
                                x |= 1 << i7;
                                if x.count_ones() != 7 {
                                    continue;
                                }

                                let r1 = INV_FUNCTIONS[i1](RESULT1.to_vec());
                                let r1 = INV_FUNCTIONS[i2](r1);
                                let r1 = INV_FUNCTIONS[i3](r1);
                                let r1 = INV_FUNCTIONS[i4](r1);
                                let r1 = INV_FUNCTIONS[i5](r1);
                                let r1 = INV_FUNCTIONS[i6](r1);
                                let r1 = INV_FUNCTIONS[i7](r1);

                                let r2 = INV_FUNCTIONS[i7](RESULT2.to_vec());
                                let r2 = INV_FUNCTIONS[i6](r2);
                                let r2 = INV_FUNCTIONS[i5](r2);
                                let r2 = INV_FUNCTIONS[i4](r2);
                                let r2 = INV_FUNCTIONS[i3](r2);
                                let r2 = INV_FUNCTIONS[i2](r2);
                                let r2 = INV_FUNCTIONS[i1](r2);

                                if r1 == r2 {
                                    println!("{:?}", r1);
                                    println!("{:?}", vec![i1, i2, i3, i4, i5, i6, i7]);
                                    
                                    return r1;
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    panic!("Brute force step 1 failed");
}

// bruteforce the remaining sequence of 7 functions that yields an ouput that starts with `ptm{`
fn brute_step_2(reference: Vec<u8>){
    for i1 in 0..14 {
        for i2 in 0..14 {
            for i3 in 0..14 {
                for i4 in 0..14 {
                    for i5 in 0..14 {
                        for i6 in 0..14 {
                            for i7 in 0..14 {
                                let mut x: u64 = 0;
                                x |= 1 << i1;
                                x |= 1 << i2;
                                x |= 1 << i3;
                                x |= 1 << i4;
                                x |= 1 << i5;
                                x |= 1 << i6;
                                x |= 1 << i7;
                                if x.count_ones() != 7 {
                                    continue;
                                }

                                let r1 = INV_FUNCTIONS[i1](reference.to_vec());
                                let r1 = INV_FUNCTIONS[i2](r1);
                                let r1 = INV_FUNCTIONS[i3](r1);
                                let r1 = INV_FUNCTIONS[i4](r1);
                                let r1 = INV_FUNCTIONS[i5](r1);
                                let r1 = INV_FUNCTIONS[i6](r1);
                                let r1 = INV_FUNCTIONS[i7](r1);

                                if r1.starts_with(b"ptm{") {
                                    for x in r1 {
                                        print!("{}", x as char);
                                    }
                                    print!("\n");
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

fn main() {
    // check that the setup is sane
    let r1 = RESULT1.to_vec();
    assert_eq!(r1, func1_inv(func1(r1.clone())));
    assert_eq!(r1, func2_inv(func2(r1.clone())));
    assert_eq!(r1, func3_inv(func3(r1.clone())));
    assert_eq!(r1, func4_inv(func4(r1.clone())));
    assert_eq!(r1, func5_inv(func5(r1.clone())));
    assert_eq!(r1, func6_inv(func6(r1.clone())));
    assert_eq!(r1, func7_inv(func7(r1.clone())));
    assert_eq!(r1, func8_inv(func8(r1.clone())));
    assert_eq!(r1, func9_inv(func9(r1.clone())));
    assert_eq!(r1, func10_inv(func10(r1.clone())));
    assert_eq!(r1, func11_inv(func11(r1.clone())));
    assert_eq!(r1, func12_inv(func12(r1.clone())));
    assert_eq!(r1, func13_inv(func13(r1.clone())));
    assert_eq!(r1, func14_inv(func14(r1.clone())));
    
    // run the bruteforce
    brute_step_2(brute_step_1());
}
```
Which gives the result:
```
$ time cargo run --release
[159, 91, 170, 77, 137, 115, 185, 199, 151, 69, 59, 246, 125, 88, 183, 111]
[7, 8, 13, 3, 9, 11, 6]
ptm{brut¸Õë­öñ¸ò
ptm{brut¸Õë­öñ¸ò
ptm{brut¸Õë­öñ¸ò
ptm{brut¸Õë­öñ¸ò
ptm{brut3_f0rc3}
ptm{brut3_f0rc3}
ptm{brut¸Õë­öñ¸ò
ptm{brut¸Õë­öñ¸ò
ptm{brut¸Õë­öñ¸ò
ptm{brut¸Õë­öñ¸ò
ptm{brut3_f0rc3}
ptm{brut¸Õë­öñ¸ò
ptm{brutÌ ÏÌ
ptm{brutÌ ÏÌ
ptm{brutÌ ÏÌ
ptm{brut3_f0rc3}
ptm{brut3_f0rc3}
ptm{brut3_f0rc3}
ptm{brut¸Õë­öñ¸ò
ptm{brut¸Õë­öñ¸ò
ptm{brut3_f0rc3}
ptm{brut¸Õë­öñ¸ò
ptm{brutÌ ÏÌ
ptm{brut3_f0rc3}
ptm{brutÌ ÏÌ
ptm{brutÌ ÏÌ
ptm{brutÌ ÏÌ
ptm{brutÌ ÏÌ
ptm{brutÌ ÏÌ
ptm{brutÌ ÏÌ
ptm{brutÌ ÏÌ
ptm{brutÌ ÏÌ
ptm{brutÌ ÏÌ
ptm{brutÌ ÏÌ
cargo run --release  5.36s user 0.01s system 99% cpu 5.390 total
```