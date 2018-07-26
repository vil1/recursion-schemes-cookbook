https://gitter.im/slamdata/matryoshka?at=5b5742a2c86c4f0b472f38c0

When you’re transforming data between two recursive structures, your best bet is to define a natural transformation rather than using an `Algebra` or `Coalgebra` directly.

```haskell
-- ideally
myNat :: forall a. f a -> g a
-- however, sometimes your `f` can result in multiple `g`s
myNatOrMore :: forall a. f a -> g (Free g a)
-- and sometimes zero `g`s
myNatMoreOrLess :: forall a. f a -> Free g a

-- so now you can
cata (embed <<< myNat)
-- or
ana (myNat <<< project)
-- the former can give you a finite structure (but only if you’ve started with
-- one) and the later can take any structure, but will result in a potentially-
-- infinite one. So rather than defining both a `Algebra` and a `Coalgebra` for
-- the various cases, you have one natural transformation for whichever you
-- need.

-- The Free cases are _mostly_ a bit trickier, except for
gana distFree (myNatOrMore <<< project)

-- But there’s also magical fusion …
myGAlg :: Algebra g b

myFFold :: Mu f -> b
myFFold cata myAlg <<< cata (embed <<< myNat)

myFFold' :: Mu f -> b
myFFold' = cata (myAlg <<< myNat)
-- natural transformations compose very nicely with whatever other (co)algebras
-- you may have, eliminating ever constructing a value of `Mu g` along the way.
```
