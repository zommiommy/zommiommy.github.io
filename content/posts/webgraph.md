+++
title = "Webgraph from scratch (Draft)"
description = "Long post on Webgraph, these are my notes about all topics related to WebGraph"
date = 2023-04-19
+++
# Introduction

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

This code has length \\(s + 1\\) therefore, its intended distribution is:
\\[p(x) = \left(\frac{1}{2}\right)^{x + 1}\\]  

Thus, it's optimal for geometric distributions with \\(p = \frac{1}{2}\\).

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

Most modern CPUs architectures, like x86_64 and Aarch64 (Arm), have instructions
to compute the leading or traling zeros in a registers. Using these we can decode
unary codes in 1 ns on average. 

### [Elias-Gamma](https://en.wikipedia.org/wiki/Elias_gamma_coding#Further_reading)

Unary + Fixed Length

<table>
<thead>
  <tr><th>Number</th><th>Code</th></tr>
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

### [Elias-Delta](https://en.wikipedia.org/wiki/Elias_delta_coding)

What happens if we write the fixed length part of gamma, using another gamma?

MEME: Oh boy here I go recursing again
MEME: I heard you like gamma, so I put another gamma inside your gamma

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

### Golomb

for a given b, write \\(\lfloor \sfrac{s}{b} \rfloor \\) in unary and 
\\(s mod b\\) using fixed lenght (or truncated encoding)

The length is \\(|c(s)| = \left \lfloor \frac{s}{b} \right \rfloor + \lceil \log_2 b \rceil\\) thus, its 

For a Geometric distribution of ratio \\(p\\) we can compute the \\(b\\) so that
the Golomb code is optimal:
\\[b = \left \lceil \frac{\log_2 (2 - p)}{-\log(1 - p)} \right \rceil\\]

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

## Encoding of negative numbers
All the discussed codes are defined over some subset of \\(\mathbb{N}\\), to 
encode over \\(\mathbb{Z}\\), we have to define a bijection \\(\phi: \mathbb{Z} \to \mathbb{N}\\).

If we can assume that the values form some kind of bell shape centered around zero,
as we often do, it's best to map values with small absolute value to small integers, so it's natural to use the following bijection:

\\[\phi(x) =  \left\\{\begin{matrix} -2 x & \text{if } x \le 0\\\\2 x - 1 & \text{otherwise}\end{matrix}\right.\\]
which has inverse:
\\[\phi^{-1}(x) =  \left\\{\begin{matrix} -x / 2 & \text{if } x = 0 \mod 2\\\\ (x + 1) / 2 & \text{otherwise}\end{matrix}\right.\\]

this can be efficiently implemented as:
```python
def nat2int(x):
    return (x >> 1) ^ ~((x & 1) - 1)

def int2nat(x):
    if x > 0:
        return x << 1
    else:
        return (x << 1) | 1
```

which in x86_64 assembly should look like:

```asm
; int2nat
; rax = x
shl rbx, rax, 1 ; x << 1 = 2 * x
cmp rax
adc rax

; nat2int
```

## Codes Summary

<table>
<thead>
  <tr><th>Code</th><th>Distribution</th></tr>
</thead>
<tbody>
  <tr><td>Fixed Length</td><td>Uniform were N is a power of two</td></tr>
  <tr><td>Truncated</td><td>Uniform</td></tr>
  <tr><td>Unary</td><td>Geometric with p = \\(\frac{1}{2}\\)</td></tr>
  <tr><td>Gamma</td><td></td></tr>
  <tr><td>Delta</td><td></td></tr>
  <tr><td>Zeta k = 3</td><td></td></tr>
</tbody>
</table>

# BVGraph
BVGraph is a graph compression data-structure which exploits some common 
properties of real-world graphs. The basic idea is to use instantaneous codes 
to represent an adjacency list (a.k.a. CSR).

The BVGraph format has three core compression methods:
- A node can copy successors from a previous node
- Intervals of nodes with consecutive ids are written as the tuple (start, len)
- The remaning nodes are sorted and encoded as a sequence of gaps (differences between successive nodes).

We can already notice that this compression depends from the node ids. 
Finding the optimal order is most likely NP Complete, so later in this post we 
will discuss some heuristic methods to find good orders.  

## Example
![](/webgraph_graph.svg)
Let's start by computing the neighbours of each node:
```
0: 1,2
1: 3
2: 3
3: 4,5,6
4: 5,6,8
5: 7
6: 7
7:
8:
```
The bits properly identified 
<div>
<span class="tag tag_degree">011</span><span class="tag tag_reference_offset">1</span><span class="tag tag_nintervals">1</span><span class="tag tag_first_residual">1010</span><span class="tag tag_residual_gap">1010</span>
</div>
Reading the codes we obtain:
<div>
<span class="tag tag_degree">3</span><span class="tag tag_reference_offset">0</span><span class="tag tag_nintervals">0</span><span class="tag tag_first_residual">2</span><span class="tag tag_residual_gap">2</span>
</div>
And after applying all the needed +1, -1 and nat -> signed converstions, we have:
<div>
<span class="tag tag_degree">2</span><span class="tag tag_reference_offset">0</span><span class="tag tag_nintervals">0</span><span class="tag tag_first_residual">1</span><span class="tag tag_residual_gap">2</span>
</div>
</div>


The default codes for BVGraph are:
<table>
<thead>
  <tr><th>Value</th><th>Code</th></tr>
</thead>
<tbody>
  <tr><td class="tag tag_degree">Outdegree</td><td>Gamma</td></tr>
  <tr><td class="tag tag_reference_offset">ReferenceOffset</td><td>Unary</td></tr>
  <tr><td class="tag tag_blocks_count">BlockCount</td><td>Gamma</td></tr>
  <tr><td class="tag tag_blocks">Blocks</td><td>Gamma</td></tr>
  <tr><td class="tag tag_nintervals">IntervalCount</td><td>Gamma</td></tr>
  <tr><td class="tag tag_interval_start">IntervalStart</td><td>Gamma</td></tr>
  <tr><td class="tag tag_interval_len">IntervalLen</td><td>Gamma</td></tr>
  <tr><td class="tag tag_first_residual">FirstResidual</td><td>Zeta(3)</td></tr>
  <tr><td class="tag tag_residual_gap">ResidualGap</td><td>Zeta(3)</td></tr>
</tbody>
</table>

## Compression tradeoffs
Compression windows, Reference chain length



## Offsets list
To have have random-access on a BVGraph bitstream we need to store the bitoffsets
at which each node codes start, analogously to the offsets list of a CSR matrix.

While this could just be a vector of integers, we can notice that the offsets are
a monotone non decreasing sequence of positive integers, thus they can be 
represented using Elias-Fano!

For a BVGraph bitstream of length \(l\) of a graph with \(|V|\) nodes, Elias-Fano stores the offsets using at most \\(|V| (2 + \lfloor \log2 \frac{l}{|V|} \rfloor)\\) bits while having \\(O(1)\\) access (select). 

[*See my other post for more info on Elias-Fano*](https://zom.wtf/posts/elias_fano_pt1)

## Benchmarks

## How to choose the best codes?
Use universal codes, gamma is really fast so it's a good default choice.
Otherwise we can "simulate" writing every piece with all the codes, this requires
just a complete iteration over the nodes so it's feasible also on large graphs.

Webgraph-rs allows has an optimized binary utility for this.

In my practical application we don't save much memory, about 3 GB over a 126GB graph,
but we found out that zeta3 and delta have similar compressions, so we use delta which is 4 times faster than zeta3.

# Computing an order

## BFS

## Layered Labels Propagation

## Benchmarks

# References
- <https://vigna.di.unimi.it/algoweb/WebGraph.pdf>
- <https://webgraph.di.unimi.it/>
- <https://vigna.di.unimi.it/ftp/papers/Codes.pdf>
- <https://www.ics.uci.edu/~djp3/classes/2008_01_01_INF141/Materials/p595-boldi.pdf>
- <https://github.com/vigna/webgraph>
- <https://github.com/vigna/webgraph-rs>
- <https://github.com/vigna/sux-rs>
- <https://boldi.di.unimi.it/Corsi/AlgorithmsForLargeGraphs/lesson1.pdf>
- <https://boldi.di.unimi.it/Corsi/AlgorithmsForLargeGraphs/lesson2.pdf>
- <https://boldi.di.unimi.it/Corsi/AlgorithmsForLargeGraphs/lesson3-1.pdf>
- <https://boldi.di.unimi.it/Corsi/AlgorithmsForLargeGraphs/lesson3-2.pdf>
- <https://boldi.di.unimi.it/Corsi/AlgorithmsForLargeGraphs/lesson4-1.pdf>
- <https://boldi.di.unimi.it/Corsi/AlgorithmsForLargeGraphs/lesson4-2.pdf>