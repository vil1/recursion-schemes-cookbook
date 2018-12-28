# How do I  annotate a tree?

Once you’ve created a [pattern functor](../glossary.md#pattern-functor) for your structure, there is a general-purpose structure for tree annotation – `Cofree f ann`. Since it is also a tree, it has a pattern functor itself – `EnvT ann f a`. These two types _may_ be familiar to you from other contexts. `Cofree` is the dual to `Free` and has had corresponding recognition for a particular use case. `EnvT` is known as the environment monad transformer. However, we’re not taking advantages of any of these things – we only care about the structure. `EnvT ann f a` is isomorphic to `(ann, f a)`, so you can picture `Cofree` similarly – as a recursive tuple, where `fst` contains the annotation and `snd` contains the tree from the current node.

The most direct way to annotate a tree is to use an [algebra](../glossary.md#algebra) like `f (Cofree f a) -> Cofree f a`, at each step, creating the next level of the tree. But there are a number of ways to reduce the boilerplate implied by this (and make the operation more general).

## `Transform`

Rather than explicitly [folding](../glossary.md#fold), you can create a [natural transformation](../glossary.md#natural-transformation) `forall a. f a -> EnvT ann f a`. This is only possible if you don’t need context from the rest of the tree to create the annotation, but if it _is_ possible, then [you get a lot of flexibility](../NaturalTransformations/README.md).

## `attributeAlgebra`

(NB: Should perhaps rename this to `annotateAlgebra`? Also, `attributeAlgebra` in Matryoshka is probably very broken.)

`attributeAlgebra :: (f ann -> ann) -> f (Cofree f ann) -> Cofree f ann`

This “algebra transformer” can convert an algebra can calculate the annotation for a node based on the annotations of its children into one that annotates the original tree.

This is another example of how recursion schemes allow us to focus on only one thing at a time. We don’t need to think about annotating the tree with a value – only about how to calculate that value for the node we’re looking at. And then there are generic operations to give us the fully annotated tree from that.

## But …

Often you annotate a tree and then consume the annotation in a second pass immediately afterward. And one of the things that recursion schemes supposedly offers is a way to “fuse” multiple passes over a structure into one. Is there some way we can do that here?

Yes! There is a generalized fold called a “[zygomorphism](../glossary.md#zygomorphism)”. _zygo-_ is a prefix meaning something like “paired” (think of a _zygote_, which is a cell formed by the pairing of two gametes). And the algebra used for a zygomorphism is paired like that – `f (ann, a) -> a`. It pairs the [carrier](../glossary.md#carrier) with an extra value providing context to the algebra.

Where does that context come from, though? The name of the type variable may give us a hint – we previously had an algebra that could give us the annotation we want – `f ann -> ann`. So, a zygomorphism takes this “annotation algebra” in addition to the algebra containing the tuple.

Here’s an example where we have `annotate` as the “annotation algebra” and `consume` as the primary algebra.
```haskell
annotate :: f ann -> ann
consume :: f (ann, a) -> a

myFold :: Mu f -> a
myFold = gcata (distTuple annotate) consume
```
```scala
annotate: F[Ann] => Ann
consume: F[(Ann, A)] => A

myFold: Mu[F] => A = _.zygo(annotate, consume)
```

You might have noticed that `f (ann, a)` isn’t quite the same shape as our `EnvT` pattern functor. Often this new shape works, but there is a [Elgot](../glossary.md#Elgot) variation that gives us the `(ann, f a)`  shape we might need
```haskell
econsume :: (ann, f a) -> a

myFold' :: Mu f -> a
myFold' = egcata (distTuple annotate) econsume
```
```scala
econsume: (Ann, F[A]) => A

myFold0: Mu[F] => A = _.ezygo(annotate, econsume)
```

So, we now have a fusion property for annotation –
```haskell
cata (econsume . runEnvT) . cata (attributeAlgebra annotate)
    == egcata (distTuple annotate) econsume
-- Alternatively, if your `econsume` algebra uses `EnvT` (as it would if you had
-- previously been folding a Cofee):
cata econsume             . cata (attributeAlgebra annotate)
    == egcata (distTuple annotate) (econsume . EnvT)
```

There is a simpler equality for the case of simply _building_ an annotated tree, without additionally folding it.
```haskell
cata (attributeAlgebra annotate) == egcata (distTuple annotate) (embed . EnvT)
```
