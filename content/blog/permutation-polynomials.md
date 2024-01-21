+++
title = "Permutation Polynomials"
date = "2023-09-19"
+++

A permutation polynomial is any polynomial which acts as a permutation of a given set of numbers. In other terms, the polynomial defines a bijection from x to f(x), or a 1:1 mapping of unique inputs to unique outputs and also that each x value in the input domain appears in the output range.

Generally these polynomials operate on a finite range of numbers, most of them having their input modulo some n.

The simplest such polynomial is $$f(x) = x$$, because every unique input is mapped to a unique output -- for example, 1 goes to 1, 2 goes to 2, 3 goes to 3, and so on. A slightly more complex example is $$f(x) = 3x$$ for the integers mod 4. In this case, we have the property that:

$$
f(0) = 0
$$
<br>
$$
f(1) = 3
$$
<br>
$$
f(2) = 2
$$
<br>
$$
f(3) = 1
$$

This gives us [0, 1, 2, 3] => [0, 3, 2, 1] or a permutation of the set of integers mod 4.

The rest of this post will be about binary permutation polynomials, or those that are evaluated modulo a power of two. These polynomials have certain properties that make them easier to work with, and are what one would most commonly find in the wild, as the bit-widths of integers in most programming languages are powers of two.

### Testing

Determining whether an arbitrary polynomial is a permutation polynomial has not been shown to be possible using an algorithm that executes in polynomial time. This can make it computationally expensive to determine whether arbitrary polynomials of high degree are permutation polynomials.

Polynomials that are modulo a power of two, however, are much easier to reason about and determining whether a given binary polynomial is a permutation polynomial is possible in linear time. A polynomial modulo a power of two is a permutation polynomial if and only if:

1. the 1-degree monomial (i.e. $$x$$) has an odd coefficient
2. the sum of the coefficients of all even degree monomials _excluding 0_ (e.g. $$x^2$$, $$x^4$$, $$x^6$$, ...) is even
3. the sum of the coefficients of all odd degree monomials _excluding 1_ (e.g. $$x^3$$, $$x^5$$, $$x^7$$, ...) is even

Examples of valid permutation polynomials modulo a power of 2:

$$
24x^2 + 13x + 15
$$
<br>
$$
13x^4 + 72x^3 + 9x^2 + 85x + 32
$$
<br>
$$
248x^2 + 97x
$$

Examples of invalid permutation polynomials modulo a power of 2:

$$
9x^2 + x
$$
<br>
$$
10x^4 + 3x^2
$$
<br>
$$
19x^4 + 13x^3 + 13x^2 + 13x
$$

More reading about this property can be found in [this paper](https://www.sciencedirect.com/science/article/pii/S107157970090282X?via%3Dihub).

### Inversion

Finding the inverse of a permutation polynomial is useful for some problems. Given a permutation polynomial, $$P$$, and its inverse, $$Q$$, these two polynomials have the property that $$P(Q(x)) = x$$ and $$Q(P(x)) = x$$.

The first paper to demonstrate a method for inverting arbitrary binary permutation polynomials, is [Barthelemy et al.](https://inria.hal.science/hal-01388108/document) in 2016. In it, they use an iterative Newtonian method to converge on a valid inverse using the formula:

$$
g_{i+1} = g_i - g_i' \times (f \circ (g_i - X))
$$

Where $$f(x)$$ is the original polynomial, and $$g(x)$$ is its inverse. This formula is to be iteratively applied until the inverse has been found.

In 2018 Barthelemy released [a second paper](https://hal.science/hal-01981320/document) describing two additional algorithms for finding the inverse of a permutation polynomial, this time using Lagrange interpolation. I am less familiar with these algorithms, and as far as I am aware, they are not strictly better than the above Newtonian approach.

As the first paper describes, inversion has useful properties in the field of obfuscation where a [formula such as $$P(E + Q(K))$$ is used to obfuscate constants](https://openaccess.uoc.edu/bitstream/10609/146182/8/arnaugamezFMDP0622report.pdf#page=14), where $$K$$ is a constant, $$P$$ is a permutation polynomial, $$Q$$ is its inverse, and $$E$$ is an arbitrarily complex expression that evaluates to 0.



<!-- 
https://plzin.github.io/posts/mba
https://dl.acm.org/doi/10.1145/2995306.2995310
https://www.sciencedirect.com/science/article/pii/S107157970090282X?via%3Dihub
https://www.sciencedirect.com/science/article/pii/S107157970090282X?ref=pdf_download&fr=RR-2&rr=821349c6cd2ecfdd
https://link.springer.com/chapter/10.1007/978-3-540-77535-5_5
https://inria.hal.science/hal-01388108/document
https://hal.science/hal-01981320/document
https://en.wikipedia.org/wiki/Vandermonde_matrix

https://arxiv.org/pdf/1008.1012.pdf
-->
