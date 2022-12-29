+++
title = "Road to dot-product: Pt.1 How to make a Python library in Rust"
description = "How to create a fast python library using Rust"
date = 2021-12-23
draft = true
+++

I

## Installing Rust
Since we are going to work in Rust, we need to install it. this can be easily done following <https://rustup.rs/>.
Since we are going to use `Pyo3`, we need to install a nightly version.
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

If you have already installed rust with a stable version you can run the following command to switch to the nightly channel:
```bash
rustup default nightly
```

Before proceeding ahead, ensure that your version of Rust is the latest, and update it:
```bash
rustup update
```

## Creating a Python native library in Rust with Pyo3
Let's start by creating an empty rust project:
```bash
cargo new --lib "dotproduct"
```

The first thing to do is to add [`crate-type = ["cdylib"]`](https://doc.rust-lang.org/reference/linkage.html) to the `Cargo.toml` file, so that rust will produce a
 compiled shared library (a `.so`, `.dylib`, `.dll`) like the ones we would get if we were to compile C code. 
Also we are going to add `pyo3` as a bindings helper to produce a valid python library. The line `features = ["extension-module", "abi3-py36"]` tells `Pyo3` to just use the [stable ABI](https://docs.python.org/3/c-api/stable.html) so that the compiled library will be compatible with any python version >= 3.2. While this slightly limits what our library can do on Python, it saves us to compile a different file for **every version we want to support**.

Your `Cargo.toml` should look like this:
```toml
[package]
name = "dotproduct"
version = "0.1.0"
edition = "2021"

[dependencies]

[lib]
name = "dotproduct"
crate-type = ["cdylib"]

[dependencies.pyo3]
version = "0.13.2"
features = ["extension-module", "abi3-py36"]
```

**Techincal details:**
> Python doesn't directly expose a way to create modules in Rust.
> Python can only be extended through a shared library which will be loaded and executed as if it was builtin Python code.
> We are going to use Rust t
> So we are going to use the [C functions](https://docs.python.org/3/extending/extending.html) through a library called [`Pyo3`](https://pyo3.rs/v0.17.3/) that will help us reduce the writing overhead, but it's completely optional.
> https://peps.python.org/pep-0387/

### Base setup

### Compilation