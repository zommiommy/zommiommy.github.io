+++
title = "Road to dot-product: Pt.1 How to make a Python library in Rust"
description = "How to create a fast python library using Rust"
date = 2021-12-23
draft = true
+++

```sh
# Download the sources for python
wget https://www.python.org/ftp/python/3.10.1/Python-3.10.1.tar.xz -O Python.tar.xz
# Extract them
tar -xvf ./Python.tar.xz

# Download zig
wget https://ziglang.org/builds/zig-linux-x86_64-0.11.0-dev.935+bb62d5105.tar.xz -O zig.tar.xz
# Extract them
tar -xvf ./zig.tar.xz
# Rename the output folder to just zig
mv zig-linux-x86_64-0.11.0-dev.935+bb62d5105 zig

# Set the downloaded zig cc as the HOST C and C++ compiler
export TRIPLE="aarch64-linux-gnu"
export HOST_CC="${PWD}/zig/zig cc"
# Set the TARGET compiler to the different arch 
export CC="${HOST_CC} -target ${TRIPLE}.2.28"
export AR="${PWD}/zig/zig ar"

export PREFIX="/tmp/python_${TRIPLE}"

# (OPTIONAL) Disable optimization on python, and add debug symbols
# for easier debugging
export OPT="-O0"
export CFLAGS="$CFLAGS -g"

export ac_cv_file__dev_ptmx=no
export ac_cv_file__dev_ptc=no

# Patch readelf away (?)

cd Python
./configure --with-lto=no --host=${TRIPLE} --build=x86_64-linux-gnu --prefix="$PREFIX" --disable-ipv6


```