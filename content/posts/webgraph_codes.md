+++
title = "Webgraph from scratch: Instantaneous Codes (Draft)"
description = "Everything You Always Wanted to Know About Instantaneous Codes* (*But Were Afraid to Ask)"
date = 2023-09-24
+++
# Introduction
This post is a collection of notes I took and benchmarks I run while implementing and optimizing [dsi-bitstream-rs](https://github.com/vigna/dsi-bitstream-rs).

# Theoretical preliminaries
All bit sequences will be read from left to right, from top to bottom.
For ease of reading I will be using the convention that **all codes start from 0** 
by adding a +1 where needed. This differs from the **book definitions** of the codes,
but it's how you practically implement them.

## Information

Information is define by the following axioms:
- The infomration \\(I(p)\\) of an event with probaility \\(p\\) is a monotone decreasing function of \\(p\\), i.e. \\(p \le q \implies I(p) \ge I(q)\\)
- \\(I(1) = 0\\): The information content of a certain event is 0
- Information is additive, i.e. the information of two independent events is the sum of the information of each event. \\(I(p \cup q) = I(p) + I(q)\\)

The only function that satisfies these axioms is:
\\[I(p) = -\log\_2 p\\]

#### Example
A simple way to derive the information function for a discrete uniform distribution is the following:

Using \\(n\\) bits we can store \\(2^n\\) different values.
We can invert this relationship as: a set of \\(N = 2^n\\) values with uniform probability has \\(I(p) = \log_2 N = n\\) bits of information.

*We use uniform probability because we don't know anything about the data.*

By definition, each value in the set has probability \\(p = \frac{1}{N}\\) so \\(N = \frac{1}{p}\\),
so we can rewrite the information as:
\\[I(p) = -\log\_2 \frac{1}{p} = -\log\_2 p\\] 

which is exactly the definition of Information.

### [Entropy](https://en.wikipedia.org/wiki/Entropy_(information_theory))

*The entropy is the expected information.*

Given a probability distribution \\(p: S \to [0, 1]\\) with support \\(S\\), the entropy of \\(p\\) in bits is defined as:
\\[\mathbb{H}(p) = -\sum\_{s \in S} p(s) \log\_2 p(s) = \mathbb{E}\_p[-\log\_2 p]\\]

The entropy is a measure of the uncertainty of a random variable, it's maximized when all the values are equally likely and minimized when the distribution is a delta function i.e. one value has probability 1 while the other have probability 0.

**For our use cases, the entropy is the minimum number of bits needed to store a value extracted from a distribution.**

## [Instantaneous code](https://en.wikipedia.org/wiki/Prefix_code)
An instantaneous (binary) code (a.k.a. prefix-free code) for the set \\(S\\) is a function \\(c : S \to \{0, 1\}^*\\) such that, for all \\(x, y \in S\\), if \\(c(x)\\) is a prefix of \\(c(y)\\), then \\(x = y\\).

*This implies that there is one unique way of decoding a given sequence*

[**Kraft-McMillan**](https://en.wikipedia.org/wiki/Kraft%E2%80%93McMillan_inequality) tell us that there exists an instantaneous code with lengths \\(|c(s)|\\) (\\(s \in S\\)) if, and only if:
\\[\sum_{s \in S} 2^{-|c(s)|} \\le 1\\] 

where \\(l_s = |c(s)|\\) is the length in bits of the code \\(c(s)\\) for value \\(s \in S\\).

A **complete** code is an instantaneous code where:
\\[\sum_{s \in S} 2^{-l_s} = 1\\] 

*A complete code means that there are no unused codewords.*

Furthermore, the expected length of \\(c\\) with respect to some probability
distribution \\(p: S \to [0, 1]\\) with support \\(S\\) is:

\\[\mathbb{E}\_p[|c|] = \sum\_\{s \in S\} p(s) |c(s)|\\]

it follows that the **optimal** code \\(c^*_p\\) for a givend probability distribution \\(P\\) is the instantaneous code with minimum expected length.

If \\(S\\) is finite, Shannon's coding theorem tells us that the minimum expected length is the entropy of the distribution \\(\mathbb{H}[p]\\): 
\\[\mathbb{E}_p[|c^*_p|] = \mathbb{H}(p)\\]

moreover, the Huffman encoding is **optimal**.

### Intended distribution
Using the same equation for the expected lenght we can also compute the **intended distribution** i.e. the probability \\(p\\) that minimize the expected length \\(\mathbb{E}_p[|c|]\\) for a given code \\(c\\).

We can derive the intended distribution starting from:
\\[\mathbb{E}\_p[|c^*\_p|] = \mathbb{H}(p)\\]
by applying the respective defenitions:
\\[\sum\_{s \ in S} p(s) |c(s)| = \sum\_{s \in S} -p(s) \log\_2 p(s)\\]
a way to make this holds is to impose that the inner terms of the sum must be equal:
\\[\forall s \in S \quad p(s) |c(s)| = -p(s) \log\_2 p(s)\\]
we can factor out the common \\(p(s)\\) term (assuming it's not 0, in which case the equation still holds):
\\[\forall s \quad |c(s)| = -\log\_2 p(s)\\] 
and finally we can derive \\(p(s)\\):
\\[\forall s \quad p(s) = 2^{-|c(x)|}\\]

Thus, the intended distribution for \\(c\\) is:
\\[p(x) = 2^{-|c(x)|}\\]

*Note that this is a proper distribution if and only if the code is complete, otherwise it won't sum to 1 and we would need some renormalization constant.*

Therefore, to achieve best compression, **we have to choose the code which intended distribution most closely match the data distribution**.

## [Universal code](https://en.wikipedia.org/wiki/Universal_code_(data_compression))

# Codes
### Fixed length
It's defined on a the set of values beween \\(0\\) and \\(N\\) (excluded) i.e. \\(S = {0, 1, ..., N - 1}\\), it reresents each element \\(s \in S\\) using \\(\forall s \in S \quad |c(s)| = \lceil \log_2 N \rceil\\) bits.

Thus, the fixed length code is optimal for uniform distributions of values in \\(\left[0, N\right)\\) if, and only if, \\(N\\) is a power of two. 

#### Example for N = 4
<table>
<thead>
  <tr><th>Number</th><th>Code</th></tr>
</thead>
<tbody>
  <tr><td>0</td><td>00</td></tr>
  <tr><td>1</td><td>01</td></tr>
  <tr><td>2</td><td>10</td></tr>
  <tr><td>3</td><td>11</td></tr>
</tbody>
</table>

*Note that any other permutation of bits is also valid.*

#### Example for N = 8
<table>
<thead>
  <tr><th>Number</th><th>Code</th></tr>
</thead>
<tbody>
  <tr><td>0</td><td>000</td></tr>
  <tr><td>1</td><td>001</td></tr>
  <tr><td>2</td><td>010</td></tr>
  <tr><td>3</td><td>011</td></tr>
  <tr><td>4</td><td>100</td></tr>
  <tr><td>5</td><td>101</td></tr>
  <tr><td>6</td><td>110</td></tr>
  <tr><td>7</td><td>111</td></tr>
</tbody>
</table>

### [Truncated Binary Encoding](https://en.wikipedia.org/wiki/Truncated_binary_encoding)
also known as minimal binary encoding, us used for uniform distributions where \\(N\\) is not a power of two.

The core idea is that we can write with \\(\lfloor \log_2 N \rfloor\\) bits the first \\(2^{\lfloor \log_2 N \rfloor}\\) values i.e. the up to the biggest power of two smaller than \\(N\\), and the remaining \\(N - 2^{\lfloor \log_2 N \rfloor}\\) values using \\(\lceil \log_2 N \rceil\\) bits.

#### Example for N = 6
<table>
<thead>
  <tr><th>Number</th><th>Code</th></tr>
</thead>
<tbody>
  <tr><td>0</td><td>00</td></tr>
  <tr><td>1</td><td>01</td></tr>
  <tr><td>2</td><td>10</td></tr>
  <tr><td>3</td><td>11</td></tr>
  <tr><td>4</td><td>100</td></tr>
  <tr><td>5</td><td>101</td></tr>
</tbody>
</table>

**Note: this is not instantaneous! 2 is a prefix of 4 and 5! We will deal with this later.**

This is **close to be optimal**, as the expected code length is:
\\[\mathbb{E}[|c|] = \frac{2^{\lfloor \log_2 N \rfloor}}{N} \lfloor \log_2 N \rfloor \\; + \\; \left(1 - \frac{2^{\lfloor \log_2 N \rfloor}}{N} \right) \lceil \log_2 N \rceil \\]

while the entropy is:
\\[\mathbb{H}[p] = \sum_{s \in S} -\frac{1}{N} \log_2 \frac{1}{N} = \log_2 N\\]

If we define \\(\alpha = \frac{2^{\lfloor \log_2 N \rfloor}}{N}\\) we can notice that
the expected length is the convex combination of \\(\lfloor \log_2 N \rfloor\\) and \\(\lceil \log_2 N \rceil\\):

\\[\mathbb{E}[|c|] = \alpha \lfloor \log_2 N \rfloor \\; + \\; \left(1 - \\alpha \right) \lceil \log_2 N \rceil \\]

When \\(N\\) is a power of two this is equivalent to the fixed length and is optimal because:
\\[\mathbb{E}[|c|] = \lfloor \log_2 N \rfloor = \log_2 N = \lceil \log_2 N \rceil\\]

Otherwise, by definition, a convex combination will only take cavlues between its
extremes \\(\lfloor \log_2 N \rfloor\\) and \\(\lceil \log_2 N \rceil\\) so the
truncated binary encoding is always smaller or equal than the fixed length.

### Unary
We can write any positive number \\(s \in \mathbb{N} \\) as a sequence of \\(s\\) zeros followed by a one, which act as a terminator.

\\[s \to \overbrace{00...00}^\text{s} 1 = 0^s 1\\]

<table>
<thead>
  <tr><th>Number</th><th>Code</th></tr>
</thead>
<tbody>
  <tr><td>0</td><td>1</td></tr>
  <tr><td>1</td><td>01</td></tr>
  <tr><td>2</td><td>001</td></tr>
  <tr><td>3</td><td>0001</td></tr>
  <tr><td>4</td><td>00001</td></tr>
  <tr><td>5</td><td>000001</td></tr>
  <tr><td>6</td><td>0000001</td></tr>
  <tr><td>7</td><td>00000001</td></tr>
  <tr><td>8</td><td>000000001</td></tr>
</tbody>
</table>

This code has length \\(s + 1\\) therefore, its intended distribution is:
\\[p(x) = \left(\frac{1}{2}\right)^{s + 1}\\]  

Thus, it's optimal for the [geometric distribution](https://en.wikipedia.org/wiki/Geometric_distribution) with \\(p = \frac{1}{2}\\).

Most modern CPUs architectures, like x86_64 and Aarch64 (Arm), have instructions
to compute the leading or traling zeros in a registers. 

x86_64 has [LZCNT](https://www.felixcloutier.com/x86/lzcnt) and [TZCNT](https://www.felixcloutier.com/x86/tzcnt) instructions, while Aarch64 has only [CLZ](https://developer.arm.com/documentation/dui0802/b/A32-and-T32-Instructions/CLZ).

Using these we can decode unary codes in 1 ns on average. 

```rust
fn read_unary(&mut self) -> u64 {
    let value = self.peek_u64();
    let result = value.leading_zeros(); // let rust do the magic
    self.skip_bits(result + 1);
    result
}

fn write_unary(&mut self, value: u64)  {
    let bits = 1 << value;
    let n_bits = value + 1;
    self.write_bits(bits, n_bits);
}
```

### [Elias-Gamma](https://en.wikipedia.org/wiki/Elias_gamma_coding#Further_reading)

The idea behind Elias-Gamma is to write a number \\(s\\) in binary using \\(l = \lceil \log_2 (s + 1) \rceil\\) bits and prefixing it with the length \\(l\\) in unary.

Moreover, if we encode \\(s + 1\\) instead of \\(s\\), the fixed length will always
have at least a 1,so we can save 1 bit by using
the unary terminator 1 as the highest bit of the fixed length.

So the final Elias-Gamma encodes a value \\(s \in \mathbb{N}\\) by writing \\(s + 1\\) in binary using \\(l = \lceil \log_2 (s + 1) \rceil\\) bits and prefixing it with the length \\(l\\) in unary.

Or equivalently, it encodes a value \\(s \in \mathbb{N}\\) writing \\(\lfloor \log_2 (s + 1) \rfloor\\) in unary and then \\(s + 1 - 2^{\lceil \log_2 (s + 1) \rceil}\\) in binary, using \\(\lfloor \log_2 (s + 1) \rfloor\\) bits.

\\[s \to \text{Unary}\left(\lfloor \log_2 (s + 1) \rfloor\right) \text{Binary}\left(s + 1 - 2^{\lceil \log_2 (s + 1) \rceil}\right)\\]

Therefore, the length of the gamma code for \\(s\\) is \\(1 + 2\lfloor \log_2 (s + 1) \rfloor\\).

Finally, it's intended distribution is:

\\[p(s) \propto 2^{-1-2\lfloor \log_2 (s + 1) \rfloor} \approx \frac{1}{2(s + 1)^2}\\]

<table>
<thead>
  <tr><th>Number</th><th>Gamma Code</th></tr>
</thead>
<tbody>
  <tr><td>0</td><td>1</td></tr>
  <tr><td>1</td><td>01 0</td></tr>
  <tr><td>2</td><td>01 1</td></tr>
  <tr><td>3</td><td>001 00</td></tr>
  <tr><td>4</td><td>001 01</td></tr>
  <tr><td>5</td><td>001 10</td></tr>
  <tr><td>6</td><td>001 11</td></tr>
  <tr><td>7</td><td>0001 000</td></tr>
  <tr><td>8</td><td>0001 001</td></tr>
</tbody>
</table>

While the math is a bit complicated, the code is really simple to implement:

```rust
fn read_gamma(&mut self) -> u64 {
    let len = self.read_unary();
    Ok(self.read_bits(len as usize) + (1 << len) - 1)
}

fn write_gamma(&mut self, mut value: u64)  {
    value += 1;
    let number_of_bits_to_write = value.ilog2(); // floor(log2(value))
    let short_value = value - (1 << number_of_bits_to_write);
    self.write_unary(number_of_bits_to_write);
    self.write_bits(short_value, number_of_bits_to_write);
}
```

### [Elias-Delta](https://en.wikipedia.org/wiki/Elias_delta_coding)

Elias-Delta just writes the legth of the gamma code in gamma code and then the value in binary.

[![](https://imgs.xkcd.com/comics/functional.png)](https://xkcd.com/1270/)

\\[s \to \text{Gamma}\left(\lfloor \log_2 (s + 1) \rfloor\right) \text{Binary}\left(s + 1 - 2^{\lceil \log_2 (s + 1) \rceil}\right)\\]

Therefore, its length is \\(1 + 2\lfloor \log_2 \log_2 (s + 1) \rfloor + \lfloor \log (s + 1) \rfloor\\).

Finally, it's intended distribution is:

\\[p(s) \propto 2^{-1 + 2\lfloor \log_2 \log_2 (s + 1) \rfloor + \lfloor \log (s + 1) \rfloor} \approx \\frac{1}{2(s + 1) (\log_2(s + 1))^2}\\]

<table>
<thead>
  <tr><th>Number</th><th>Code</th></tr>
</thead>
<tbody>
  <tr><td>0</td><td>1</td></tr>
  <tr><td>1</td><td>01 0 0</td></tr>
  <tr><td>2</td><td>01 1 1</td></tr>
  <tr><td>3</td><td>01 10 0</td></tr>
  <tr><td>4</td><td>01 10 1</td></tr>
  <tr><td>5</td><td>01 11 0</td></tr>
  <tr><td>6</td><td>01 11 1</td></tr>
  <tr><td>7</td><td>0001 00 000</td></tr>
  <tr><td>8</td><td>0001 00 001</td></tr>
</tbody>
</table>

While the math is a bit complicated, the code is really simple to implement:

```rust
fn read_delta(&mut self) -> u64 {
    let len = self.read_gamma();
    Ok(self.read_bits(len as usize) + (1 << len) - 1)
}

fn write_delta(&mut self, mut value: u64)  {
    value += 1;
    let number_of_bits_to_write = value.ilog2(); // floor(log2(value))
    let short_value = value - (1 << number_of_bits_to_write);
    self.write_gamma(number_of_bits_to_write);
    self.write_bits(short_value, number_of_bits_to_write);
}
```

### Golomb

for a given b, write \\(\lfloor \frac{s}{b} \rfloor \\) in unary and 
\\(s \mod b\\) using truncated encoding.

The length is \\(|c(s)| = \left \lfloor \frac{s}{b} \right \rfloor + \lceil \log_2 b \rceil\\) thus, its 

For a Geometric distribution of ratio \\(p\\) we can compute the \\(b\\) so that
the Golomb code is optimal:
\\[b = \left \lceil \frac{\log_2 (2 - p)}{-\log(1 - p)} \right \rceil\\]

```rust
fn read_golomb(&mut self, b: u64) -> u64 {
    let slot = self.read_unary();
    let rem = self.read_truncated_binary(b);
    Ok(slot * b + rem)
}

fn write_golomb(&mut self, mut value: u64, b: u64)  {
    let slot = value / b; 
    let rem = value % b;
    self.write_unary(slot);
    self.write_truncated_binary(rem, b);
}
```

### VBytes

This code, in its variation LEB128 is really common as it's present in both
DWARF format and Webassembly binary format.

Byte-aligned 

write 7 bits and use the highest to set 0 if continue 1 if stop.

The faster variation is to have all the 0s at the start, so basically writing the number of bytes - 1 in unary at the start.

LEB128 is not complete and redundant as it has infinite encodings for each number
as we can just add 0 padding on the top. Moreover, if a value uses 2 bytes we know that it must be bigger than 127, so we can subtract it to fit more values, and this
can be done recursively.

### [Zeta](https://vigna.di.unimi.it/ftp/papers/Codes.pdf)

Unary + Truncated Encoding

```python
from math import floor, ceil, log2

def write_zeta(value, k):
    value += 1
    h = int(floor(log2(value)) / k)
    u = 2**((h + 1) * k)
    l = 2**(h * k)

    data  = write_unary(h)
    data += write_minimal_binary(value - l, max=u - l)
    return data

def len_zeta(value, k):
    value += 1
    h = int(floor(log2(value)) / k)
    u = 2**((h + 1) * k)
    l = 2**(h * k)
    return len_unary(h) + len_minimal_binary(value - l, max=u - l)


def read_zeta(data, k):
    h, data = read_unary(data)
    u = 2**((h + 1) * k)
    l = 2**(h * k)
    r, data = read_minimal_binary(data, max=u - l)
    return l + r - 1, data
```

<table>
<thead>
  <tr><th>Number</th><th>Zeta1</th><th>Zeta2</th><th>Zeta3</th><th>Zeta4</th></tr>
</thead>
<tbody>
  <tr><td>0</td><td>1</td><td>1 0</td><td>1 00</td><td>1 000</td></tr>
  <tr><td>1</td><td>01 0</td><td>1 10</td><td>1 010</td><td>1 0010</td></tr>
  <tr><td>2</td><td>01 1</td><td>1 11</td><td>1 011</td><td>1 0011</td></tr>
  <tr><td>3</td><td>001 00</td><td>01 000</td><td>1 100</td><td>1 0100</td></tr>
  <tr><td>4</td><td>001 01</td><td>01 001</td><td>1 101</td><td>1 0101</td></tr>
  <tr><td>5</td><td>001 10</td><td>01 010</td><td>1 110</td><td>1 0110</td></tr>
  <tr><td>6</td><td>001 11</td><td>01 011</td><td>1 111</td><td>1 0111</td></tr>
  <tr><td>7</td><td>0001 000</td><td>01 1000</td><td>01 00000</td><td>1 1000</td></tr>
</tbody>
</table>

*Note that Zeta1 == Gamma*

## Encoding of negative numbers
All the discussed codes are defined over some subset of \\(\mathbb{N}\\), to 
encode over \\(\mathbb{Z}\\), we have to define a bijection \\(\phi: \mathbb{Z} \to \mathbb{N}\\).

If we can assume that the values form some kind of bell shape centered around zero,
as we often do, it's best to map values with small absolute value to small integers, so it's natural to use the following bijection:

\\[\phi(x) =  \left\\{\begin{matrix} 2 x & \text{if } x \ge 0\\\\- 2 x - 1 & \text{otherwise}\end{matrix}\right.\\]
which has inverse:
\\[\phi^{-1}(x) =  \left\\{\begin{matrix} x / 2 & \text{if } x = 0 \mod 2\\\\ -(x + 1) / 2 & \text{otherwise}\end{matrix}\right.\\]

<table>
<thead>
  <tr><th>\(x \in \mathbb{N}\)</th><th>\(\phi^{-1}(x) \in \mathbb{Z}\)</th></tr>
</thead>
<tbody>
  <tr><td>0</td><td>0</td></tr>
  <tr><td>1</td><td>-1</td></tr>
  <tr><td>2</td><td>1</td></tr>
  <tr><td>3</td><td>-2</td></tr>
  <tr><td>4</td><td>2</td></tr>
  <tr><td>5</td><td>-3</td></tr>
  <tr><td>6</td><td>3</td></tr>
</tbody>
</table>

this can be efficiently implemented as:
```rust
#[inline]
pub const fn int2nat(x: i64) -> u64 {
    (x << 1 ^ (x >> 63)) as u64
}

#[inline]
pub const fn nat2int(x: u64) -> i64 {
    ((x >> 1) ^ !((x & 1).wrapping_sub(1))) as i64
}
```

which in x86_64 assembly should look like:

```asm
int2nat:
        sar     rdi, 63
        xor     rax, rdi

nat2int:
        and     edi, 1
        shr     rax
        neg     rdi
        xor     rax, rdi
```

This is also called [zigzag encoding](https://en.wikipedia.org/wiki/Variable-length_quantity#Zigzag_encoding/) ([another reference](https://lemire.me/blog/2022/11/25/making-all-your-integers-positive-with-zigzag-encoding)).

# Reading codes fast

### Bitorder, Big or Little endian?
We have to choose if for us the bit 0 of a byte is the Most Significant Bit (MSB) or the Least Significant Bit (LSB). Both have different pro and cons for performance, but the main difference is that MSB is the default in Java.

### Buffering?

In this section, we'll see the core ideas we had while implementing our rust crate for reading and wring bitstreams: [`dsi-bitstream-rs`](https://github.com/vigna/dsi-bitstream-rs).

By definition, most codes we will read will take just a few bits so the idea is to have a buffer of 64 bits and read from it.
This avoids having a memory access for each read, which is a common
bottleneck. 

If we don't allow reads over 32-bits, we can use a 64-bits buffer, so we know that there will always be space for the next read, and
operations on 64-bits words are fast on most modern CPUS.

```rust
pub struct Reader<'a> {
    /// The buffer of bits we will read from
    buffer: u64,
    /// how many bits are valid in the buffer
    valid_bits: usize,
    /// The data to read from
    data: &'a [u32]
}
```

Now we have two possible convention, we can either read the bits in a byte from the Most Significant Bit to the Least Significant Bit or viceversa.

MSB to LSB implies that we will have to read the data using BigEndian order of bytes, while LSB to MSB implies LittleEndian order.

It's not clear at priori which is the best choice, but we can easily implement both and benchmark them. In the following examples we will only discuss the Big Endian (MSB to LSB) version as is the default in the Java implementation of webgraph.


Now let's implement a method for how to read the next `bits` bits from the buffer:

```rust
impl<'a> Reader<'a> {
    pub fn read_bits(&mut self, bits: usize) -> u32 {
        debug_assert!(bits <= 32);
        // if there aren't enough bits
        if self.valid_bits < bits {
            // read a new word and add it tot he buffer
            self.buffer |= (self.data[0] as u64) << self.valid_bits;
            self.valid_bits += 32;
            self.data = &self.data[1..];
        }
        // extract the lowest bits
        let res = (self.buffer as u32) & ((1 << bits as u32) - 1);
        self.buffer >>= self.valid_bits;
        self.valid_bits -= bits;
        res
    }
}
```

And now, how to read unary codes we can use rust's `leaing_zeros` which uses the `lzcnt` instruction on x86_64 and `clz` on Aarch64, if the feature is enables.
Otherwise Rust will generate the broadword code for it.

```rust
impl<'a> Reader<'a> {
    pub fn read_unary(&mut self) -> usize{
        let zeros = self.buffer.leading_zeros();
        // the unary code fits inside the valid bits
        if zeros <= self.valid_bits {
            self.buffer <<= zeros + 1;
            self.valid_bits -= zeros + 1;
            return zeros + 1
        }
        let mut res = zeros;
        
        // otherwise we have to read another word
        self.buffer = self.data[0] << 32;
        self.valid_bits += 32;
        self.data = &self.data[1..];

        let zeros = self.buffer.leading_zeros();
        // our assumption is that all unary and bits are under 32 bits
        // this is for simplicity, but it can be generalized
        debug_assert!(zeros < 32); 
        res += zeros;
        self.valid_bits -= zeros + 1;
        self.buffer <<= zeros + 1;
        res
    } 
}
```
### How big should the buffer be?
The bigger the buffer, the less memory accesses we will have, but we want the 
buffer to fit a single register, so we can do fast operations on it.

In our benchmarks we see that from u8 to u64 the speed increase is linear, but
u128 is slower than u64.

We use an u64 buffer, because u128 are not as efficient on most CPUs, expecially with `LZCNT`. Moreover, Rust/LLVM generates suboptimal code for it.

### How big should we make the tables?
What's the optimal size for tables?

### Merged or separated tables?
Merged have better cache locality and a single memory access, but they are bigger than separated ones, due to
padding.

**The answer to all these questions is to benchmark!**

# Benchmarks
These benchmarks were run on a [**Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz**](https://en.wikichip.org/wiki/intel/core_i7/i7-8750h), the csv with the raw data can be found [here for reads](/codes/read.csv) and [here for writes](/codes/write.csv).

To run these benchmarks on your own machine you can use the following command:
```bash
# Clone the repo
git clone https://github.com/vigna/dsi-bitstream-rs.git
cd dsi-bitstream-rs
# Run the read benchmarks
python3 ./python/bench_code_tables_read.py > read.csv
# Make the plots
cat read.csv | python3 ./python/plot_code_tables_read.py
# Run the write benchmarks
python3 ./python/bench_code_tables_write.py > write.csv
# Make the plots
cat write.csv | python3 ./python/plot_code_tables_write.py
```

## Read

#### Fixed Length
TODO!:

#### Truncated
TODO!:

#### Unary
![](/codes/unary_read_tables.svg)

#### Gamma
![](/codes/gamma_read_tables.svg)

#### Delta
Since Delta has a gamma code inside, we can choose if the gamma code is read using a table or not.

Without gamma table:
![](/codes/delta_read_tables.svg)
With gamma table (8 bits):
![](/codes/delta_gamma_read_tables.svg)

#### Zeta 3
![](/codes/zeta3_read_tables.svg)

## Write

#### Fixed Length
TODO!:
#### Truncated
TODO!:

#### Unary
![](/codes/unary_write_tables.svg)

#### Gamma
![](/codes/gamma_write_tables.svg)

#### Delta
Since Delta has a gamma code inside, we can choose if the gamma code is write using a table or not.

Without gamma table:
![](/codes/delta_write_tables.svg)
With gamma table (TODO bits):
![](/codes/delta_gamma_write_tables.svg)

#### Zeta 3
![](/codes/zeta3_write_tables.svg)

# Summary

<table>
<thead>
  <tr><th>Code</th><th>Distribution</th><th>Avg. Read (ns)</th><th>Avg. Write (ns)</th></tr>
</thead>
<tbody>
  <tr><td>Fixed Length</td><td>\(\frac{1}{|S|}\), Uniform were \(|S|\) is a power of two</td><td>TODO</td><td>TODO</td></tr>
  <tr><td>Truncated</td><td>\(\frac{1}{|S|}\), Uniform</td><td>TODO</td><td>TODO</td></tr>
  <tr><td>Unary</td><td>\((1 / 2)^s\), Geometric with p = \(\frac{1}{2}\)</td><td>1.58</td><td>1.49</td></tr>
  <tr><td>Gamma</td><td>Universal, \(\approx \frac{1}{2s^2}\), zipfian with \(\alpha = 2\)</td><td>1.90</td><td>1.59</td></tr>
  <tr><td>Delta</td><td>Universal, \(\approx \frac{1}{2s(\log (s + 1))^2}\), zipfian with \(\alpha \approx 1\)</td><td>3.11</td><td>2.77</td></tr>
  <tr><td>Zeta (k = 3)</td><td>\(\approx \frac{1}{x^{1 + 1 / k}}\), close to an arbitrary zipfian</td><td>5.20</td><td>3.76</td></tr>
</tbody>
</table>

From all the benchmarks we learn that for **reads**:
- **Unary**: no table is needed.
- **Gamma**: with or without table is pretty close, so we can not use a table to leave
more cache space for the other codes.
- **Delta**: It's fastest with no table, but with gamma table enabled.
- **Zeta**: It's fastest with a 12-bit table.

**Gamma** is fast and universal, so it's a good
default choice.

**Writes** are generally faster than **Reads**, and all codes gets fasters with 
bigger tables.

Specifically, for **Writes** is optimal to use:
- **Unary**: 3 bits table.
- **Gamma**: 6 bits table.
- **Delta**: 13 bits table, with gamma tables.
- **Zeta3**: 15 bits table.

**LittleEndian** is slightly faster than **BigEndian** when **reading** with no tables, but with tables they are close due to the better cache locality of Big Endian tables.
**Writes** otherwise, are faster with **BigEndian**.

Generally, **BigEndian** seems the better choiche, as they `LZCNT` is supported
on more CPUs than `TZCNT`.

**Separated** tables are faster than **Combined** tables when the table is bigger than the L1 cache (this cpu has 32KiB \\(\approx 2^{15}\\) L1D per core), and basically the same otherwise.

*Benchmarks are useful :)*


# References
- <https://github.com/vigna/dsi-bitstream-rs>
- <https://vigna.di.unimi.it/ftp/papers/Codes.pdf>
- <https://boldi.di.unimi.it/Corsi/AlgorithmsForLargeGraphs/lesson1.pdf>