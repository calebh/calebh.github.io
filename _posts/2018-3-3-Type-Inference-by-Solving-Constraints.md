---
layout: post
title: Type Inference by Solving Constraints
---

Type inference is used in functional programming languages to automatically deduce the type of expressions based on how the expression is used. Inference reduces the burden on the programmer who would otherwise have to write all the types manually. Anyone who uses an imperative statically typed programming language knows how quickly explicit type annotations can become burdensome. Type inference is slowly permeating non-functional languages such as C++ and C#, which widens the potential use of inference.

Programming language designers are very interested in how type inference is implemented. Unfortunately the vast majority of the content on type inference is contained in academic publications from the 1990s and early 2000s. The implementation details have not yet made their way to the sphere of general public knowledge, which hinders further adoption by programming language designers. This blog post will present an overview of solving type inference problems by solving a **system of constraints**. The solver will be capable of solving constraints in a Hindley-Milner type system. The algorithm presented here is significantly simpler than [Algorithm W](https://en.wikipedia.org/wiki/Hindley%E2%80%93Milner_type_system#Algorithm_W), which is the classic method of type inference. Most of the resources on the Internet revolve around Algorithm W, which is not ideal.

## Understanding Kinds

Understanding kinds is a critical prerequisite to understand how types are represented by type expressions. Just as expressions have a type signature, a type expression has a *kind signature*. In its most basic form, a kind tells us how we can construct a type. We represent kinds by using asterisks ``*`` and kind functions ``->``. **The asterisk is pronounced as "type".** Here is how we will represent kinds in the constraint solver:

{% gist a421518c10969445ac21ae2b852c2676 %}

So we have a representation of kinds, but what do they actually mean? The easiest way to understand kinds is by looking at a bunch of examples of types and type constructors. Monomorphic types such as ``Int`` and ``Bool`` have kind ``*``. **Type constructors** are handled differently. An example of a type constructor is ``[]`` (list), which has kind ``* -> *``. So list is a type constructor that takes in a type (which we represent with an asterisk), and returns another type. Therefore ``[Int]`` has kind ``*``, since we applied the type ``Int`` to the list type constructor ``[]``, resulting in the type ``[Int]``. Types constructors can also in some situations be partially applied, just like value constructors. Kinds are right associative, so the kind ``*->*->*`` is the same as ``*->(*->*)``

The table below gives the kinds of various type expressions from Haskell:

| Type Expression       | Kind              |
|-----------------------|-------------------|
| ``Int``               | ``*``             |
| ``Bool``              | ``*``             |
| ``Maybe``             | ``*->*``          |
| ``Either``            | ``*->*->*``       |
| ``[]``                | ``*->*``          |
| ``->``                | ``*->*->*``       |
| ``a->b``              | ``*``             |
| ``[Either Int Bool]`` | ``*``             |
| ``Either Int``        | ``*->*``          |
| ``(->) a``            | ``*->*``          |
| ``(,,,)``             | ``*->*->*->*->*`` |
| ``Mu`` where <br/>``newtype Mu f = In { out :: f (Mu f) }`` | ``(* -> *) -> *``|

Note that partial application of type constructors have limitations. For example, the kind of ``[Either Int]`` is nonsensical, and Haskell gives the following error: ``Expecting one more argument to ‘Either Int’. Expected a type, but ‘Either Int’ has kind ‘* -> *’``. The kind of the type constructors that we write are determined by the number of type variables on the left side of the type declaration. For example, the type ``data Foo a b c = ...`` has kind ``*->*->*->*`` since we have three type variables ``a``, ``b``, and ``c``. So kinds fundamentally represent the [arity](https://en.wikipedia.org/wiki/Arity) of type constructors.

## Type Expressions

Programs are typically composed of **expressions** of values, constants, variables, operators, and functions. Analogously, we can have **type expressions** which live at the type level. First we will define type expressions in terms of type variables, type constructors, and type applications:

{% gist 6d37ca6b66831a8d9d7831b560d5b4a0 %}

The definition above also includes the type ``Scheme``, which represents a type with a forall quantifier. Schemes are used when defining parametric polymorphic functions. Note that the constraint solver does not deal with ``Scheme`` in any way. When we want to solve a constraint that involves a polymorphic function, we will first have to *instantiate* a scheme by substituting the quantified variables with *fresh type variables*. This will be discussed in more detail later.

We will now define some functions which will be helpful for creating specific types:

{% gist 8b9f7bebf8a5f0f0e374f09131ec7e6b %}

Defining some string conversion functions will be useful for debugging. These functions may introduce extraneous parenthesis, but they will get the job done for our purposes:

{% gist bd1eb6645db4fe5f9ad779873424acd6 %}

Although the constraint solver will not deal with type schemes directly, it is useful to define some functions that work with them.
* ``tysubst`` is a very important function. It is used both by the constraint solver and the type scheme functions. ``tysubst`` takes in ``theta``, which maps type variables to type expressions. Every type variable in the domain of ``theta`` is replaced with a corresponding type expression. Type variables that are not present in the domain of ``theta`` remain untouched.
* ``freshtyvar`` is used all the time when generating constraints. This generates a globally unique type variable.
* ``instantiate`` converts a type scheme to a type expression by substituting the universally quantified type variables with the list of type expressions in the ``actuals`` parameter.
* ``freevars`` will tell us what type variables exist within a certain type expression
* ``generalize`` does the opposite of ``instantiate`` by looking for free type variables in the type expression and then making them universally quantified. The ``skipTyVar`` parameter tells ``generalize`` which type variables it should ignore. This function is used when generalizing let bindings.

{% gist b3a7bd676667482ccab9b724c8faaef4 %}

## Constraints

We are now ready to define the types which will represent the constraints. The constraint type will be divided into three different value constructors:
* ``Equal`` which tells the solver that the two type expressions must be equal. Equal also includes an ``ErrorMessage``, which is a lazily evaluated string. This ensures that the error messages are not actually created unless they are actually needed. In some of the compilers that I've written, the string includes information obtained from a disk read (to show the programmer where the error was in their source code), so making this lazy is very important for performance reasons. Including an error message here makes type errors that are very understandable.
* ``And`` which conjoins two constraints that must both be satisfied.
* ``Trivial`` is a constraint which is always satisfied.

``conjoinConstraints`` is a helper function which joins a list of constraints together by using the ``And`` value constructor.

{% gist f94131812b1dcd37e83090b13cdbfc5e %}

## Solver

Now for the big finale! Given a system of constraints, we want to get a ``Map<TyVar, TyExpr>`` (aka ``Subst``), which will tell us how we should map the unknown variables to concrete ``TyExpr``s.
* First we define some extension functions to F#'s ``Map`` module, which will be helpful in writing the solver.
* The ``varsubst`` function takes in a ``Subst`` and a ``TyVar``. If the ``TyVar`` is in the domain of the substitution, we apply the substitution, otherwise we leave the ``TyVar`` alone.
* ``compose`` will join two ``Subst``s together into a new map. ``compose`` is more sophisticated than a simple ``Map`` merge. To start off, we union the two domains of the substitutions together. Then for every ``TyVar`` in the new ``domain``, first apply ``theta1`` by running it through ``varsubst``, then pass this result through ``theta2`` via the ``tysubst`` function we defined before.
* ``(|--->)`` will create a new binding in an empty ``Subst``. If the variable being bound ``a`` is exactly the same as ``tau`` there is no need to define a new substitution, so just return ``idSubst``. If ``a`` is in the free variables of ``tau``, then the constraint is **unsolvable**. An example where we can run into this is with the constraint ``a ~ [a]``. In all other cases, we should return a new ``Subst``, where ``a`` is mapped to ``tau``.
* ``consubst`` will update a ``Constraint`` with the results of the ``Subst``.
* The ``solve`` function will solve a system of constraints and return a ``Subst`` as a solution.
  * In the case where the constraint is ``Trivial``, then the system is solvable by the identity substitution.
  * In the case where the constraint is ``And``, we first ``solve`` the left hand side, then apply the resulting substitution to the right hand constraints. Then we solve the new right hand constraints by calling ``solve`` again. The final step is to ``compose`` the results together.
  * In the case where the constraint is ``Equal``, we pattern match on the type pair ``(tau1, tau2)``.
    * In the case where one of the two sides is a type variable, then we try to bind one side to the other. If ``(|--->)`` returns ``None``, then the system cannot be solved, so we should raise a ``TypeError``. Otherwise, we return the substitution given by ``(|--->)``.<br/><br />Notice that if *both* sides are type variables, then the left type variable ends up mapping to the right type variable. From the perspective of the constraint solver, this preference for mapping from left to right doesn't matter much. In the larger picture, keeping this asymmetry in mind when generating constraints is important. For example, we should always prefer to place a user declared constraint on the right hand side so that the right side ends up in the codomain of the ``Subst`` map instead of the domain.
    * In the case where the two sides were both type constructors, then we check if they are equal. If they are the same, then the identity substitution is returned. Otherwise, we raise a ``TypeError``.
    * In the case where the two sides are type applications, then we make a *new* system of constraints by demanding that the left side of the type applications must be equal, and the right hand sides of the type applications must be equal. We then solve the new system of constraints via a recursive call.
    * In all other cases, the system is unsolvable, so we raise a ``TypeError``.

{% gist 87b9f7232b07f474da8e777e8ed3a283 %}

Now we can test our solver by writing some test cases:

{% gist 7bf20f9112dd0b27ad187a0f43350bd1 %}

On lines 1-3, we can see the result of the first call to solve. The constraint we were trying to solve was ``(t1, Number) ~ (Unit, t2)``, and the output was the ``Map`` ``{t1 → Unit, t2 → Number}``, as expected. The second constraint was ``Unit ~ Number``, which wasn't solvable. The solver correctly raised a ``TypeError`` which notified us of the mistake in addition to showing the error message attached to the constraint.

{% gist c08ed85c27c196659185a441ab3af968 %}

## Wrapping Up

Generating constraints is usually a straightforward process, and can be accomplished by using a recursive function. The constraint generation process will be the subject of my next blog post.

The constraint solver presented here is based on Norman Ramsey's book *Programming Languages: Build, Prove, and Compare*. The book is still unpublished at the time of this post's publication.

The code for the full constraint solver is available here under the Unlicense license (which places the code presented here in the public domain): https://gist.github.com/calebh/05300bad85b9a1ab1d4e1cb7e2f0383c

## Related Links

* [Typing Haskell in Haskell](https://gist.github.com/chrisdone/0075a16b32bfd4f62b7b)
* [Let Should not be Generalised](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tldi10-vytiniotis.pdf)
