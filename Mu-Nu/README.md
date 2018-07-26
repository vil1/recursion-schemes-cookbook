https://gitter.im/slamdata/matryoshka?at=5b46226f7b811a6d63e33981

# `Mu` is smaller than `Nu`

The various fixed-point operators are a source of confusion. You commonly see at least three: `Fix`, `Mu`, and `Nu`. When should you choose one over the other, and why do they all appear to have the same operations available?

For starters, `Mu` represents the “least fixed point” of a functor, and `Nu` represents the “greatest fixed point” of a functor. “Least” and “greatest” have some relative meaning there, but what does it really mean in this context? For our purposes, we can say that `Mu` models _finite_ structures and `Nu` models _potentially_-infinite ones. Note that values in `Nu` don’t have to be infinite – it subsumes all the values of `Mu` – but once we have a value of `Nu`, we have lost any guarantee that it is finite.

Where do other recursive structures fall on this spectrum from “least” to “greatest”? Well, it largely comes down to laziness – if a data structure is strict in its recursive parameter, it is finite, and if it is lazy, it is potentially-infinite.

```haskell
data LeastList a = Nil | Cons a !(LeastList a)
data GreatestList a = Nil | Cons a (GreatestList a)

data XNor a b = Neither | Both a b

type LeastList' a    = Mu (XNor a)
type GreatestList' a = Nu (XNor a)
```

In this example, we have created strict (`!`) and lazy versions of linked lists in Haskell. Since Haskell’s built-in list is also lazy, there is no finitary proof. Most data structures in Haskell are lazy, so they are the greatest fixed points of their functors, while in Scala, most data structures are strict, so they are the least fixed points of their functors. In Haskell, we tend to _pretend_ that our structures are finite (e.g., by calling `foldr` on a list, expecting it to terminate).

And that (finally) brings us to `Fix`. `Fix` is commonly defined in a way that makes what recursion schemes do as obvious as possible:

```haskell
data Fix f = Fix { unfix :: f (Fix f) }
```

```scala
final case class Fix[F[_]](unfix: F[Fix[F]])
```

These use “direct recursion” to show how `Fix` repeatedly nests the same functor recursively. But these two similar statements have quite different meanings – since Haskell is lazy by default, it’s `Fix` isn’t finitary, while Scala’s is. So, for some pedagogical ease, the notion of least/greatest fixed points is glossed over. In summary – Haskell’s `Fix` is akin to `Nu`, while Scala’s is akin to `Mu`.

You may be familiar with the phrase “ [making illegal states unrepresentable]”. This is often touted as a benefit of strong type systems. What this generally means is having the fewest values of your type without precluding any valid ones. So, as a rough guideline, we should try to use `Mu` when we can, and fall back to `Nu` when we have to.

## When does this question come up?

### `hylo`morphisms

### building structures

### transforming structures

Often you want to do a transformation from one recursive type to another one (although, this can frequently be avoided, but that is for a different chapter). If you know you are starting from a finite structure, you can retain that finitary proof by using a fold to transform the structure. However, if the result is potentially infinite, that approach won’t work, and you’ll be forced to use an unfold. Also, if you don’t know whether the input has a finitary proof (`Mu`) or not (`Nu`), then you can use an unfold, and you will end up with a result without a finitary proof.

Let’s take a simple example – `drop`, which will shorten a list by some amount. It is common to see this as an unfold operation:
```haskell
drop :: Coalgebra (Either [a]) (XNor a) (Natural -> [a])
drop l 0 = 
```
