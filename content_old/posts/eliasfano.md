---
title: "On storing a set of integers and Elias-Fano"
date: 2021-08-31T20:44:23+02:00
draft: true
---

In this post we will tackle the problem of indexing a set \\(X\\) of positive integers
that takes values in the range \\(0 \le x \le u \quad \forall x \in X\\) for a given upper-bound \\(u\\). 

Our goal is to store this set in as little memory as possible and to have the following two operations in \\(\mathcal{O}(1)\\) (constant time):
- \\(\text{rank}(X, v)\\) returns the number of values in \\(X\\) that are strictly smaller than \\(v\\)
- \\(\text{select}(X, i)\\) returns the \\(i\\)-th smallest value in \\(X\\)

These two operations were introduced by Jacobson in its influential paper and were
originally meant to simulate a tree, but they are generally usefull and fundamental
operations for all succint data-structures.

To be more precise, we are searching for a **succint data-structure**, which is a representation that allows for fast **rank** and **select** and uses \\(Z + o(Z)\\) bits, where
\\(Z\\) is the information-theoretical minimum space required.

------
## Theoretical Lowerbound
Before thinking about the data-structure we should know what's the theoretical minimum 
number of bits \\(Z\\) required for our task. To do so we can borrow Shannon's entropy.
Shannon's tells us that the entropy \\(\mathbb{H}(X_S)\\) of a random variable \\(X_S\\) that has support over a set \\(S\\) of element \\(s \in S\\) with probability \\(P(s)\\) is:
\\[\mathbb{H}(X_S) = \left \lceil -\sum_{s \in S} P(s) \log_2 P(s) \right \rceil \quad \text{bits}\\]

Since we cannot make any assumption over the probability of each possible set we want
to store and index, we have to consider an uniform distribution \\(P(s) = \frac{1}{|S|} \quad \forall s \in S\\). In this particular condition, the entropy of our random variable greatly simplify:

\\[\mathbb{H}(X_S) = \left \lceil -\sum_{s \in S} P(s) \log_2 P(s) \right \rceil ~ \text{bits} = \left \lceil -\sum_{s \in S} \frac{1}{|S|} \log_2 \frac{1}{|S|} \right \rceil =\left \lceil  - \log_2 \frac{1}{|S|} \right \rceil  = \left \lceil \log_2 |S| \right \rceil ~\text{bits}  \quad \Rightarrow\quad  \mathbb{H}(S) = \left \lceil \log_2 |S| \right \rceil ~\text{bits}\\]  

So the next step is to figure out what's the cardinality of our set \\(|S|\\), which
is the set of all the possible sets of integers with \\(|X|\\) values between \\(0\\) and \\(u\\).
**If we don't allows for duplicates**, this can be thought as how many ways we can extract \\(n = |X|\\) values for a set of \\(u\\) elements. This is a classical combinatorial problem so we know that \\(|S| = \binom{u}{n}\\). So finally we obtain that:

\\[Z = \mathbb{H}(X_S) = \left \lceil \log_2 \binom{u}{n} \right \rceil \quad \text{bits}\\]

To be able to better understand how many bits these are, we can asymptotically simplify it using
the binomal coefficient definition and Stirling's approximation to all the factorials:

\\[\log_2 \binom{u}{n} = \log_2 \frac{u!}{n!(u-n)!}\qquad\qquad n! \sim_{n \to \infty} \sqrt{2\pi n } \left (\frac{n}{e} \right)^n \qquad\Rightarrow\qquad \log_2 \binom{u}{n} \sim_{u, n \to \infty} \log_2 \frac{\sqrt{2 \pi u} \left ( \frac{u}{e} \right)^u}{\sqrt{2 \pi n} \left ( \frac{n}{e} \right)^n \sqrt{2 \pi (u - n)} \left ( \frac{(u - n)}{e} \right)^{(u - n)}}\\]

Now hold your breath and let's start doing some arithmetic simplifications:

\\[
\begin{aligned}
\log_2 \binom{u}{n} &\sim_{u, n \to \infty}  \log_2 \frac{\sqrt{u} u^u e^{-u}}{\sqrt{n} n^n e^{-n} \sqrt{2 \pi} \sqrt{u - n} (u - n)^{(u - n)} e^{n - u}} \\\\
&\sim_{u, n \to \infty}  \log_2 \frac{\sqrt{u} u^u}{\sqrt{2 \pi} \sqrt{n} n^n  \sqrt{u - n} (u - n)^{(u - n)}} \\\\
&\sim_{u, n \to \infty}  \log_2 \frac{1}{\sqrt{2 \pi}} \sqrt{\frac{u}{n (u - n)}} \frac{u^u}{n^n (u - n)^{(u - n)}} \\\\
&\sim_{u, n \to \infty}  \log_2 \frac{1}{\sqrt{2 \pi}} + \log_2 \sqrt{\frac{u}{n (u - n)}} + \log_2 \frac{u^u}{n^n (u - n)^{(u - n)}} \\\\
&\sim_{u, n \to \infty}  -\frac{1}{2} \log_2 2 \pi + \frac{1}{2}\log_2 \frac{u}{n (u - n)} + \log_2 \frac{u^u}{n^n (u - n)^{(u - n)}} \\\\
&\sim_{u, n \to \infty}  -\frac{1}{2} \log_2 2 \pi + \frac{1}{2}\log_2 u - \frac{1}{2}\log_2 (u - n) + u \log_2 u - n \log_2 n - (u - n) \log_2 (u - n) \\\\
&\sim_{u, n \to \infty}  u \log_2 u - n \log_2 n - (u - n) \log_2 (u - n) -\frac{1}{2} \log_2 2 \pi + \frac{1}{2}\log_2 u - \frac{1}{2}\log_2 (u - n)  \\\\
&\sim_{u, n \to \infty}  u \log_2 u - n \log_2 n - (u - n) \log_2 (u - n) \underbrace{+ (u -n) \log_2 u - (u - n) \log_2 u}_{0} -\frac{1}{2} \log_2 2 \pi + \frac{1}{2}\log_2 u - \frac{1}{2}\log_2 (u - n)  \\\\
&\sim_{u, n \to \infty}  n \log_2 \frac{u}{n} + (u - n) \log_2 \frac{u}{u - n} + \frac{1}{2} \log_2 \frac{u}{(u - n)} - \log_2 \sqrt{2 \pi} \\\\
&\sim_{u, n \to \infty}  n \log_2 \frac{u}{n} + (u - n) \log_2 \frac{u}{u - n} + \mathcal{O}(\log u) \\\\
&\sim_{u, n \to \infty}  n \log_2 \frac{u}{n} + \mathcal{O}(n) \\\\
\end{aligned}
\\]

Therefore, we finally obtain that:

\\[Z \sim_{u, n \to \infty} n \log_2 \frac{u}{n} + \mathcal{O}(n)\\\]
(It can be trivially proved that in this context the ceil does not change the complexity)

We can try to interpret this result, \\(u / n\\) is the average gap between two
successive values in the set. This means that the minimum we can do is to encode the gaps
between the values in the set.

-------
## Elias-Fano
There's a data-strcutrue introduced by Sebastiano Vigna which, based on the work from Perer Elias and Rober Fano encodings, can solve our task using memory which is **close** to the lower-bound we found and it's called Elias-Fano.

This encoding sort the values and split them into high and low bits. The low bits are stored
contiguosly, while the gap between the high-bits are stored using the inverted unary encoding.

{{< figure src="/elias_fano.png" width="100%">}}

To be precise, each integer can be represented as a binary sequence using \\(\lceil \log_2 u \rceil\\) bits.
This representations split this binary sequence into the first \\(\lceil \log_2 n \rceil\\) bits,called the the higher bits, and the lower \\(l = \lceil \log_2 u \rceil - \lceil \log_2 n \rceil \le \left \lceil \log_2 \frac{u}{n} \right \rceil\\) bits.
The lower bits takes \\(L = n l \le n \left \lceil \log_2 \frac{u}{n} \right \rceil\\) bits, which we can recognize as the lower bound previously proved. While the gaps between the higher-bits are encoded using the inverted unary encoding \\(\mathcal{U}: \mathbb{N} \to 0^*1\\) which for a given integer \\(x\\) it encodes it as a sequence of \\(x\\) **0s** and then a delimiter **1**:
\\[ \mathcal{U}(0) \to 1 \quad \mathcal{U}(1) \to 01 \quad \mathcal{U}(2) \to 001 \quad \mathcal{U}(x) \to 0^x1 \qquad\Rightarrow\qquad |\mathcal{U}(x)| = x + 1\\]

If we define the higher-bits as \\(h_i \quad \forall 0 \le i \le n\\), where \\(h_0 = 0\\), we can compute the space used by the high-bits \\(H\\) as:
\\[H = \sum_{0 \le i < n} |\mathcal{U}(h_{i + 1} - h_{i})| = \sum_{0 \le i < n} h_{i + 1} - h_{i} + 1 = \sum_{0 \le i < n} h_{i + 1} - h_{i} + \sum_{0 \le i < n } 1 = n + \sum_{0 \le i < n} h_{i + 1} - h_{i}\\]

We can recognize that this is a telescopic series, so we can simplify it by using the definition of high-bits:
\\[H = n + \sum_{0 \le i \le n} h_{i + 1} - h_{i} = n + h_n = n + \left\lfloor \frac{u}{2^l} \right \rfloor \le n +  \left\lfloor \frac{u}{2^{\left \lceil \log_2 \frac{u}{n} \right \rceil}}\right \rfloor  \le  n + \frac{u}{2^{\log_2 \frac{u}{n}}} = n + \frac{u}{\frac{u}{n}} = 2n\\]

So the space used by Elias-Fano \\(\mathcal{EF}(u, n)\\) is composed by its higher \\(H\\) and lower \\(L\\) bits so:
\\[\mathcal{EF}(u, n) = H + L \le 2n + n \left \lceil \log_2 \frac{u}{n} \right \rceil\\]

This means that Elias-Fano is asymptotically optimal as:
\\[\mathcal{EF}(u, n) \le 2n + n \left \lceil \log_2 \frac{u}{n} \right \rceil = n \log_2 \frac{u}{n} + \mathcal{O}(n) \sim_{u, n \to \infty} Z\\]
Therefore, we can prove that this structure uses at most 2-extra bits for element than the theoretical minimum:
\\[\mathcal{EF}(u, n) - Z \sim_{u, n \to \infty} 2n - \mathcal{O}(n) \le 2n\\]
Fano proved a better bound, this encoding uses at most half-bit for element more than the theoretical minimum, so this structure it's not a succint data-structure, but it's really close thus Vigna's name of quasi-succint data-structure.

------

## Building Elias-Fano

There's an useful property about the high-bits that allows us to speed-up the construction of elias-fano, that can be proved in an analogous way to the space used by the high-bits. The position \\(p_j\\) of the \\(j\\)-th **1** in the high-bits is:

\\[p_j = \sum_{0 \le i < j} |\mathcal{U}(h_{i + 1} - h_{i})| = h_j + j\\]
This little fact remove the dependancy between different values, which allows,
given a sorted vector of values, to build Elias-Fano completely in parallel.