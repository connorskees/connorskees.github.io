+++
title = "Understanding Modern Implementations of CRC32"
+++
<!-- title = "An Approachable Explanation of SIMD Accelerated crc32" -->

`CRC32` is a common checksum algorithm, notably found in PNG files. The "CRC" portion stands for cyclic-redundancy-check, as the algorithm is largely based around cyclic codes. The 32 refers to the number of bits 

Modern implementations of this algorithm are heavily optimized and can take a while to understand. This article attempts give a simple explanation for the scalar implementation of crc32, and then work through the intuition behind the SIMD implementation.

The vectorized portion of this post is largely based on this Intel paper from 2009, [Fast CRC Computation for Generic Polynomials Using PCLMULQDQ Instruction](https://www.intel.com/content/dam/www/public/us/en/documents/white-papers/fast-crc-computation-generic-polynomials-pclmulqdq-paper.pdf).

### CRC

<!-- https://en.wikipedia.org/wiki/Computation_of_cyclic_redundancy_checks -->
<!-- https://github.com/rurban/crcutil -->
<!-- https://en.wikipedia.org/wiki/Cyclic_redundancy_check -->
<!-- https://github.com/cloudflare/zlib -->

### Multiply _with_ carry

If you were given a pencil and paper, how would you compute 26 * 7 by hand? Some of you might skip a few steps in your head if you compute this mentally, but imagine how your 3rd grade self would solve this. You'd probably set up something that looks like this:

```
  26
*  7
----
```

And then you'd start by multiplying the 6 times the 7. If the result is over 10, you'd split up the number and write the number in the ones place below the bar, and the number in the 10s place above the 2.

```
  4
  26
*  7
----
   2
```

That 4 is our carry. When we multiply the 7 by the 2, we add the carry to the final result and get 18. So our full number is 182. 

If we don't add this carry, we're performing an operation called carry-less multiply, which underlies the basis of a number of cryptographic algorithms and the SIMD solution to CRC32.

### Multiply _with_ carry in binary

### Carry-less Multiply

### How CPUs multiply

How would you quickly multiply a number by 3? You could do 

<!-- Most programmers are familiar with simple bit hacks. If you bitshift to the left (`>>`), you can quickly divide a number by 2. If you bitshift to the right (`<<`) you can do the inverse and multiply by 2. -->


<!-- https://lemire.me/blog/2015/10/26/crazily-fast-hashing-with-carry-less-multiplications/ -->
<!-- https://github.com/lemire/StronglyUniversalStringHashing -->
<!-- https://arxiv.org/abs/1503.03465 -->

### Cyclic Codes

<!-- https://home.work.caltech.edu/~ling/webs/EE127/EE127A/handout/Ch8.pdf -->
<!-- https://en.wikipedia.org/wiki/Cyclic_code -->


<!-- https://www.intel.com/content/dam/www/public/us/en/documents/white-papers/crc-iscsi-polynomial-crc32-instruction-paper.pdf -->
<!-- https://lxp32.github.io/docs/a-simple-example-crc32-calculation/ -->
<!-- https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#expand=4474&ig_expand=4673,7124&cats=Cryptography&text=crc32 -->