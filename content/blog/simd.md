+++
title = "A Practical Introduction To SIMD"
+++

simd is powerful
i talk about it a lot
lets discuss why it's so useful, what it can be used for, and some concrete examples

# What is SIMD

SIMD stands for single-instruction, multiple-data. In this context, it's referring to a type of machine instruction. To this end, when someone says "SIMD," they usually mean "SIMD instruction."

# Why do people use it

# When not to use SIMD

Don't use SIMD if
 - performance isn't important
 - time or maintainability are important

# How to use SIMD and understanding intrinsics

SIMD is extremely powerful. 



Why is SIMD useful? Let's take a simple and contrived example in C.

Say we have two arrays, `a` and `b`, both containing 8 ints. And we want to add each element to its corresponding array and write it to a 3rd array, `c`. 

SIMD is a type of machine instruction. When someone says "SIMD," they are typically referring to SIMD instructions. These are machine instructions that operate on wider lanes of data at once than a typical instruction.

```c
void add_arrs(int a[8], int b[8], int* c) {
    for (int i = 0; i < 8; i++) {
        c[i] = a[i] + b[i];
    }
}
```

```c
#include <x86intrin.h>

void add_arrs_simd(int a[8], int b[8], int* c) {
    __m256i a_avx = _mm256_loadu_si256((__m256i*)&a[0]);
    __m256i b_avx = _mm256_loadu_si256((__m256i*)&b[0]);
    __m256i sum = _mm256_add_epi8(a_avx, b_avx);

    _mm256_storeu_si256((__m256i*)&c[0], sum);
}
```