---
layout: post
title: Gotta Monomorphize Em All
---

As a functional programming language, Juniper supports the primary feature of any functional language - first class functions. Since Juniper targets Arduino compatible systems that have minimal amounts of RAM, we would like to avoid heap allocations. Indeed, if you look at recommendation guides for embedded system programming, you will find that heap allocation shouldn't be used or minimized. This poses a conundrum for language designers that want to support first class functions, since in most language implementations the closures of first class functions are allocated on the heap. The std::function wrapper in C++ does this, which is obviously not ideal.

Another way of looking at first class function closures is through the lens of existential types. If we view a closure as a simple record type, we can pack the closure value into an existential and then pass it explicitly to the function when we invoke it. Therefore a function that takes an $a$ and returns a $b$ and captures some closure record $\delta$ actually has the type $\\{\exists \delta, (\delta, a) \rightarrow b\\}$.

The astute reader will immediately point out that it's not possible to store existentials on the stack. This turns out to not be the case thanks to a clever analysis that will convert universally quantified values to intersection types and existentially quantified types to union types. Additionally we will assume that our language does not allow mutation, so it's safe to copy values into a closure, removing the need for a heap location to store mutable state. Juniper does allow local mutation, however variables captured by a closure are instead saved as a 'snapshot' of the variable state at which the first class function is declared. These captured variables are then immutable within the function.

The encoding for the universally quantification and existential quantification is nice since many language features can be cast in terms of these types. Local polymorphic functions, abstract datatypes, and closures are the obvious examples. I think that universal quantification is easier to understand, so let's take a look at how we can convert them to intersection types:

Suppose we have the polymorphic identity function $\forall a . a \rightarrow a$. If we can track where the identity function is used, and check what types we want the identity function to be monomorphized with, we can convert it directly to an intersection type. Let's say we track the locations where $id$ is used, and deduce that we use it with int and bool only. Then instead of $\forall a . a \rightarrow a$ we can type it as $(int \rightarrow int) \& (bool \rightarrow bool)$. Passing around this intersection type value is easy enough, for example we could represent it as some sort of tuple, and project out the proper value as needed.

Existentials are a similar story, except that they concert to union types. Let’s say we have a conditional expression where the two branches return different functions that are packed with different closures:

```
if cond then (x) => x + y else (x) => x + y + z
```

The return type of the if expression is $\\{\exists \delta, (\delta, int) \rightarrow int\\}$. In this case our use of existential is hiding the type of the closure record and there is an implicit use of existential packing to save the captured variables in a record. Once we convert the existential to a union type, the type of the if expression becomes $( \\{ y : int \\}, int) \rightarrow int \| ( \\{y : int, z : int \\}, int) \rightarrow int$. This type will flow to the points where the program unpacks the existential, at which the program inspects the contents of the union. Implementing existentials and universal quantification in this way introduces additional code duplication, with implementations being generated for each monomorphized type. This is no different than the traditional top level monomorphization, except that it can be done for non-top level values.

Another way to look at the situation is that intersection types and union types are finite versions of universally quantified types and existentials. The flow analysis could get pretty complicated, we have to track where the quantifiers are introduced and see where they are used. Polymorphic recursion in our use is outlawed, since it could force our intersection and union types to be infinite, which we do not want. By utilizing this encoding scheme, it should be possible to fully monomorphize System F and execute without the heap, except in the case of polymorphic recursion.

## Further reading:

- [A calculus with polymorphic and polyvariant flow types. Wells et al. 2022.](https://dl.acm.org/doi/10.1017/S0956796801004245)
- [Faithful translations between polyvariant flows and polymorphic types. Amtoft and Turbak. 2000.](https://link.springer.com/chapter/10.1007/3-540-46425-5_2)
- [Tracking Data-Flow with Open Closure Types. Scherer and Hoffmann. 2013.](https://link.springer.com/chapter/10.1007/978-3-642-45221-5_47)
