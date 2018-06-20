# Recursion Schemes Cookbook

Welcome to the recursion schemes cookbook! This document is intended as a complement to [matryohska](https://github.com/slamdata/matryoshka)'s documentation. Its goal is to help you come up with strategies to solve real-life problems using matryoshka (and recursion schemes in general).

## Ingredients

Every recipe usually starts with a list of ingredients. In a recursion schemes cookbook though, we only use a very small wariety of ingredients. Actually we only have three of them and some recipes use only two. Our three ingredients are *pattern-functors*, *f-algebras* and *fixpoint types*.

### Pattern-functors

This is the main ingredient for each recipe. It will determine the "shape" of the computation. For example, if we want to model a binary tree traversal, we would use the following functor as our pattern:

``` scala
sealed trait TreeF[A]
final case class Node[A](left: A, right: A) extends TreeF[A]
final case class Leaf[A]()                  extends TreeF[A]
```

It is not always the case that the notation of a functor (here a simple ADT) resembles the shape of the resulting computation so closely. For instance, you might be surprised that the following functor allows to represent computations over an infinite stream of `Foo`s:

``` scala
type Stream[A] = (Foo, A)
```

Every recipe will start by picking the right pattern-functor. Matryoshka provides a bunch of general-purpose patterns in the `matryoshka.patterns` package, but sometimes you'll have to build a custom pattern to meet your needs (most of the time using and ADT like `TreeF` above). In such cases you should always try to avoid polluting your ADT with unnecessary information. In other words, try not adding fields that do not participate to the recursive *structure* (or shape) of your pattern. As an illustration you can notice that we didn't define any `label` field in `TreeF`.

### F-Algebras

As pattern-functors determine the shape of the computation we build, f-algebras determine what operation we perform at each step of the computation. Given a functor `F` and a *carrier* `A`, 
* An `Algebra[F, A]` is simply a function `F[A] => A`
* A `Coalgebra[F, A]` is simply a function `A => F[A]`

From now on we'll use the term f-algebra to talk about algebras and coalgebras indistinctively, otherwise we'll specifically say "algebra" or "coalgebra" when the distinction is necessary.

Algebras are always applied on the pattern "bottom-up". In our `TreeF` example that means that an algebra will first be applied a leaf, the result of this application will be "pushed" into that leaf's parent, then the algebra will be applied to the parent, and so on up to the root node. 

Dually, coalgebras are always applied "top-down". We start from a seed value of type `A`, the coalgebra produces the root of the tree and then is recursively applied to the root to produce its children, and so on until the leaves are reached. 

It's important to keep in mind that f-algebras can only operate on a single node at a time. In other words, from the body of an f-algebra it is not possible to know where we are in our structure or to peek at another part of it.

Nevertheless, most non-trivial problems will require such capabilities. Fortunately, we can emulate these capabilities using "embellished" — or, to stick with our cooking metaphor, "flavored" — f-algebras.



### Fixpoint Types

Pattern-functors are just regular functors that are used recursively (ie applied to themselves). In a statically-typed language this can be problematic when it comes to writing functions that take or return arbitrary instances of a given pattern.

Consider the following values:

``` scala
val leaf: TreeF[Nothing]       = Leaf()
val node: TreeF[TreeF[Nothing] = Node(Leaf(), Leaf())
```

Both have a different static type, so without any helper, there's no way to write a function that would accept both as an argument (unless we dirty our hands by typing `Any` or `AnyRef`).

Fixpoint types are exactly that helper we need. A fixpoint type is a type of the shape `T[_[_]]`, ie a type constructor that takes a type constructor as its parameter. If we're able to *embed* every "node" of a structure drawn from a pattern `F` into `T`, then we can use the type `T[F]` to represent arbitrary structure based on `F`:

``` scala
val leaf: T[TreeF] = Leaf().embed
val node: T[TreeF] = Node(leaf, leaf).embed

```

In addition, we need a way to *project* a `T[F]` onto a `F[T[F]]` to regain access to our functor and apply algebras.

Matryoshka provides several fixpoint types that have at least one of these two capabilities (*project* or *embed*) that differ only by their run-time characteristics (stack-safety in particular).

Picking the right fixpoint type isn't crucial in coming up with a correct solution to a given problem, although it *is* crucial to use a fixpoint suited to the characteristics of your run-time inputs). It is therefore a good practice to abstract over the fixpoint type and delegate the choice as much as possible.

The `Recursive`, `Corecursive` and `Birecursive` typeclasses capture the two capabilities of fixpoint types: 
* An instance of `Recursive.Aux[T, F]` has a method `project(t: T): F[T]`
* An instance of `Corecursive.Aux[T, F]` has a method `embed(f: F[T]): T`
* An instance of `Birecursive[T, F]` has both

So instead of writing this overly specific signature:

``` scala
def foo(tree: Fix[TreeF]): Fix[TreeF] = ???
```

We will always write 

``` scala
def foo[T](tree: T)(implicit T: Recursive.Aux[T, TreeF]): T = ???
```

Choosing between `Recursive`, `Corecursive` or `Birecursive` depending on whether we need to embed, project or both within the body of the function.

From now on we'll say "recursive `T` on `F`" (resp. corecursive or birecursive) to refer to a type that has an instance of `Recursive.Aux[T, F]`, sometimes omitting "on `F`" when it's unambiguous.


## Cooking Method

And finally to complete our recipe we need to decide how we combine these ingredients together. That is, choosing the right scheme for the job.

There are three families of schemes: folds, unfolds and refolds.
* folds take an algebra on `F` with carrier `A` and produce a function from `T` to `A` (for any recursive `T`)
* unfolds take a coalgebra on `F` with carrier `A` and produce a function from `A` to `T`
* refolds take an algebra on `F` with carrier `B` and a coalgebra on the same `F` with carrier `A` and produce a function from `A` to `B` (notice that no fixpoint type is needed here).

In each of these families, schemes differ only by the "flavor" of the f-algebra(s) they take as parameter. That means that you can either choose your f-algebra(s) first and deduce the scheme you need, or choose a scheme first and it will tell you the f-algebra(s) you need. Whichever you find the easiest.

A thing that's worth keeping in mind though is that all schemes are internally expressed in terms of a refold (there's no other way actually). Folds are implemented as refolds with a no-op coalgebra (and vice versa with unfolds). 

If you want to minimize the number of traversals needed by your solution, you should always try to express it in terms of a minimal numbed of successive refolds. For example, if you find yourself chaining an `ana` and then a `cata` on the same structure, you should definitely use a `hylo` instead.

## Reading this book

We can see recursion schemes as a (quite unusual) way to build functions by using a recursive data structure as a blueprint of the computation (a... pattern). Understanding how the three ingredients work together requires a little effort but is within the reach of any motivated programmer. On the other hand, this being a rather unusual way to express computations, the main difficulty is to find the right ingredients to solve a concrete problem.

That's the goal of this cookbook: provide you with detailed examples of the most frequent uses of recursion schemes that explain the thought process that leads from the definition of a problem to the expression of a solution in terms of pattern-functors and f-algebras.

In turns out that people seem to use recursion schemes to solve two major families of problems:
* Compiling or interpreting programs (ie working with ASTs)
* Manipulating schemas (or any generic representation of data)

Both come with its own set of challenges (although those sets have a non-empty intersection). This cookbook is therefore divided in two main parts, each dedicated to one of these two families of problems.

## Table of content

* [Working With Schemas](https://github.com/vil1/recursion-schemes-cookbook/tree/master/schemas)
  * [Working With Schemas Only](https://github.com/vil1/recursion-schemes-cookbook/tree/master/schemas#working-with-schemas-only)
    * [Knowing Where You Are](https://github.com/vil1/recursion-schemes-cookbook/tree/master/schemas#knowing-where-you-are)
  * [Working With Schemas and Data](https://github.com/vil1/recursion-schemes-cookbook/tree/master/schemas#working-with-schemas-and-data)
* [Working With ASTs](TODO)
