Here is my version of "yet another monad tutorial". In this tutorial I will not cover specific examples of ``Monad`` implementations in detail since I think that this distracts from the goal of understanding the overall abstraction. I will also not cover some of the simpler type classes such as ``Functor`` and ``Monoid`` in detail, since these aren't really that important for understanding ``Monad``.

One more thing to clear up - **monads are used for more things than error handling**. This is a big mistake that I see all the time in monad tutorials. The authors will lead off by trying to handwave through the ``Maybe`` monad while completely missing that the point that they are actually much more general!

> 1. A monad may not injure a human being or, through inaction, allow a human being to come to harm.
> 2. A monad must obey orders given it by human beings except where such orders would conflict with the First Law.
> 3. A monad must protect its own existence as long as such protection does not conflict with the First or Second Law. 
> -Isaac Asimov

## Function Syntax

In most mainstream languages, the arguments to a function are given in a comma separated list while enclosed with parenthesis. In Haskell, function arguments are given in a space separated list. This syntax difference is not just for asthetics - it has a purpose. When you define a new function in Haskell, the function can automatically be partially applied (the technical term is [currying](https://en.wikipedia.org/wiki/Currying)). Let's take a look at an example:

{% gist c40f2c1e73cf1743fca14e221e02a5ee %}

Here I have defined a new Haskell function called ``myconcat``, which takes two lists as arguments. When I want to invoke the ``myconcat`` function, I can pass it a single parameter and get a new function as a result. Since strings in Haskell are just lists of characters, I can partially apply it with the "Hello " string to get a new function that will prepend ``"Hello "`` to the front of a character list.

The partial application is also reflected in the types of the ``myconcat`` and ``greet`` functions. The type of ``myconcat`` is ``[a] -> [a] -> [a]`` and the type of ``greet`` is ``[Char] -> [Char]``. Function types in Haskell are written using an ``->``, where the argument type is on the left side of the ``->``, and the return type is on the right side. The arrow type is right associative, so the type of ``myconcat`` can also be written as ``[a] -> ([a] -> [a])``. So if I pass a single argument to ``myconcat``, I get back a new function of type ``[a] -> [a]``. This is what happened when I defined the greet function - since I partially applied the function I get a new function back as a result!

Now I can use my greet function in action in the Haskell REPL (GHCI):

```
greet "Sally"
=> "Hello Sally"
greet "Joe"
=> "Hello Joe"
```

Type parameters in most languages are given in a comma separated list enclosed in angle brackets ``<>``. In Haskell the arguments are separated by spaces, just like in functions. For example, a binary tree type containing booleans in a language like C# might be written as ``BinaryTree<bool>``. In Haskell this type would be written like ``BinaryTree Bool``. Types can also be partially applied for types parameterized by multiple arguments.

## Kinds

Understand kinds is a critical prerequisite to understanding monads. Have you ever wondered what the type of a type is? Kinds are a simple construction that serves as an answer to this question. Just as expressions have a type signature, a type expression has a kind signature. In its most basic form, a kind tells us how we can construct a type. We represent kinds by using asterisks ``*`` and kind functions ``->``. The asterisk is pronounced as "type".

The easiest way to understand kinds is by looking at a bunch of examples of types and type constructors. Monomorphic types such as Int and Bool have kind ``*``. Type constructors are handled differently. An example of a type constructor is ``[]`` (list), which has kind ``* -> *``. So list is a type constructor that takes in a type (which we represent with an asterisk), and returns another type. Therefore ``[Int]`` has kind ``*``, since we applied the type ``Int`` to the list type constructor ``[]``, resulting in the type ``[Int]``. Types constructors can also in some situations be partially applied, just like value constructors. Kinds are right associative, so the kind ``* -> * -> *`` is the same as ``* -> ( * -> * )``.

Here is a table of some types in Haskell with their kinds:

| Type Expression       | Kind              | Description                                                                       |
|-----------------------|-------------------|-----------------------------------------------------------------------------------|
| ``Int``               | ``*``             | Integer                                                                           |
| ``Bool``              | ``*``             | Boolean                                                                           |
| ``Maybe``             | ``*->*``          | Optional type                                                                     |
| ``Either``            | ``*->*->*``       | A value that can be one of two possible types                                     |
| ``[]``                | ``*->*``          | List                                                                              |
| ``->``                | ``*->*->*``       | Function                                                                          |
| ``a->b``              | ``*``             | A function that takes in a value of type ``a`` and returns a value of type ``b `` |
| ``[Either Int Bool]`` | ``*``             | A list of integers or booleans                                                    |
| ``Either Int``        | ``*->*``          | A partially applied ``Either`` type                                               |
| ``(->) a``            | ``*->*``          | A partially applied function type                                                 |
| ``(,,,)``             | ``*->*->*->*->*`` | A four element tuple                                                              |
| ``Mu`` where <br/>``newtype Mu f = In { out :: f (Mu f) }`` | ``(* -> *) -> *``| Least fixpoint of functors                   |

## Type Classes

Type classes are a feature of Haskell that allow you to overload functions. Each type class declaration is parameterized by one or more arguments. The arguments are types - they determine which type you are overloading. Let's take a look at an example:

{% gist 882c702a92b0192d7e9a0f5a557959ed %}

{% gist 2d0c2a26a73980e775eb6ed9cd4ccd80 %}

In the example above I have defined the ``Mappable`` (aka [Functor](http://hackage.haskell.org/package/base-4.12.0.0/docs/Data-Functor.html)) typeclass, which contains a single function that we can overload. The ``map`` function should take in elements in the container and apply the mapping function to them. For example, we could overload the ``map`` function so that it can handle both trees and lists. Notice that the ``m`` parameter in the typeclass definition has kind ``* -> *`` (ie, it is a type constructor for a container type that consumes a single type argument).

## Monads

Now we are ready to look at the type class definition for monad:

{% gist 6626ab34cdfb61378f2bc78a526b84e0 %}

{% gist 8a7458b66458b318416f7c2f5974fc92 %}

On the first line, we give the name of the type class: ``Monad``. This type class will be parameterized by a parameter named ``m``. Based on how m is used in the function signatures, we deduce that ``m`` must have kind ``* -> *``. This means that when we define an instance of the ``Monad`` type class, we tell what ``m`` should be. And ``m`` can be anything, as long as it has the kind ``* -> *``! In fact if we look at the table above, we can see that ``Maybe`` and the ``[]`` (list) type constructors have the required kind. If you've read any monad tutorials, these are often used as simple examples of monad instances. Here's an example of instances of the ``Monad`` type class for ``Maybe`` and ``List``. Note that the equivalent for ``Maybe`` in C# is ``Nullable``, so I've used that for the C#-like syntax example.

{% gist 0c882099ba72afd8cf9068ec82c42111 %}

{% gist bd70279b3bd2a691018443fa50930227 %}

I will not dwell on the specific instance implementations here since it's mostly irrelevant for understanding what a monad *is*. However the specific instances are good for illustrating the *motivation* of why you would want to use a monad in the first place. The Learn You a Haskell book covers these instances in more detail here: http://learnyouahaskell.com/a-fistful-of-monads

Notice in the above implementations that the ``>>`` function can always be implemented by using ``>>=``. The ``return`` function is also important, but it is not used as often as ``>>=`` in practice. This means that ``>>=`` is the most important function, so let's focus on that.

``>>=`` is an infix function with two parameters, one of type ``m a`` and the other of type ``a -> m b``. The return type of ``>>=`` is ``m b``. So if we were to define a monad instance for ``m=Maybe``, the type of that particular overloaded version of ``>>=`` has to be ``Maybe a -> (a -> Maybe b) -> Maybe b``. What should the definition of ``>>=`` be? Well it can be anything as long as the type signatures match up! Also remember that ``>>=`` is an overloaded function (since it was declared in a type class), so the overloaded implementation that is actually used depends on the types of the input values.

Take a look at the second parameter that has type ``a -> m b``. You can think of this as a sort of *callback* or *continuation* function. Typically the ``>>=`` function takes the first parameter (which has type ``m a``), does some processing to unwrap the value, passes this value to the callback function and then takes this result and does some more processing before returning the result (which must have type ``m b``). The ``>>=`` can call the callback function as many times as it wants (or maybe not at all). It could do anything, as long as the type signature matches up. There's one more detail that I've so far been ignoring: the overloaded functions need to follow some rules beyond the type signature constraint. These are called the monad laws, and you can read more about them here: https://wiki.haskell.org/Monad_laws

Why even bother with the ``Monad`` type class at all? It turns out that programmers have discovered many different programming patterns that seem to match up with these signatures. In fact, this particular pattern has become observed to be so common and useful that the authors of Haskell decided to provide some built in language operations to make them easier to use.

The ``do`` notation in Haskell is used for chaining invocations of ``>>=`` together. When a binding operation (written with a left arrow ``<-``) is used in a ``do`` block, the callback is automatically generated, and the return value of a second, nested ``>>=`` operation is used as the return value of this new callback. The easiest way to see this in action is by looking at a ``do`` block and the equivalent de-sugared code:

{% gist 7bc2280c6ce123cae73625c67504c9c9 %}

Notice that ``expr2`` has access to ``x1`` due to the scoping of the lambdas, and the return value of the nested ``>>=`` call is used as the return value of the callback.

Another way that these ``Monad`` functions can be sequenced is by using the ``>>`` operation. To syntax de-sugaring in this case is a little bit simpler:

{% gist 363496e04a1e68d53e9d83295f0a87af %}

For more details on how this syntax sugar works, see this helpful guide: https://en.wikibooks.org/wiki/Haskell/do_notation

Here is what the Haskell docs say about ``>>``:
> Sequentially compose two actions, discarding any value produced by the first, like sequencing operators (such as the semicolon) in imperative languages.

## Conclusion

The overloaded ``>>=`` (bind) operator is pararmeterized by a specific callback/continuation function, but the overall program flow is dictated by the specific overloaded version of ``>>=``. In practice, the callback function is automatically generated by the ``do`` notation syntax, and the return value is given by a second, nested invocation to ``>>=``. Monads and higher order functions blur the lines between programming and metaprogramming. When we define a new monad, we are essentially creating a miniature domain specific language (DSL) that handles a sequential computation flow pattern for a specific container type (one that has kind ``* -> *``). Another great example of this is F#'s [computation expression](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions) feature. A computation expression can explicitly override language operations, which means that they fall more on the metaprogramming side.