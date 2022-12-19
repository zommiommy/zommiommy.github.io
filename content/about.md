+++
title = "About "
path = "about"
+++


### TLDR
It's a me, zom. I like Math and Computers, I'm what happend when you can't choose a single career path.

![](/pfp.jpg)

### Security
I play Capture The Flags with the Italian Teams [@mHackeroni](https://mhackeroni.it/) (4 times [DEFCON](https://defcon.org/html/links/dc-ctf.html) finalists) and [@towerofhanoi](https://toh.necst.it/) (The oldest CTF team in Italy).  
I specialize in Reverse engineering and Pwning. Expecially on the topics of Fuzzing, Compilers, Interpreters, and Emulation.

![](/mhack_pix_sticker.png)

### Machine Learning
With my colleagues, I implemented high-performance Graph Machine learning models in Rust, currently I'm focussing on scalable Natural Language Processing.

I collaborated and applied my research with the following laboratories: [BBOT @ LBL](http://www.berkeleybop.org/), [Monarch Institute](https://www.monarch.edu.au/), [The Jackson Laboratory](https://www.jax.org/), [AnacletoLAB](https://anacletolab.di.unimi.it/), [San Raffaele](https://www.hsr.it/).
These collaborations resulted in [several published papers](https://scholar.google.com/citations?hl=en&user=9oTPcNUAAAAJ) on drug reporpousing.

### Computer Science & Math
I enjoy compilers, high performance code, system programming, low level, 
Statistics, Information Theory, Category Theory, and Optimizzation.

I've implemented in Rust succint and lock-less datastructures such as:
- [WebGraph](https://webgraph.di.unimi.it/)
- EliasFano
- RsDict
- RoaringBitMap

In the last 2 years I focussed on graph data-structures and parallel algorithms for graphs.

### Hardware
One of my current side-projects is to write an USB1.1 sniffer using a cheap Raspberry Pico.
USB1.1 (Full Speed) has a bandwidth of [12MHz](https://support.saleae.com/faq/technical-faq/what-sample-rate-is-required), and we need to sample around [4 times faster than the bandwidth](https://support.saleae.com/faq/technical-faq/what-sample-rate-is-required).
On a Raspberry Pico, which runs at 125 MHz, we have $$\frac{125\text{MHz}}{4 \cdot 12\text{MHz}} \approx 2.6 \frac{\text{Cycles}}{\text{Sample}}$$.
I'm trying to achieve this writing a baremetal os in Rust which uses the PIO to sample the data and DMA it directly to the
USB controller buffers. 

My usecase for this project is to experiment with electromagnetic fault injections to dump the bootroom of a chip. 