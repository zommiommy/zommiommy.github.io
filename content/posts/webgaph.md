+++
title = "Webgraph from scratch (Draft)"
description = "First post on Webgraph, here we discuss about the instantaneous codes needed to approach the discussion on webgraph"
date = 2023-04-19
+++
# Introduction

# Theoretical preliminaries
All bit sequences will be read from left to right, from top to bottom.
For ease of reading I will be using the convention that **all codes start from 0** 
by adding a +1 where needed. This differs from the **book definitions** of the codes,
but it's how you practically implement them.
## Entropy

# Instantaneous Codes

### Intended Distributions

### Fixed length
Optimal for uniform distributions of values in \\(\left[0, 2^\text{n_bits}\right)\\): 
```python
def write_fixed_length(value, n_bits):
    return ("{:0%sb}"%n_bits).format(value)

def read_fixed_length(data, n_bits):
    return int(data[:n_bits], 2), data[n_bits:]
```

### Unary
Optimal for geometric distributions with \\(p = \frac{1}{2}\\), i.e. \\(P(x) = \left(\frac{1}{2}\right)^{x + 1}\\):
```python
def write_unary(value):
    return "0" * value + "1"

def len_unary(value):
    return value + 1

def read_unary(data):
    res = len(data.ltim("0")) - len(data)
    return res, data[res + 1:]
```
### Elias-Gamma
```python
from math import floor, ceil, log2

def write_gamma(value):
    value += 1
    l = floor(log2(value))
    s = value - (1 << l)
    data = write_unary(l, bitstream, m2l)
    if l != 0:
        data += write_fixed_length(s, n_bits=l)
    return v

def len_gamma(value):
    return 2*floor(log2(value + 1)) + 1

def read_gamma(data):
    l, data = read_unary(data)
    if l == 0:
        return 0, data
    f, data = read_fixed_length(data, n_bits=l)
    v = f + (1 << l) - 1
    return v, data
```
### Elias-Delta
```python
from math import floor, ceil, log2

def write_delta(value):
    value += 1
    l = floor(log2(value))
    s = value - (1 << l)
    data = write_gamma(l, bitstream, m2l)
    if l != 0:
        data += write_fixed_length(s, n_bits=l)
    return data

def len_delta(value):
    l = floor(log2(value + 1))
    return l + 2*floor(log2(l + 1)) + 1

def read_delta(data):
    l, data = read_gamma(data)
    if l == 0:
        return 0, data
    f, data = read_fixed_length(data, n_bits=l)
    v = f + (1 << l) - 1
    return v, data
```

### Truncated Binary Encoding
```python
from math import floor, ceil, log2

def write_minimal_binary(value, max):
    l = int(floor(log2(max))) 
    limit = (1 << (l + 1)) - max

    if value < limit:
        data = write_fixed_length(value, n_bits=l)
    else:
        to_write = value + limit
        data  = write_fixed_length(to_write >> 1, n_bits=l)
        data += write_fixed_length(to_write & 1, n_bits=1)
    return data

def len_minimal_binary(value, max):
    l = int(floor(log2(max)))
    limit = (1 << (l + 1)) - max
    if value >= limit:
        return l + 1
    else:
        return l

def read_minimal_binary(data, max):
    l = int(floor(log2(max)))
    v, data = read_fixed_length(data, n_bits=l)
    limit = (1 << (l + 1)) - max

    if v < limit:
        return v, data
    else:
        b, data = read_fixed_length(data, n_bits=1)
        v = (v << 1) | b
        return v - limit, data
```

### Zeta
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

# Encoding of negative numbers
```python
def nat2int(x):
    return (x >> 1) ^ ~((x & 1) - 1)

def int2nat(x):
    if x > 0:
        return x << 1
    else:
        return (x << 1) | 1
```

# Webgraph


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

# Benchmarks