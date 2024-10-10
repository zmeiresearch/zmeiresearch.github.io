---
layout: post
title:  "GCC compiler performance"
author: Ivan
date:   2023-09-05 14:56:24 +0300
categories: gcc embedded benchmark
---

## What?
A totally unscientific benchmark of GCC on a real project that I'm currently working.

## Why?
Because switching from GCC 6 to GCC12 felt slow and I wanted to know exactly how much slower it really is.

## Background
It's a real project, a custom firmware for a Cortex-M4F MCU (Atmel AT-SAME51 to be specific). It's developed mainly in plain old C, with a bit of C++ thrown in. Even when C++ is used, it's mostly as C-with-classes. The specific compiler options in place are:
```
-Os 
-mthumb -mcpu=cortex-m4 -mfloat-abi=hard -mfpu=fpv4-sp-d16 
-ffunction-sections -mlong-calls 
-Waddress -Wformat-nonliteral -Wformat-security -Wformat -Winit-self -Wmissing-declarations 
-Wno-multichar -Wunreachable-code -Wwrite-strings -Wpointer-arith -Wfloat-equal -Wparentheses 
-Wunused-parameter -Wunused-variable -Wreturn-type -Wuninitialized -Wextra -Wall  
-std=gnu11 -MD -MT
```

## Methodology
Execute the below script two times, changing `PATH` to point to different compiler versions.

```shell
test_compile () {
    sleep 30
    make clean
    time make -j4 | grep real
}

for i in {1..10}; do test_compile; done
```

Basically compile the firmware 10 times in a loop, using up to 4 parallel compiler invokations - as this is the number of cores that I have on my setup.

## Raw Results
### GCC 6.3.1
```shell
$ arm-none-eabi-gcc --version 
arm-none-eabi-gcc (Atmel build: 508) 6.3.1 20170620 (release) [ARM/embedded-6-branch revision 249437]

$ ./test_compile.sh | grep real
real	0m13,659s
real	0m14,090s
real	0m15,397s
real	0m14,367s
real	0m14,815s
real	0m15,611s
real	0m18,213s
real	0m15,756s
real	0m18,644s
real	0m17,026s
```
### GCC 12.2
```shell
$ arm-none-eabi-gcc --version 
arm-none-eabi-gcc (Arm GNU Toolchain 12.2.MPACBTI-Rel1 (Build arm-12-mpacbti.34)) 12.2.1 20230214

$ ./test_compile.sh | grep real
real	0m20,078s
real	0m22,832s
real	0m19,660s
real	0m19,067s
real	0m18,097s
real	0m18,159s
real	0m19,469s
real	0m19,113s
real	0m18,993s
real	0m20,311s
```

## Results:

### Build time
Build time in seconds:

| GCC version | Avg. time | StdDev |
| --- | --- | --- |
| 6.3.1 | 15.757 | 1.616 |
| 12.2 | 19.578 | 1.281 |

### Binary size
Sizes in bytes as obtained from `arm-none-eabi-size`:

| GCC version | Text | Data | Bss | Total |
| --- | --- | --- | --- | --- |
| 6.3.1 | 54704	| 228 | 129344 | 184276 |
| 12.2 | 53752 | 220 | 129656 | 183628 |


## Conclusion
**GCC 12** is around 25% slower, which is noticeable even with this small project if doing a full build. I don't have a statistic how much of that time is spent between the compiler and linker but subjectively even building and linking only a few files do seem slower.
On the plus side, **GCC 12** produces a binary that's ~600 bytes smaller - which is rarely a deal-breaker but it may be under specific circumstances.  

