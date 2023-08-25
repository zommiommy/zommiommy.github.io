+++
title = "Webgraph from scratch: BVGraph (Draft)"
description = "My notes on implementing and optimizing webgraph"
date = 2023-09-25
+++
*This post assumes familiarity with the instantaneous codes we discussed in [the previous post](/posts/webgraph-codes)*
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
Let's start with a simple graph:
![](/bvgraph_graph.svg)

I've compressed it with the default codes, which are:
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

This graph has 9 nodes and 12 arcs, once compressed with BVGraph we obtain two
files, the `test.properties`:
```ini
#BVGraph properties
version=0
graphclass=it.unimi.dsi.webgraph.BVGraph
nodes=9
arcs=12
minintervallength=3
maxrefcount=3
windowsize=7
zetak=3
compressionflags=
```
and the `test.graph` file which contains the compressed graph as the following bytes:
```
7d c5 ea 64 a7 27 72 97 a9 e0
```
which is equivalent to the bits:
```
01111101 11000101 11101010 01100100 10100111 00100111 01110010 10010111 10101001 11100000
```
and is equivalent to the bitstream:
```
011111011100010111101010011001001010011100100111011100101001011110101001111
```
With syntax highlighting:
<div>
<span class="tag tag_degree">011</span>
<span class="tag tag_reference_offset">1</span>
<span class="tag tag_nintervals">1</span>
<span class="tag tag_first_residual">1011</span>
<span class="tag tag_residual_gap">100</span>
<span>|</span>
<span class="tag tag_degree">010</span>
<span class="tag tag_reference_offset">1</span>
<span class="tag tag_nintervals">1</span>
<span class="tag tag_first_residual">1101</span>
<span>|</span>
<span class="tag tag_degree">010</span>
<span class="tag tag_reference_offset">01</span>
<span class="tag tag_blocks_count">1</span>
<span>|</span>
<span class="tag tag_degree">00100</span>
<span class="tag tag_reference_offset">1</span>
<span class="tag tag_nintervals">010</span>
<span class="tag tag_interval_start">011</span>
<span class="tag tag_interval_len">1</span>
<span>|</span>
<span class="tag tag_degree">00100</span>
<span class="tag tag_reference_offset">1</span>
<span class="tag tag_nintervals">1</span>
<span class="tag tag_first_residual">1011</span>
<span class="tag tag_residual_gap">100</span>
<span class="tag tag_residual_gap">1010</span>
<span>|</span>
<span class="tag tag_degree">010</span>
<span class="tag tag_reference_offset">1</span>
<span class="tag tag_nintervals">1</span>
<span class="tag tag_first_residual">1101</span>
<span>|</span>
<span class="tag tag_degree">010</span>
<span class="tag tag_reference_offset">01</span>
<span class="tag tag_blocks_count">1</span>
<span>|</span>
<span class="tag tag_degree">1</span>
<span>|</span>
<span class="tag tag_degree">1</span>
</div>
Reading the codes we obtain:
<div>
<span class="tag tag_degree">2</span>
<span class="tag tag_reference_offset">0</span>
<span class="tag tag_nintervals">0</span>
<span class="tag tag_first_residual">2</span>
<span class="tag tag_residual_gap">0</span>
<span>|</span>
<span class="tag tag_degree">1</span>
<span class="tag tag_reference_offset">0</span>
<span class="tag tag_nintervals">0</span>
<span class="tag tag_first_residual">4</span>
<span>|</span>
<span class="tag tag_degree">1</span>
<span class="tag tag_reference_offset">1</span>
<span class="tag tag_blocks_count">0</span>
<span>|</span>
<span class="tag tag_degree">3</span>
<span class="tag tag_reference_offset">0</span>
<span class="tag tag_nintervals">1</span>
<span class="tag tag_interval_start">2</span>
<span class="tag tag_interval_len">0</span>
<span>|</span>
<span class="tag tag_degree">3</span>
<span class="tag tag_reference_offset">0</span>
<span class="tag tag_nintervals">0</span>
<span class="tag tag_first_residual">2</span>
<span class="tag tag_residual_gap">0</span>
<span class="tag tag_residual_gap">1</span>
<span>|</span>
<span class="tag tag_degree">1</span>
<span class="tag tag_reference_offset">0</span>
<span class="tag tag_nintervals">0</span>
<span class="tag tag_first_residual">4</span>
<span>|</span>
<span class="tag tag_degree">1</span>
<span class="tag tag_reference_offset">1</span>
<span class="tag tag_blocks_count">0</span>
<span>|</span>
<span class="tag tag_degree">0</span>
<span>|</span>
<span class="tag tag_degree">0</span>
</div>

Finally, we can start iterating on the graph:

**Node 0**
<div>
Bits: <span class="tag tag_degree">011</span>
<span class="tag tag_reference_offset">1</span>
<span class="tag tag_nintervals">1</span>
<span class="tag tag_first_residual">1011</span>
<span class="tag tag_residual_gap">100</span><br>
Outdegree: <span class="tag tag_degree">2</span><br>
Reference Offset: <span class="tag tag_reference_offset">0</span><br>
Number of Intervals: <span class="tag tag_nintervals">0</span><br>
First Residual: <span class="tag tag_first_residual">2</span><br>
Residual Gap: <span class="tag tag_residual_gap">0</span><br>
</div>

This node has outdegree 2, doesn't copy any node and has no intervals, so it has
only residuals. 

The first residual is a signed offset from the current node,
so the first successor is \\(0 + \phi^{-1}(2) = 0 + 1 = 1\\).

The second residual is the gap from the previous residual, minus one, so:
\\(1 + (0 + 1) = 2\\).

So the successors of node 0 are \\(\\{1, 2\\}\\).

**Node 1**
<div>
Bits: 
<span class="tag tag_degree">010</span>
<span class="tag tag_reference_offset">1</span>
<span class="tag tag_nintervals">1</span>
<span class="tag tag_first_residual">1101</span><br>
Outdegree: <span class="tag tag_degree">1</span><br>
Reference Offset: <span class="tag tag_reference_offset">0</span><br>
Number of Intervals: <span class="tag tag_nintervals">0</span><br>
First Residual: <span class="tag tag_first_residual">4</span><br>
</div>

This node has outdegree 1, doesn't copy any node and has no intervals, so it has
only residuals. 

The first residual is a signed offset from the current node,
so the first successor is \\(0 + \phi^{-1}(4) = 1 + 2 = 3\\).

So the successors of node 1 are \\(\\{3\\}\\).


## Compression tradeoffs
Compression windows, Reference chain length

## Offsets list
To have have random-access on a BVGraph bitstream we need to store the bitoffsets
at which each node codes start, analogously to the offsets list of a CSR matrix.

While this could just be a vector of integers, we can notice that the offsets are
a monotone non decreasing sequence of positive integers, thus they can be 
represented using Elias-Fano!

For a BVGraph bitstream of length \(l\) of a graph with \(|V|\) nodes, Elias-Fano stores the offsets using at most \\(|V| (2 + \lfloor \log2 \frac{l}{|V|} \rfloor)\\) bits while having \\(O(1)\\) access (select) if paired with an appropriate index.

[*See my other post for more info on Elias-Fano*](https://zom.wtf/posts/elias_fano_pt1)

## Benchmarks

## How to choose the best codes?
Use universal codes, gamma is really fast so it's a good default choice.
Otherwise we can "simulate" writing every piece with all the codes, this requires
just a complete iteration over the nodes so it's feasible also on large graphs.

Webgraph-rs has an optimized binary utility for this.

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