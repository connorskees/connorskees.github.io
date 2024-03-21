+++
title = "Multiplication Algorithms"
+++

I'm intimidated by modern mathematical algorithms, like those used for [float parsing](https://arxiv.org/abs/2101.11408) and [formatting](https://dl.acm.org/doi/10.1145/3192366.3192369), or famously the [inverse square root](https://www.youtube.com/watch?v=p8u_k2LIZyo). 

I do love math, but it often feels like these algorithms are unapproachable to casual understanding. That is, effectively understanding the intuition behind these algorithms requires a lot of pre-requisite reading of logic- and notation-dense papers.

I don't necessarily have the time or mathematical background to dig into the existing literature for a lot of these algorithms. But recently, I've been interested in algorithms for multiplication, after hearing that some bigint libraries rely on the fast Fourier transform to perform multiplication in some scenarios.

The below is what I hope to be the most comprehensive overview of modern multiplication algorithms with explanations written in plain English.

<!-- - [Integers](#integers)
  - [Looping](#looping)
  - [Long Multiplication](#long-multiplication)
  - [Karatsuba (1962)](#karatsuba-1962)
  - [Toom–Cook](#toomcook)
  - [Schönhage–Strassen (1971)](#schönhagestrassen-1971)
  - [Fürer (2007)](#fürer-2007)
  - [De–Kurur–Saha–Saptharishi (2008)](#dekurursahasaptharishi-2008)
  - [Harvey–van der Hoeven–Lecerf (2014)](#harveyvan-der-hoevenlecerf-2014)
  - [Covanov–Thomé (2015)](#covanovthomé-2015)
  - [Harvey–van der Hoeven (2019)](#harveyvan-der-hoeven-2019)
- [Reals](#reals)
  - [Floats](#floats)
  - [Fixed point](#fixed-point)
- [Hardware Algorithms](#hardware-algorithms)
  - [Shift + Add](#shift--add)
  - [Modified Baugh-Wooley](#modified-baugh-wooley)
  - [Booth (1950)](#booth-1950)
  - [Wallace Trees (1964)](#wallace-trees-1964)
  - [Dadda Multiplier (1965)](#dadda-multiplier-1965)
- [Modular Multiplication](#modular-multiplication)
  - [Kochanski](#kochanski)
  - [Montgomery](#montgomery)
- [Floats](#floats-1) -->

# Integers

## Looping

Fundamentally multiplication describes successive additions. If we say `x * 2`, we can expand this as `x + x`. Similarly, `x * 4` is the same as `x + x + x + x`. 

The easiest way to implement this in code would be a `for` loop:

```rust
fn multiply(multiplicand: i32, multiplier: i32) -> i32 {
    let mut product = 0;
    
    for _ in 0..multiplier {
        product += multiplicand;
    }

    product
}
```

We loop `multiplier` times, adding `multiplicand` to the total product each time. For right now we're ignoring complexities like overflow.

For small multipliers, this doesn't seem so bad. However, as our multiplier grows, the number of additions we have to perform grows similarly. In the worst case scenario, if our muliplier is `u32::MAX`, we'd be performing over 4 billion additions -- this is pretty slow.

Ideally our algorithm wouldn't scale with the value of our multiplier, but rather the scale. That is, our algorithm would get slower relative to the number of digits used to represent the number, not the actual value it contains.

## Long Multiplication

Long multiplication solves the problem of scaling relative to the number of digits.

It's also likely the way you were taught in school to multiply two numbers. If we look at a simple base-10 example:

```
  25
*  7
----
```

We multiply all individual digits and then sum them up:

```
20 * 7 + 5 * 7 = 140 + 35 = 175
```

If our multiplier has two digits:

```
       25
*      34
---------
    4 * 5
+  4 * 20
+  30 * 5
+ 30 * 20
---------
       20
+      80
+     150
+     600
---------
      850
```

We end up performing 4 additions and multiplications.

Long multiplication has a time complexity of O(n<sup>2</sup>), and so it isn't really suitable for extremely large numbers. You may also be familiar with other methods for doing multiplication by hand; in general, these are all also O(n<sup>2</sup>).

Although this algorithm is quadratic, it also is quite simple to implement and has a pretty low constant factor. This makes it suitable -- and even optimal -- for small (say, <=64 bit) integers. This algorithm forms the basis of most hardware implementations of multiplication, as we'll discuss [later in this post](#hardware-algorithms).

<!-- https://gmplib.org/manual/Basecase-Multiplication -->

## Karatsuba (1962)

Anatoly Karatsuba's algorithm for multiplication was the first to break the O(n<sup>2</sup>) complexity barrier. Previously it had been conjectured that no algorithm could do better than O(n<sup>2</sup>).

Karatsuba is a recursive divide and conquer algorithm.

If we split our multiplication up into multiple separate multiplications, we can re-use some of the intermediate results.

The algorithm starts by splitting our multiplicand and multiplier in half, getting four numbers total. For example, we would split the number `23` into `20 + 3`, or the number `123456` into `123000 + 456`. 

More formally: given a number of `n` digits, we split it into two numbers `a` and `b` such that our original number is equal to $$a \times 10^{\frac{n}{2}} + b$$.

Let's look at a multiplication of two numbers, `x` and `y`. Using the splitting idea we just mentioned,

$$
x = a + b
$$
<!-- \times 10^{\frac{n}{2}} -->
<br>
$$
y = c + d
$$
<!-- \times 10^{\frac{n}{2}} -->

This is the same equation used above, but for our multiplier `y` we use the variables `c` and `d`. If we substitute these values into our $$x \times y$$ equation, we get

$$
x \times y = (a + b)(c + d)
$$

If we distribute everything out:

$$
x \times y = (a \times 10^{\frac{n}{2}}) \times (c \times 10^{\frac{n}{2}}) + (ad \times 10^{\frac{n}{2}}) + (bc  \times 10^{\frac{n}{2}}) + bd
$$

Simplifying:

$$
x \times y = ac \times 10^{n} + (ad + bc) \times 10^{\frac{n}{2}} + bd
$$

<!-- What does this get us?

 - https://www.mathnet.ru/php/archive.phtml?wshow=paper&jrnid=dan&paperid=26729&option_lang=eng
 - http://www.ccas.ru/personal/karatsuba/divcen.pdf
 - https://gmplib.org/manual/Karatsuba-Multiplication
 - https://www.classes.cs.uchicago.edu/archive/2004/winter/37000-1/handouts/karatsuba.pdf -->

## Toom–Cook

The Toom–Cook algorithm is a generalization of Karatsuba multiplication. Where Karatsuba multiplication splits a number into 2 constituent parts, Toom–Cook splits the number into a potentially arbitrary number of parts. Though, most commonly you'll see 3 parts (Toom 3) or 4 parts (Toom 4). Karatsuba, in this way, can also be thought of as Toom 2. Regular long multiplication is Toom 1.


 <!-- - http://www.bodrato.it/toom-cook/
 - http://cs.indstate.edu/~syedugani/ToomCook.pdf
 - https://arxiv.org/pdf/1602.02740.pdf
 - https://gmplib.org/manual/Toom-3_002dWay-Multiplication
 - https://gmplib.org/manual/Toom-4_002dWay-Multiplication
 - https://gmplib.org/manual/Higher-degree-Toom_0027n_0027half -->

## Schönhage–Strassen (1971)
 <!-- 
 https://gmplib.org/manual/FFT-Multiplication
 https://journals.sagepub.com/doi/10.1177/10943420221077964
 http://www.allisons.org/ll/AlgDS/Arithmetic/SandS/
 https://tonjanee.home.xs4all.nl/SSAdescription.pdf
  -->

Schönhage–Strassen is also sometimes called "FFT multiplication," as the core optimization of this algorithm is that it uses the fast Fourier transform to achieve O($$n \log n \log \log n$$) runtime.

Schönhage–Strassen works by splitting the multiplier and multiplicand, similar to Karatsuba, and constructs two polynomials with the decomposed integers as coeffecients. It can then make use of [existing algorithms to multiply two polynomials in O($$n \log n$$)](http://www.cs.toronto.edu/~denisp/csc373/docs/tutorial3-adv-writeup.pdf) time.

To understand this algorithm more concretely, we first have to understand Fourier transforms, the fast Fourier transform, and some abstract algebra.

## Fürer (2007)
 
 - paper: https://ivv5hpp.uni-muenster.de/u/cl/WS2007-8/mult.pdf
 - https://ir.lib.uwo.ca/cgi/viewcontent.cgi?article=7368&context=etd
 

## De–Kurur–Saha–Saptharishi (2008)
 - https://arxiv.org/abs/0801.1416

## Harvey–van der Hoeven–Lecerf (2014)
 - https://arxiv.org/abs/1407.3361

## Covanov–Thomé (2015)
 - https://arxiv.org/abs/1502.02800

## Harvey–van der Hoeven (2019)
- https://annals.math.princeton.edu/2021/193-2/p04
- https://hal.science/hal-02070778v2/document


# Reals

## Floats

## Fixed point

# Hardware Algorithms

One of the simpler and faster ways to multiply two numbers on modern computers is:

```asm
mov rax, 25
mul rax, 34
; rax = 850
```

 
 <!-- - Dadda multiplier (https://en.wikipedia.org/wiki/Dadda_multiplier)
 - https://acg.cis.upenn.edu/milom/cis371-Spring08/lectures/04_integer.pdf -->

## Shift + Add
## Modified Baugh-Wooley
 <!-- - https://www.ijert.org/research/design-of-fixed-width-multiplier-using-baugh-wooley-algorithm-IJERTV2IS110751.pdf
 - https://www.ece.uvic.ca/~fayez/courses/ceng465/lab_465/project2/multiplier.pdf
 - https://stackoverflow.com/questions/54268192/understanding-modified-baugh-wooley-multiplication-algorithm -->
## Booth (1950)
 <!-- - https://en.wikipedia.org/wiki/Booth%27s_multiplication_algorithm
 - http://bwrcs.eecs.berkeley.edu/Classes/icdesign/ee241_s00/PAPERS/archive/booth51.pdf -->
## Wallace Trees (1964)
 <!-- - https://en.wikipedia.org/wiki/Wallace_tree (1964) -->
## Dadda Multiplier (1965)
<!-- - https://en.wikipedia.org/wiki/Dadda_multiplier -->


# Modular Multiplication

## Kochanski
 <!-- - https://en.wikipedia.org/wiki/Kochanski_multiplication -->
## Montgomery
 <!-- - https://en.wikipedia.org/wiki/Montgomery_modular_multiplication -->

# Floats

https://github.com/microsoft/referencesource/blob/master/System.Numerics/System/Numerics/BigIntegerBuilder.cs
https://github.com/rust-num/num-bigint/blob/f09eee83f174619ac9c2489e3feec62544984bc5/src/biguint/multiplication.rs#L95