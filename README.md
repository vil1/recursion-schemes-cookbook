# Recursion Schemes Cookbook

Welcome to the recursion schemes cookbook! This document is intended as a complement to [matryohska](https://github.com/slamdata/matryoshka)'s documentation. Its goal is to help you come up with strategies to solve real-life problems using matryoshka (and recursion schemes in general).

## Reading this book

We can see recursion schemes as a (quite unusual) way to build functions by using a recursive data structure as a blueprint of the computation (a... pattern). Understanding how the three ingredients work together requires a little effort but is within the reach of any motivated programmer. On the other hand, this being a rather unusual way to express computations, the main difficulty is to find the right ingredients to solve a concrete problem.

That's the goal of this cookbook: provide you with detailed examples of the most frequent uses of recursion schemes that explain the thought process that leads from the definition of a problem to the expression of a solution in terms of pattern-functors and f-algebras.

In turns out that people seem to use recursion schemes to solve two major families of problems:
* Compiling or interpreting programs (ie working with ASTs)
* Manipulating schemas (or any generic representation of data)

Both come with its own set of challenges (although those sets have a non-empty intersection). This cookbook is therefore divided in two main parts, each dedicated to one of these two families of problems.

## Table of contents

* [Working With Schemas](schemas/README.md)
  * [Working With Schemas Only](schemas/README.md#working-with-schemas-only)
    * [Knowing Where You Are](schemas/README.md#knowing-where-you-are)
    * [Remembering What You Did Before](schemas/README.md#remembering-what-you-did-before)
  * [Working With Schemas and Data](schemas/README.md#working-with-schemas-and-data)
* Working With ASTs
  * [Which fixed-point operator should I use?](Mu-Nu/README.md)
  * [How do I convert between (co)recursive structures?](NaturalTransformations/README.md)
  * [How do I pass data toward the leaves?](AttributeGrammars/README.md)
  * How do I restrict which nodes can occur where? (mutual recursion, inductive type families)
  * How can I apply this to DAGs? (`RootedGraph`)
  * Fixing frustrations of directly-recursive types
    * How do I avoid creating two ASTs that are 90% the same, so I can tighten the type after a transformation? (`Coproduct`)
    * How do I annotate a tree? (`zygo`, `EnvT`)
    * How do I compose algebras and coalgebras efficiently?
* [Streaming](Streaming/README.md)
