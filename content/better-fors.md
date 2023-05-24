---
layout: sip
permalink: /sips/:title.html
stage: design
status: waiting-for-design
title: SIP-55 - Better fors
---

**By: Kacper Korban (VirtusLab)**

## History

| Date          | Version            |
|---------------|--------------------|
| May 24th 2023 | Initial Draft      |

## Summary

`for`-comprehensions in Scala 3 improved their usability in comparison to Scala 2, but there are still some pain points relating both usability of `for`-comprehensions and simplicity of their desugaring.

This SIP tries to address some of those problems, by changing the specification of `for`-comprehensions.

## Motivation

There are some clear pain points related to Scala'3 `for`-comprehensions and those can be divided into two categories:

1. User-facing and code simplicity problems

    Specifically, for the following example written in a Haskell-style do-comprehension

    ```haskell
    do
      a = largeExpr(arg)
      b <- doSth(a)
      combineM(a, b)
    ```
    in Scala we would have to write

    ```scala
    val a = largeExpr(b)
    for
      b <- doSth(a)
      x <- combineM(a, b)
      yield b
    ```

    This complicates the code even in this simple example.
2. The simplicity of desugared code
    
    The second pain point is that the desugared code of `for`-comprehensions can often be surprisingly complicated.

    e.g.
    ```scala
    for
      a <- doSth(arg)
      b = a
    yield a + b
    ```

    Intuition would suggest for the desugared code will be of the form

    ```scala
    doSth(arg).map { a =>
      val b = 2
      a + b
    }
    ```

    But because of the possibility of an `if` guard being immediately after the pure binding, the desugared code is of the form

    ```scala
    doSth(arg).map { a =>
      val b = 2
      (a, b)
    }.map { case (a, b) =>
      a + b
    }
    ```

    Those unnecessary assignments and additional function calls are adding unnecessary runtime overhead but also can block other optimizations from being performed.

## Proposed solution

This SIP suggests the following changes to `for` comprehensions:

1. Allow `for` comprehensions to start with "pure" assignments
  
    e.g.
    ```scala
    for
      a = 1
      b <- Some(2)
      c <- doSth(a)
    yield b + c
    ```
2. Introduce new syntax for monadic expressions without bindings. An `exec` keyword.
    
    e.g.
    ```scala
    for
      exec doSth1(1)
      b <- doSth2(2)
      yield b
    ```

    which will be equivalent to

    ```scala
    for
      _ <- doSth1(1)
      b <- doSth2(2)
      yield b
    ```

3. Allow `for` comprehensions to end without trailing `yield` (as long as they are monadic expressions)
  
    e.g.
    ```scala
    for
      a <- doSth1(1)
      b <- doSth2(2)
      exec combineM(a, b) // M[_]
    ```

    will be equivalent to

    ```scala
    for
      a <- doSth1(1)
      b <- doSth2(2)
      x <- combineM(a, b) // M[_]
      yield x
    ```
4. Simpler conditional desugaring of pure bindings. i.e. whenever a series of pure binding is not immediately followed by an `if`, use a simpler way of desugaring.

    e.g. 
    ```scala
    for
      a <- doSth(arg)
      b = a
    yield a + b
    ```

    will be desugared to

    ```scala
    doSth(arg).map { a =>
      val b = 2
      a + b
    }
    ```

    but

    ```scala
    for
      a <- doSth(arg)
      b = a
      if b > 1
    yield a + b
    ```

    will be desugared to

    ```scala
    Some(1).map { a =>
      val b = 2
      (a, b)
    }.withFilter { case (a, b) =>
      b > 1
    }.map { case (a, b) =>
      a + b
    }
    ```

5. Avoiding redundant `map` calls if the yielded value is the same as the last bound value.

    e.g.
    ```scala
    for
      a <- List(1, 2, 3)
    yield a
    ```

    will just be desugared to

    ```scala
    List(1, 2, 3)
    ```

<!-- ### High-level overview

A high-level overview of the proposed changes, and how they allow to better solve the running examples. This section should be example-heavy, and not dive into corner cases.

Example:

~~~ scala
// This is an @main method
@main def foo(x: Int): Unit =
  println(x)
~~~ -->

### Detailed description

#### Ad 1. Allow `for` comprehensions to start with "pure" assignments

Allowing `for` comprehensions to start with "pure" aliases is a straightforward change.

When desugaring is concerned, a for comprehension starting with "pure" aliases will generate a block with those aliases as `val` declarations and the rest of the desugared `for` as an expression.

The appropriate desugaring rule looks the following:

```scala
for (P_1 = E_1; ... P_N = E_N; ...)
  ==>
{
  val x_2 @ P_2 = E_2
  ...
  val x_N @ P_N = E_N
  for (...)
}
```

e.g.

```scala
for
  a = 1
  b <- Some(2)
  c <- doSth(a)
yield b + c
```

will desugar to

```scala
{
  val a = 1
  for
    b <- Some(2)
    c <- doSth(a)
  yield b + c
}
```

#### Ad 2. Introduce new syntax for monadic expressions without bindings. An `exec` keyword.

This change is derived from two different problems:
- Having to write `_ <- ` each time to execute a monadic expression, seems to be a pet peeve shared by a good part of the Scala FP community. Adding a keyword to use instead will make the code easier to read and write. (`exec` is only a suggestion and though it has the same number of characters as `_ <-`, all those characters are alphanumeric.)
- Implementing feature `3.` would require having a construct in `for` comprehensions that has the meaning of returning a value, instead of disposing of it or assigning it.

The relevant desugaring rules, look as follows:

```scala
*  2.
*
*      for (E) yield B  ==> E
*
*   if B is Empty
*
*  3.
* 
*      for (E; ...) yield B  ==>  for(_ <- E; ...) yield B
*
```

e.g.

```scala
for
  exec List(1, 2)
  x <- List(3, 4)
yield x
```

will desugar to:

```scala
for
  _ <- List(1, 2)
  x <- List(3, 4)
yield x
```

Discussion on the keyword choice:

Ideally, there would be no keyword needed and a code like the following should be allowed:

```scala
for
  List(1, 2)
  x <- List(3, 4)
yield x
```

The problem with this approach is that it introduces a lot of complexity to the parser. This in turn also requires similar changes in other tools that have to deal with the Scala source code e.g. parsers, IDEs. That is why introducing a keyword for monadic expressions seems like a good middle ground. The keyword choice is also just a suggestion. Other choices might be more appealing e.g. `do` (though it may introduce ambiguities with `for ... do ???`).

#### Ad 3. Allow `for` comprehensions to end without trailing `yield` (as long as they are monadic expressions)

It is often the case that one wants to return the result of the last monadic expression from the `for` comprehension. Currently, this can only be done by first assigning it to a value and then returning that value in a `yield`. This change will allow `for` comprehensions to omit the `yield` part, as long as the body of the `for` ends with a monadic expression introduced in feature `2.`.

Related desugaring rule:

```scala
*  2.
*
*      for (E) yield B  ==> E
*
*   if B is Empty
```

e.g.

```scala
for
  x <- None
  exec Some(1)
```

will desugar to:

```scala
None.flatMap { x =>
  Some(1)
}
```

#### Ad 4. Simpler conditional desugaring of pure bindings. i.e. whenever a series of pure binding is not immediately followed by an `if`, use a simpler way of desugaring.

Currently, for consistency, all "pure" aliases are desugared as if they are followed by an if condition. Which makes the desugaring more complicated than expected.

e.g.

The following code:

```scala
for
  a <- doSth(arg)
  b = a
yield a + b
```

will be desugared to:

```scala
Some(1).map { a =>
  val b = 2
  (a, b)
}.map { case (a, b) =>
  a + b
}
```

The proposed change is to introduce a simpler desugaring for common cases, when aliases aren't followed by a guard, and keep the old desugaring method for the other cases.

Related desugaring rules:

```scala
*  6. For any N:
*
*      for (P <- G; P_1 = E_1; ... P_N = E_N; if E; ...)
*        ==>
*      for (TupleN(P, P_1, ... P_N) <-
*        for (x @ P <- G) yield {
*          val x_1 @ P_1 = E_2
*          ...
*          val x_N @ P_N = E_N
*          TupleN(x, x_1, ..., x_N)
*        }; if E; ...)
*
*  7. For any N:
*
*      for (P <- G; P_1 = E_1; ... P_N = E_N; ...)
*        ==>
*      G.flatMap (P => for (P_1 = E_1; ... P_N = E_N; ...))
*
*  8.
*
*      for (P_1 = E_1; ... P_N = E_N; ...)
*        ==>
*      {
*        val x_2 @ P_2 = E_2
*        ...
*        val x_N @ P_N = E_N
*        for (...)
*      }
* 
*      if the aliases are not followed by a guard, otherwise an error.
```

e.g.

```scala
for
  a <- doSth(arg)
  b = a
yield a + b
```

will be desugared to

```scala
doSth(arg).map { a =>
  val b = 2
  a + b
}
```

but

```scala
for
  a <- doSth(arg)
  b = a
  if b > 1
yield a + b
```

will be desugared to

```scala
Some(1).map { a =>
  val b = 2
  (a, b)
}.withFilter { case (a, b) =>
  b > 1
}.map { case (a, b) =>
  a + b
}
```

#### 5. Avoiding redundant `map` calls if the yielded value is the same as the last bound value.

This change is strictly an optimization. This allows for the compiler to get rid of the final `map` call, if the yielded value is the same as the last bound pattern. The pattern can either be a single variable binding or a tuple.

e.g.
```scala
for
  a <- List(1, 2, 3)
yield a
```

will just be desugared to

```scala
List(1, 2, 3)
```

### Compatibility

This change is binary and TASTY compatible, since for-comprehensions are desugared in Typer. So both class and TASTY files only ever see the desugared versions of programs.

This change is also forward source compatible, it is obviously not backward compatible, since it is a new feature.

### Other concerns

<!-- TODO(kπ) -->

<!-- If you think of anything else that is worth discussing about the proposal, this is where it should go. Examples include interoperability concerns, cross-platform concerns, implementation challenges. -->

### Open questions

<!-- TODO(kπ) -->

<!-- If some design aspects are not settled yet, this section can present the open questions, with possible alternatives. By the time the proposal is accepted, all the open questions will have to be resolved. -->

<!-- ## Alternatives

This section should present alternative proposals that were considered. It should evaluate the pros and cons of each alternative, and contrast them to the main proposal above.

Having alternatives is not a strict requirement for a proposal, but having at least one with carefully exposed pros and cons gives much more weight to the proposal as a whole. -->

## Related work

1. Scala contributors discussion thread (pre-SIP): https://contributors.scala-lang.org/t/pre-sip-improve-for-comprehensions-functionality/3509/51
2. Github issue discussion about for desugaring: https://github.com/lampepfl/dotty/issues/2573
3. Implementation of one of the simplifications: https://github.com/lampepfl/dotty/pull/16703
4. 

## FAQ

