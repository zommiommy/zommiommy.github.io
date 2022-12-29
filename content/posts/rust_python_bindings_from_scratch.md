+++
title = "Rust-Python bindings from scratch"
description = "How to write a python module in Rust with no external dependancies"
date = 2021-12-23
draft = true
+++


Create a new rust lib:
```bash
cargo new --lib "stones"
```

Make it create a `.so`:
```toml
[package]
name = "stones"
version = "0.1.0"
edition = "2021"

[lib]
name = "stones"
crate-type = ["cdylib"]
```


```rust
```