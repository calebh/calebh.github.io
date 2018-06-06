---
layout: post
title: Continuous Approximations of Logical Functions
---

Logical functions are critical parts of almost every computer program. However, optimization problems are often solved over the set of real numbers, which does not fit nicely into this rigid binary logic. In many cases, we'd like to take advantage of applying logical operations, which includes their use in things like deep neural networks. How to describe these discrete functions in terms of a continuous function seems challenging. In this post I will show how we can apply the rules of logic to arrive at concise functions.

We begin by making the restriction that the input to the continuous functions must fall in the interval [0, 1]. Real numbers outside of this interval can be squashed into the range [0, 1] by a sigmoid function:

```
sigmoid(x) = 1 / (1+e^(-x))
```

The two easiest logical operations to convert to continuous form are AND and NOT. Writing AND in terms of a simple multiplication keeps many of the same properties as the discrete version. When both arguments to AND are 1, the result is 1. In any other cases the input remains small, which is what we desire. This effect is maintained even when there are multiple arguments to AND. The NOT function is even simpler: it is simply 1-x.

```
and(x, y) = x * y
and(a, b, ..., z) = a * b * ... * z
not(x) = 1 - x
```

Combining the definitions for AND and NOT gives us the continuous versions of the NAND gate:

```
nand(x, y) = 1 - x * y
```

One important property of the NAND gate is its [universality](https://en.wikipedia.org/wiki/NAND_logic). This means that any logical system can be written in terms of NAND. This means that we can easily derive the other logic gates just from the definition of AND and NOT! Here they are:

```
or(x, y) = 1 - (1 - a^2) * (1 - b^2)
nor(x, y) = (1 - a^2) * (1 - b^2)
xor(x, y) = 1 - (1 - a + a^2 * b) * (1 - b + a * b^2)
xnor(x, y) = (1 - a + a^2 * b) * (1 - b + a * b^2)
```

Due to the universality, any logical function you can think of can be written in terms of NAND. As a bonus, here are a few more helpful functions. ``many_equal`` determines whether or not all of its inputs are equal (a generalization of the XNOR gate), and ``which`` can be used in replacement of an ``if`` statement.

```
many_equal(a, b, ..., z) = 1 - (1 - (1-a)^2 * (1-b)^2 * ... * (1-z)^2) * (1 - a^2 * b^2 * ... * z^2)
which(cond, true_branch, false_branch) = cond * true_branch + (1 - cond) * false_branch
```

<pre>
test
</pre>