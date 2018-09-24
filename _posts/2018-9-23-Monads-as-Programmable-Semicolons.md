# Monads as Programmable Semicolons

Some other monad tutorials assume that you have the prerequisite understanding of basic Haskell and some simpler type classes (``Functor`` and ``Monoid``). I will skip these in favor of jumping straight to monads. Understanding basic static typing like those found in C#, Java, or C++ is the only prerequisite for this guide.

> A monad is just a monoid in the category of endofunctors, what's the problem?

## Type Classes

Haskell has a feature called type classes which allows you to overload functions. The definition of a type class gives a list of functions with their type signatures and names. When you write an instance of a type class, you must give implementations for the functions defined in the type class. All the functions that you define in the instance must match the type signatures given in the type class definition.

## Kinds

Understand kinds is another critical prerequisite to understanding monads. Have you ever wondered what the type of a type is? Kinds are a simple construction that serve as an answer to this question. Just as expressions have a type signature, a type expression has a kind signature. In its most basic form, a kind tells us how we can construct a type. We represent kinds by using asterisks ``*`` and kind functions ``->``. The asterisk is pronounced as "type".

The easiest way to understand kinds is by looking at a bunch of examples of types and type constructors. Monomorphic types such as Int and Bool have kind ``*``. Type constructors are handled differently. An example of a type constructor is ``[]`` (list), which has kind ``* -> *``. So list is a type constructor that takes in a type (which we represent with an asterisk), and returns another type. Therefore ``[Int]`` has kind ``*``, since we applied the type ``Int`` to the list type constructor ``[]``, resulting in the type ``[Int]``. Types constructors can also in some situations be partially applied, just like value constructors. Kinds are right associative, so the kind ``* -> * -> *`` is the same as ``* -> ( * -> * )``.

Here is a table of some types in Haskell with their kinds:
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

## Monads

Now we are ready to look at the type class definition for monad:

{% gist 6626ab34cdfb61378f2bc78a526b84e0 %}

On the first line, we give the name of the type class: Monad. This type class will be parameterized by a parameter named ``m``. Based on how m is used in the function signatures, we deduce that m must have kind ``* -> *``. This means that when we define an instance of the Monad type class, we tell what ``m`` should be. And m can be anything, as long as it has the kind ``* -> *``! In fact if we look at the table above, we can see that ``Maybe`` and the ``[]`` (list) type constructors have the required kind. If you've read any monad tutorials, these are often used as simple examples of monad instances.

The most important function in the monad type class is ``>>=``, so let's focus on that. ``>>=`` is an infix function with two parameters, one of type ``m a`` and the other of type ``a -> m b``. The return type of ``>>=`` is ``m b``. So if we were to define a monad instance for ``m=Maybe``, the type of that particular overloaded version of ``>>=`` has to be ``Maybe a -> (a -> Maybe b) -> Maybe b``. What should the definition of ``>>=`` be? Well it can be anything as long as the type signatures match up!

Take a look at the second parameter that has type ``a -> m b``. You can think of this as a sort of *callback* or *continuation* function. Typically the ``>>=`` function takes the first parameter (which has type ``m a``), does some processing to unwrap the value, passes this value to the callback function and then takes this result and does some more processing before returning the result (which must have type ``m b``). The ``>>=`` can call the callback function as many times as it wants (or maybe not at all). It could do anything, as long as the type signature matches up. There's one more detail that I've so far been ignoring: the overloaded functions need to follow some rules beyond the type signature constraint. These are called the monad laws, and you can find out more about them elsewhere by searching for "monad laws".

Why even bother with the Monad type class at all? It turns out that programmers have discovered many different programming patterns that seem to match up with these signatures. In fact, this particular pattern has become observed to be so common and useful that the authors of Haskell decided to provide some useful built in language operations to make them easier to use (do notation in Haskell). The ``do`` notation in Haskell is used for chaining uses of ``>>=`` together. You can think of the ``>>=`` as a sort of programmable semicolon, where the overloaded bind operator is parameterized by a specific continuation function, but the overall program flow is dictated by the specific overloaded version of ``>>=``.

## Metaprogramming & Abstraction

Monads and higher order functions blur the lines between programming and metaprogramming. When we define a new monad, we are essentially creating a miniature domain specific language (DSL) that handles a sequential computation flow pattern for a specific container type (one that has kind ``* -> *``). Another great example of this is F#'s [computation expression](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/computation-expressions) feature. A computation expression can explicitly override language operations, which means that they fall more on the metaprogramming side.

Whenever we write new programs, libraries, or frameworks, a line in the sand is drawn at the abstraction boundary. Pushing to higher levels of abstraction tends to give greater re-usibility potential at the expense of paying the cognitive overhead to understand the abstraction in the first place. As a mediums of communication of ideas, the programming languages we use shapes how programs are written.