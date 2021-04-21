---
title: "How to check for memory leak"
date: 2020-03-01
categories: [Development,General]
tags: [memory,valgrind]
---

# Install

To check for memory leak we will use [valgrind](https://www.valgrind.org/) tools.
First we need to compile and install it:

Install `valgrind`:
``` bash
$ wget https://sourceware.org/pub/valgrind/valgrind-3.16.1.tar.bz2
$ tar -xf valgrind-3.16.1.tar.bz2 && rm valgrind-3.16.1.tar.bz2
$ cd valgrind-3.16.1
$ ./configure
$ make -j$(nproc)
$ sudo make install
```

Install `kgraphviewer`:
```bash
$ sudo rm /etc/apt/preferences.d/nosnap.pref
$ sudo apt update
$ sudo apt install snapd
$ sudo snap install kgraphviewer --candidate
```

# Checking

## Memory checking

[Memcheck](https://www.valgrind.org/docs/manual/mc-manual.html) detects memory-management problems by redefining `malloc/new/free/delete` methods. As a result it can detects:

* Access violation;
* Using uninitialized values in dangerous ways;
* Memory leaks;
* Invalid calls of releasing of heap memory blocks (double frees, mismatched frees);
* Memory overlapping of source and destination memory blocks.

Prepare (+debug; -opmization):
```bash
$ g++ -g -O0 example.cpp -o example.x
```

Using:
```bash
$ valgrind --tool=memcheck --leak-check=yes --show-reachable=yes example.x
```

Options:
```bash
$ valgrind --tool=memcheck --help-dyn-options
```

## Cachegrind checking

[Cachegrind](https://www.valgrind.org/docs/manual/cg-manual.html) is a cache profiler. It identifies the number of cache misses, memory references and instructions executed for each line of source code, with per-function, per-module and whole-program summaries.

Prepare (-debug; +opmization):
```bash
$ g++ -O2 example.cpp -o example.x
```

Using:
```bash
$ valgrind --tool=cachegrind example.x
```

## Massif checking

[Massif](https://www.valgrind.org/docs/manual/ms-manual.html) is a heap profiler. It performs detailed heap profiling by taking regular snapshots of a program's heap. It produces a graph showing heap usage over time, including information about which parts of the program are responsible for the most memory allocations.

Prepare (+debug; +opmization):
```bash
$ g++ -g -O2 example.cpp -o example.x
```

Using:
```bash
$ valgrind --tool=massif --massif-out-file=example.ms example.x
$ ms_print ./example.ms > example.print
```



