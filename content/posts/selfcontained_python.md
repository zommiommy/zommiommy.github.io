+++
title = "Self-contained Python for difficult environments"
date = "2023-07-11"
description = "Write up on how to make a self-contained Python environment to be able to run it on a system with no internet access and no root access."
draft = true
+++

Download and extract python 3.6 (EOL ofc :) )
```bash
wget https://www.python.org/ftp/python/3.6.6/Python-3.6.6rc1.tar.xz
tar -xvf Python-3.6.6rc1.tar.xz
```
Download and extract openssl 3.1.1
```bash
wget https://github.com/openssl/openssl/releases/download/OpenSSL_1_1_1u/openssl-1.1.1u.tar.gz
tar -xvf openssl-1.1.1u.tar.gz
```

Start a docker container with manylinux1_x86_64 (Centos 7 based, EOL ofc :) )
```bash
docker run -it -v "$PWD:/io" quay.io/pypa/manylinux1_x86_64 bash
```
Compile stuff
```bash
# Make the folder where we will install python
mkdir -p /opt/sc_python/openssl

# Compile openssl
cd /io/openssl-1.1.1u
./config --prefix="/opt/sc_python/openssl" --openssldir="/opt/sc_python/openssl"
make -j$(nproc)
make -j$(nproc) install

# Compile python
cd /io/Python-3.6.6rc1
./configure --with-ensurepip="upgrade" --enable-optimizations --prefix="/opt/sc_python"
make -j$(nproc)
make -j$(nproc) install
```
