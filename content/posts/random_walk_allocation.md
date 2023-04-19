+++
title = "On the optimal memory allocation for Random Walks on a Directed Graph (Draft)"
date = 2021-08-31
+++

In this post I will share some simple results around the optimal memory 
allocation for a random walk on a directed graph, and its time-memory tradeoffs.
I studied this to speed-up the computation of random-walks by optimally guessing 
how much memory to allocate for each walk.

To precisely define what optimality means in this context, we have to setup our 
cost framework. 
We are going to study a simplified allocator which by default allocates arrays 
of length \\(k\\) and if needed, it expand the array to the maximum length \\(L\\),
and possibly copy the old values to the new array (for a total cost of \\(k + L\\)).

Therefore we can define the cost function for a random walk of length \\(l\\) as:

\\[m(l) = \left\\{\begin{matrix}k & l \le k \\\\ k + L & else\end{matrix}\right.\\]

Therefore the expected cost is:

\\[\mathbb{E}\[m(k)\] = k P(X \le k) + (k + L) P(X > k) \\]

Where \\(X\\) is the aleatory variable associated to the length of a random walk
on a given graph.

These probabilities depends on several factors and are hard to analyze, so to be
able to gain some insight on this complex problem, we are going to follow 
[the first Engineering rule](https://i.redd.it/qrywf2y4e6u31.jpg) and 
approximate it.

The idea is that at each step of the random walk the random walk will either 
stop or continue so we can **approximate** it as a Bernulli trial.

Therefore, are going to use a simplified model where we have a fully connected graph except for traps nodes (in blue in the image), which by definition are nodes with only inbound edges.

![test](/model_graph_trap.svg)

They are called trap nodes because once the random walk it can no longer continue, because the node doesn't have any outbound edges.

Using this simplified model, each step is a bernulli trial, and we are intrested
in the probability of a sequence of \\(k\\) successes and a failure which by
definition is a Geometric distribution of parameter \\(p\\).

Which mean that we can approximate the terms of the previous formula as:
\\[P(X \le k) \approx (1 - p)^k p \qquad P(X > k) \\approx (1 - p)^k\\]


\\[\mathbb{E}\[m(k)\] \approx k (1 - p)^k p + (k + L) (1 - p)^k \\]

since we want to minimize the expected cost, we can compute it's partial derivate 
to find the critical points:

\\[\frac{\partial}{\partial k}\mathbb{E}\[m(k)\] \approx p(1-p)^k \big(k \log(1-p) + 1 \big) + (1-p)^k \big ((L + k) \log(1 - p) + 1 \big)\\]

## Extimate the parameter \\(p\\) 

\\(p\\), it's the probability that during a walk, the successor of a given node it's a trap.

This is expensive to compute **during the walk** so we might approximate it computing the **Average** probability over all nodes.


For a given node \\(n\\), with neighbours \\(N(n)\\), the campionary probability for the node to have traps as neighbours is:

\\[P(\text{Node \\(n\\) has a trap neigbour}) = \frac{1}{|N(n)|} \sum\_{d \in N(n)} \text{is\_trap}(d)\\]

To have the probability that on any given graph a node has a trap as a neighbouring node, we need to compute the mean:

\\[
\mathbb{E}[P(\text{Node $n$ has a trap neigbour})]= \frac{1}{|N|}\sum_{n\in N}\frac{1}{|N(n)|}\sum_{d\in N(n)} \text{is_trap}(d)
\\]

For node_to_vec we can use:

\\[
\mathbb{E}[P(\text{Node $n$ has a trap neigbour})]= \frac{1}{|N|}\sum_{n\in N}\sum_{n_i\in {Ne}_n} P_i \quad \text{is_trap}(d)
\\]

Fs is the Forward-Star aka the neighbours of the given node.

We can use this value to compute the expected length of a random walk.

## Average random walk length

We need to compute the average length until a trap, therefore we need a geometric distribution.

$$P(X = k) = p(1-p)^k$$

$$P(X \le k) = 1 - (1-p)^k$$

Therefore, we can choose a "safety level" \\(\beta\\) which it's an user chosen 
parameter.

So, given an estimate of \\(p\\), we can compute the random walk length \\(k\\) 
associated to the safety level \\(\beta\\):

$$k = \frac{\ln(1 - \beta)}{\ln\left(1-p\right)}$$

## Choosing the Optimal precision \\(\beta\\)

### Time and Memory Analysis
We want to model the memory and time cost of an allocation strategy for the walks of a given graph.

The simple strategy we will analyze is just to allocate \\(k\\) elements, and then if the walk excede this size, we reallocate the array with \\(L\\) elements which is the max length of walks.

Reallocating is an expensive operation since (on the current libc implemnetation of jmalloc) if the next heap chunck it's occupied, the allocator is forced to allocate a new array and copy there the content and free the old array.

We denote \\(s\_i\\) is the length of the i-th walk. Denoting with \\(m(s\_i)\\) the memory required by the i-th walk.

## Memory analysis

Since the allocation strategy allocate \\(k\\) elements and then, if the walk exceeds \\(k\\) steps, we expand it to \\(L\\), it's memory usage can be modeled as:

\\[ m(s\_i) = \left\\{\begin{matrix}k & s\_i \le k\\\\L & else\end{matrix}\right.\\]

$$M(k) = \sum_i m(s_i)$$

$$M(s) = \sum_i k + \sum_j L = w(1-\beta) k + w\beta L$$

$$m(s) = (1-\beta) \frac{\ln(1 - \beta)}{\ln\left(1-p\right)} + \beta L$$

$$m(s) = \frac{1}{\ln(1-p)} \left[(1-\beta)\ln(1 - \beta) + \beta \ln\left((1-p)^L\right)\right]$$

$$m(s) = \frac{1}{\ln(1-p)} \left[(1-\beta)\ln(1 - \beta) \underbrace{+\beta\ln(\beta) -\beta\ln(\beta)}_0 + \beta \ln\left((1-p)^L\right)\right]$$

$$m(s) = \frac{1}{\ln(1-p)} \left[- H(\beta) -\beta\ln(\beta) + \beta \ln\left((1-p)^L\right)\right]$$

$$m(s) = \frac{1}{-\ln(1-p)} \left[ H(\beta) + \beta\ln(\beta) - \beta \ln\left((1-p)^L\right)\right]$$

$$m(s) = \frac{1}{-\ln(1-p)} \left[H(\beta) + \beta \ln\left(\frac{\beta}{(1-p)^L}\right)\right]$$

Since it's the entropy of a binary event, \\(0 \le H(\beta) \le 1\\) so we can use 1 as upperbound.

$$m(s) \le \frac{1}{-\ln(1-p)} \left[1 + \beta \ln\left(\frac{\beta}{(1-p)^L}\right)\right]$$

Therefore the **final complexity** with regards to \\(\beta\\) is:

$$m(s) = \mathcal{O}\left( \beta \ln\left(\frac{\beta}{(1-p)^L}\right)\right) = \mathcal{O}\left( \beta \ln\left(\beta\right)\right)$$

### Finding the optimal \\(\beta\\) pt.1

It's to notice that since \\(0 \le \beta \le 1\\):

$$ -\frac{1}{e} \le \beta \ln(\beta) \le 0$$

and the value of \\(\beta\\) that minimize \\(\beta \ln(\beta)\\) is \\(\beta = \frac{1}{e}\\).

So we have the result:

$$m(s) \le \frac{1}{-\ln(1-p)} \left[1 + \beta \ln\left(\frac{\beta}{(1-p)^L}\right)\right]$$

$$m(s) \le \frac{1}{-\ln(1-p)} \left[1 + \beta \ln\left(\beta\right) - \beta \ln((1-p)^L)\right]$$

$$m(s) \le \frac{1}{-\ln(1-p)} \left[1 -\frac{1}{e} - \frac{1}{e} \ln((1-p)^L)\right]$$

$$m(s) \le \frac{(1 - \frac{1}{e}) - \frac{1}{e} L \ln(1-p)}{-\ln(1-p)}$$

$$m(s) \le \frac{1}{e} L + \frac{1 - \frac{1}{e}}{-\ln(1-p)}$$

### Finding the optimal \\(\beta\\) pt.2

Another assignement of \\(\beta\\) is \\(\beta = (1-p)^L\\) then we get

$$m(s) = \frac{1}{-\ln(1-p)} \left[H(\beta) + \beta \ln\left(\frac{\beta}{\beta}\right)\right]$$

$$m(s) = \frac{H(\beta)}{-\ln(1-p)} \le \frac{1}{-\ln(1-p)}$$

The unit of measure is number of integers. Therefore if we multiply the bound by 64 we get the size in bits for modern systems.

### What \\(\beta\\) to use and when

Let's find invert the disequation to know when to use which equation

$$ \frac{1}{e} L + \frac{1 - \frac{1}{e}}{-\ln(1-p)} \le \frac{1}{-\ln(1-p)}$$

$$ \frac{1}{e} L \le \frac{1}{-\ln(1-p)} - \frac{1 - \frac{1}{e}}{-\ln(1-p)}$$

$$ \frac{1}{e} L \le \frac{1 - (1 - \frac{1}{e})}{-\ln(1-p)} $$

$$ \frac{1}{e} L \le \frac{\frac{1}{e}}{-\ln(1-p)} $$

$$  L \le \frac{1}{-\ln(1-p)} $$

$$ -\ln(1-p)L \le 1 $$

$$ \ln\left(\frac{1}{(1-p)^L}\right) \le 1 $$

$$ \frac{1}{(1-p)^L} \le e $$

$$ (1-p)^L > \frac{1}{e} $$

Therefore if \\((1-p)^L > \frac{1}{e}\\) then it's best to use \\(\beta = \frac{1}{e}\\) otherwise \\(\beta = (1-p)^L\\).

Therefore:

$$\beta = \left\\{\begin{matrix}
\frac{1}{e} & (1-p)^L > \frac{1}{e}\\ 
(1-p)^L  & (1-p)^L \le \frac{1}{e}
\end{matrix}\right.$$

Which is equivalent to:

$$\beta = \min{\left\\{\frac{1}{e}, (1-p)^L\right\\}}$$

Therefore almost we will have \\(\beta = (1-p)^L\\) since \\(\frac{1}{e} \approx 0.36787944\\).

Moreover, it's easy to choose based on \\(p\\), since the walk length we expect is around 80:
$$p = 1 -\sqrt[L]{\frac{1}{e}}$$
and
$$p = 1 -\sqrt[80]{\frac{1}{e}} = 1 - 0.9875778 = 0.0124222$$

Finally, for the expected walk length, if the trap_rate is over 1.24% then it's better to use \\(\beta = (1-p)^L\\) .

The final result is that the optimal value is:

$$\beta = \min{\left\\{\frac{1}{e}, (1-p)^L\right\\}}$$

## Time analysis

Since the allocation strategy allocate \\(k\\) elements and then, if the walk exceeds \\(k\\) steps, we expand it to \\(L\\), it's memory usage can be modeled as:

$$ t(s\_i) = \left\\{\begin{matrix}
k & s\_i \le k\\\\
L + k & else
\end{matrix}\right.
$$

$$T(k) = \sum_i t(i)$$

We can greately simplify this formulations:

$$T(k) = \sum_i k + \sum_j (k + L) = w(1 - \beta) k + w\beta(k + L)$$

$$t(k) = (1 - \beta) \frac{\ln(1 - \beta)}{\ln\left(1-p\right)} + \beta\left(\frac{\ln(1 - \beta)}{\ln\left(1-p\right)} + L\right)$$

$$t(k) = \frac{1}{\ln\left(1-p\right)} \left[(1 - \beta) \ln(1 - \beta)+ \beta\ln(1 - \beta) + L\beta\ln\left(1-p\right)\right]$$

$$t(k) = \frac{1}{\ln\left(1-p\right)} \left[(1 - \beta) \ln(1 - \beta) + \underbrace{\beta \ln(\beta) - \beta \ln(\beta)}_0 + \beta\ln(1 - \beta)  + L\beta\ln\left(1-p\right)\right]$$

$$t(k) = \frac{1}{\ln\left(1-p\right)} \left[-H(\beta) - \beta \ln(\beta) + \beta\ln(1 - \beta)  + L\beta\ln\left(1-p\right)\right]$$

$$t(k) = \frac{1}{\ln\left(1-p\right)} \left[-H(\beta) + \beta\ln\left(\frac{1 - \beta}{\beta}\right)  + \beta\ln\left((1-p)^L\right)\right]$$

$$t(k) \le \frac{1}{\ln\left(1-p\right)} \left[1 + \beta\ln\left(\frac{1 - \beta}{\beta}\right)  + \beta\ln\left((1-p)^L\right)\right]$$


## Time with \\(\beta\\) optimized for memory.

if \\(\beta = \frac{1}{e}\\)

$$t(k) \le \frac{1}{e}L + \frac{1.19912}{\ln(1-p)}$$

if \\(\beta = (1-p)^L\\) 

$$t(k) \le \frac{1}{\ln(1-p)} + \frac{(1-p)^L \ln(1 - (1-p)^L)}{\ln(1-p)} \le \frac{1}{\ln(1-p)} + \frac{(1-p)^L}{\ln(1-p)}$$

$$t(k) \le \frac{1}{\ln(1-p)} + \frac{(1-p)^L}{\ln(1-p)} \le \frac{2}{\ln(1-p)}$$

