+++
title = "Fuzzing Pyo3 bindings with Atheris"
date = 2021-09-05
+++
Atheris is the new shiny toy I played with solving Google CTF pwns.
It's a Python coverage guided fuzzer, based on LLVM's libFuzzer, that instrument
python bytecode to get coverage. 

I'm one of the two developer of [ensmallen](https://github.com/AnacletoLAB/ensmallen), our high-performance graph manipulation library which is written in Rust and has Pyo3 Python bindings.

The ensmallen's rust crate is already fuzzed using both libFuzzer and Honggfuzz, so
naturally we would love to also fuzz the bindings so that we can have a rock-solid
library.


### Creating an harness
To build a proof of concept we will fuzz our [TFIDF (OKAPI BM 25)](https://en.wikipedia.org/wiki/Okapi_BM25) implementation.
This function is the one used to pre-compute the weights for our raccomender system
we implemented as the `__getattr__` magic method of our Graph class.

For our goal of fuzzing it we just need to know that it take a list of documents,
where a document is a list of the id of the terms contained in each document.

Following the great tutorial on Atheris's Github, I bult a small harness that
just create random arguments (with the only caveat that I limited the number of documents and the maximum document length to an `u8` to avoid OOM errors).

```python
#!/usr/bin/python
import sys
import atheris
import ensmallen  

def fuzz_me(input_bytes: bytes):
    fdp = atheris.FuzzedDataProvider(input_bytes)
    try:
        ensmallen.preprocessing.okapi_bm25_tfidf_int(
            [   
                [
                    fdp.ConsumeUInt(4)
                    for _ in range(fdp.ConsumeUInt(1))
                ]
                for _ in range(fdp.ConsumeUInt(1))
            ],
            fdp.ConsumeUInt(4),
            fdp.ConsumeUInt(4),
            False,
        )
    except ValueError:
        pass

atheris.instrument_all()
atheris.Setup(sys.argv, fuzz_me)
atheris.Fuzz()
```

We can test this harness:
```bash
$ python ens.py
# ...
#2      INITED cov: 1 ft: 1 corp: 1/1b exec/s: 0 rss: 40Mb
#122    NEW    cov: 2 ft: 2 corp: 2/4b lim: 4 exec/s: 0 rss: 40Mb L: 3/3 MS: 5 CrossOver-CrossOver-ChangeBit-ChangeBit-CMP- DE: "\x00\x00"-
#137    REDUCE cov: 2 ft: 2 corp: 2/3b lim: 4 exec/s: 0 rss: 40Mb L: 2/2 MS: 5 ChangeBit-ChangeBit-ChangeBinInt-EraseBytes-PersAutoDict- DE: "\x00\x00"-
#147    REDUCE cov: 2 ft: 2 corp: 2/2b lim: 4 exec/s: 0 rss: 40Mb L: 1/1 MS: 5 InsertByte-EraseBytes-EraseBytes-ChangeBit-CrossOver-
#2048   pulse  cov: 2 ft: 2 corp: 2/2b lim: 21 exec/s: 1024 rss: 40Mb
#4096   pulse  cov: 2 ft: 2 corp: 2/2b lim: 38 exec/s: 1365 rss: 40Mb
```

Easy? 

Atheris only instrument the python bytecode, so our native library get no coverage!
So, to do some useful fuzzing we must instrument the rust code.

This can be done by using these extra flags during compilation:
```bash
RUST_FLAGS="-Ctarget-cpu=native -Zinstrument-coverage -Cpasses=sancov -Cllvm-args=-sanitizer-coverage-level=4  -Cllvm-args=-sanitizer-coverage-trace-compares  -Cllvm-args=-sanitizer-coverage-inline-8bit-counters  -Cllvm-args=-sanitizer-coverage-pc-table -Cllvm-args=-sanitizer-coverage-stack-depth --verbose -Zsanitizer=address" maturin develop --release
```
(Most parameters an be found [here](https://clang.llvm.org/docs/SanitizerCoverage.html))

To properly run the harness with the now instrumented library we should preload Atheris's sanitizers:
```bash
$ LD_PRELOAD="$(python -c "import atheris; print(atheris.path())")/asan_with_fuzzer.so" python ens.py
# ...
#47528  REDUCE cov: 703 ft: 3136 corp: 253/8280b lim: 156 exec/s: 148 rss: 1168Mb L: 39/144 MS: 3 CrossOver-ChangeBit-EraseBytes-
#47840  REDUCE cov: 703 ft: 3136 corp: 253/8279b lim: 156 exec/s: 148 rss: 1168Mb L: 114/144 MS: 2 EraseBytes-ChangeBinInt-
#48414  NEW    cov: 703 ft: 3137 corp: 254/8395b lim: 156 exec/s: 148 rss: 1168Mb L: 116/144 MS: 4 ChangeBinInt-ShuffleBytes-ChangeBinInt-CopyPart-
#48857  NEW    cov: 704 ft: 3139 corp: 255/8541b lim: 156 exec/s: 147 rss: 1168Mb L: 146/146 MS: 3 InsertByte-CopyPart-InsertRepeatedBytes-
#49566  NEW    cov: 704 ft: 3140 corp: 256/8627b lim: 163 exec/s: 147 rss: 1168Mb L: 86/146 MS: 4 ChangeByte-PersAutoDict-CrossOver-CrossOver- DE: "\x08\x00"-
#50624  NEW    cov: 704 ft: 3141 corp: 257/8773b lim: 170 exec/s: 147 rss: 1168Mb L: 146/146 MS: 3 ChangeBinInt-CrossOver-PersAutoDict- DE: "\x0f\x00"-
#52580  REDUCE cov: 704 ft: 3141 corp: 257/8771b lim: 184 exec/s: 146 rss: 1168Mb L: 123/146 MS: 1 EraseBytes-
#52835  REDUCE cov: 704 ft: 3141 corp: 257/8769b lim: 184 exec/s: 146 rss: 1168Mb L: 55/146 MS: 5 ChangeByte-InsertByte-EraseBytes-CopyPart-ChangeBinInt-
#52903  REDUCE cov: 704 ft: 3141 corp: 257/8768b lim: 184 exec/s: 146 rss: 1168Mb L: 108/146 MS: 3 EraseBytes-ShuffleBytes-InsertByte-
#53748  NEW    cov: 704 ft: 3142 corp: 258/8958b lim: 191 exec/s: 146 rss: 1168Mb L: 190/190 MS: 5 CopyPart-ShuffleBytes-InsertRepeatedBytes-ShuffleBytes-CopyPart-
```

Finally it works and we get coverage over native rust code and we can effectively fuzz
our python bindings!

BTW while libFuzz is easy to use thanks to LLVM, its performance is garbage (it uses only 1 core), in the future I'd like to switch to fuzzers that **actually** use the system resources.

### Troubleshooting
If you instrument the library but you don't add the `LD_PRELOAD`, it would crash and burn because we need to link it against the sanitizers:
```bash
$ python ens.py            
Traceback (most recent call last):
  File "ens.py", line 4, in <module>
    import ensmallen
  File "/home/zom/anaconda3/lib/python3.8/site-packages/ensmallen/__init__.py", line 2, in <module>
    from .ensmallen import edge_list_utils # pylint: disable=import-error
ImportError: /home/zom/anaconda3/lib/python3.8/site-packages/ensmallen/ensmallen.cpython-38-x86_64-linux-gnu.so: undefined symbol:
 __sancov_lowest_stack
```

### References
- [SanitizerCoverage](https://clang.llvm.org/docs/SanitizerCoverage.html)
- [atheris](https://github.com/google/atheris)
- [atheris native extensions instrumentation](https://github.com/google/atheris/blob/master/native_extension_fuzzing.md)
- [rust sanitizers](https://doc.rust-lang.org/beta/unstable-book/compiler-flags/sanitizer.html)
- [cargo-fuzz](https://github.com/rust-fuzz/cargo-fuzz)
- [Rust Fuzz Book](https://rust-fuzz.github.io/book/cargo-fuzz.html)