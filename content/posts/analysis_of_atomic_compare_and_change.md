+++
title = "Analysis of the complexity of compare exchange"
description = ""
date = 2021-09-05
draft = true
+++


\subsection{Constant amortized time push of a vector}

\begin{lemma}\label{lemma:vector_push_amoritzed}
Pushing an element \(\varepsilon\) onto a vector \(\bm{a}\) takes amortized constant time. 
\[\bm{a}.\text{push}(\varepsilon) \in \Theta(1)\]
\end{lemma}

\begin{proof}
If the vector \(\bm{a}\) is not full, appending a value takes constant time, otherwise, it has linear worst-case complexity \(\Theta(\abs{\bm{a}})\) w.r.t. the size of the vector \(\abs{\bm{a}}\) due to the need for reallocation, but this happens only once every \(\abs{\bm{a}}\) appends.
Therefore, the amortized time complexity is \(\sfrac{1}{\Theta(\abs{\bm{a}})} (\Theta(\abs{\bm{a}}) - 1) \Theta(1) + \Theta(\abs{\bm{a}}) = \Theta(1)\).
\end{proof}

\begin{lemma}\label{lemma:vector_push}
Pushing an element \(\varepsilon\) onto a vector \(\bm{a}\) takes approximately constant time on modern memory allocators. 
\[\bm{a}.\text{push}(\varepsilon) \in \Theta(1)\]
\end{lemma}

\begin{proof}
While the initial allocation of a vector \(\bm{a}\) takes \(\Theta(\abs{\bm{A}})\), the allocation for new vectors can and often do reuse the allocations of previously freed vectors making re-allocations events less frequent as the computation proceeds. Thus, in practice, we can approximate its worst-case complexity as constant time.
\end{proof}

\begin{lemma}\label{lemma:frontier_push}
Pushing an element \(\varepsilon\) onto a parallel frontier \(\Frontier\) takes approximately constant time on modern memory allocators. 
\[\Frontier.\text{push}(\varepsilon) \in \Theta(1)\]
\end{lemma}

\begin{proof}
The frontier is composed of \(\NumberOfThreads\) vectors, one for each thread, and as such, there is no synchronization overhead, and the lemma~\ref{lemma:vector_push} directly applies.  
\end{proof}

\input{scraps/binom_bounds_proof.tex}

\begin{lemma}\label{lemma:binom_bound}
For any pair of integer \(n, k \in \mathbb{N}\) such that \(n \ge k\) it holds that:
\[n^{-k} \frac{n!}{(n-k)!} =  \frac{n!}{(n-k)! n^k} \le ne^{-k}\]
\end{lemma}

\begin{proof}
we apply upper-bound from lemma~\ref{lemma:factorial_bounds} to the numerator factorial and the lower-bound from lemma~\ref{lemma:factorial_bounds} to the factorial in the denominator to obtain an upper-bound of \(\binom{n}{k}\):
\begin{align*}
\frac{n!}{(n-k)! n^k} &\le n^{-k} \frac{\frac{n^{n+1}}{e^{n-1}}}{\frac{(n-k)^{n-k}}{e^{n-k-1}}}\\
&=  n^{-k} \frac{n^{n+1}}{e^{n-1}}\frac{e^{n-k-1}}{(n-k)^{n-k}}\\
&=  n^{-k} \frac{e^{n-k-1}}{e^{n-1}}\frac{n^{n+1}}{(n-k)^{n-k}}\\
&=  n^{-k} \frac{1}{e^k} \frac{n^{n+1}}{(n-k)^{n-k}}\\
&\le n^{-k} \frac{1}{e^k} \frac{n^{n+1}}{n^{n-k}}\\
&=  n^{-k} \frac{n^{k+1}}{e^k}\\
&= n e^{-k}\\
\end{align*}

Therefore:
\[\frac{n!}{(n-k)! n^k} \le ne^{-k}\]
\end{proof}

\begin{lemma}\label{lemma:collisions_probability}
The probability \(p\) that \(\NumberOfThreads\) threads have \(c\) collision writing to uniformly sampled elements from a vector of size \(\NumberOfNodes\), where each values uses \(m_\Nodes\) bits is:
\[p =\frac{l!}{(l - t + c)! l^{t-c}} \le \Landmark e^{c-t}\]
where \(l = \sfrac{\NumberOfNodes}{\left\lceil\sfrac{64 \text{bytes}}{\NodeBits \text{bits}}\right\rceil}\) is the number of cache-lines needed to to store contiguosly the \(\NumberOfNodes\) values on x86\_64 CPUS. 
This theorem holds if, and only if, the number of cache-lines \(\Landmark\) is bigger than the number of threads, i.e. \(\left\lceil\sfrac{\NumberOfNodes}{k}\right\rceil \ge t - c\), otherwise \(P = 1\) due to the pigeon-hole principle.
\end{lemma}

\begin{proof}
An atomic operation might fail if, and only if, two or more threads write to the same cache-line, which on x86\_64 is 64 bytes.
Thus on the same cache-line we will have the bits of \(k = \left\lceil\sfrac{64 \text{bytes}}{\NodeBits \text{bits}}\right\rceil\) different features.
Therefore we can simplify the problem to the probability that each thread chooses a different cache-line, and the \(\NumberOfNodes\) features will be stored contiguously using \(\sfrac{\NumberOfNodes}{k}\) cache-lines.

\[p = \frac{l!}{(l - t + c)! l^{t-c}}\]

\end{proof}

\begin{lemma}\label{lemma:saturating_add}
The expected number of iterations \(k\) a thread will have to execute the CAS operation to win a tie is:
\[p(k, \NumberOfThreads) \ge \left(1 - \frac{1}{l}\right)^k\]
\end{lemma}

\begin{proof}
The CAS operation is the instruction LOCK CMPXCHG on x86\_64 CPUs.

The hardware usually does not provide any mechanism to avoid lock-starvation~\footnote{Intel Manual 3A Section 8.1 LOCKED ATOMIC OPERATIONS}, 
Therefore, if two or more threads try to execute a CAS on the same memory cell, only one will succeed, and all the others will have to try again.
Thus, a thread will retry to execute CAS until it succeeds. Once a competitor thread wins the tie, it will have to "sample" a new cache line to write to, so it might or might not be the same.
Therefore the probability of winning in the beginning is:
\[p(0, \NumberOfThreads) = \frac{l!}{(l - \NumberOfThreads)! l^{\NumberOfThreads}} \qquad p(k, 1) = 1\]
Each time we fail, the probability follows this recurrence relation:
\[p(k, \NumberOfThreads) = \frac{1}{l} p(k-1, \NumberOfThreads) + \frac{l-1}{l} p(k-1, t-1)\]

Therefore asymptotically, we know that:
\[p(k, \NumberOfThreads) \ge \frac{l-1}{l} p(k-1, t-1) = \left(\frac{l-1}{l}\right)^k = \left(1 - \frac{1}{l}\right)^k\]

\end{proof}