# How do I Pass Data Toward the Leaves?

One common problem that often leads developers toward `ana` is the desire to be able to pass some extra data (“attributes”) toward the leaves. Since `ana` works top-down, building the outermost nodes first, this is a natural intuition. And if your transformation looks like `Mu f -> Nu g`, then `Coalgebra g (Mu f)` can easily become `Coalgebra g (attributes, Mu f)` and you’re on your way!

However, things are often more complicated than that. For example, what if your transformation is more like `Mu f -> a`, or even worse, `Mu f -> Either error (Nu g)`? These are cases where `cata` tends to serve you much better, but you still have to figure out how to pass data in the opposite direction of the fold!

## Avoiding Partiality

`Mu f -> a` is a problem for `ana`, because the result needs to be some fixed-point type, but we simply have `a`. The simple approach for this is the `Partial` monad, which is `type Partial a = Nu (Either a)`. This gets us close to the type we want: `Mu f -> Partial a`. But what that `Partial` type means is that forcing the result may never terminate. (Pretty sweet that the type system can represent that, right?) Sometimes `Partial` is totally justified, but if you’re introducing it simply to pass data toward the leaves, you’re better off trying to use a fold.

`Mu f -> a` is pretty straightforward as folds go. Just use `Algebra f a`. But the problem is that during the transformation, we somehow need to pass attributes from closer to the root. To do this, we introduce the attributes as an “environment” in which the value (`a`) is calculated. The environment monad is usually called `Reader`, and `Reader a b` is `a -> b`. So, we can use `Algebra f (attributes -> a)` to introduce attributes passed from the root. At each step, the current node can pass whatever `attributes` it wants to its children – taking into account the `attributes` passed to itself by its parent or not. For the entire fold, it means the result is a function, which needs to be passed some initial value (often something like a monoidal identity). This isn’t too different from the `Coalgebra` case above, where the unfold needs to be passed a tuple with the initial attribute along with the seed.

## Avoiding `Unsafe` Partiality

In the case of `Mu f -> Either error (Nu g)`, it’s tempting to see the `Nu` in the result and lean toward an unfold. However, using a Coalgebra like `Mu f -> Either error (g (Mu f))` is problematic, because the only way to pull the monad (`Either error` in this case) to the outside of the entire fixed-point is force the entire tree to ensure that you never get a `Left` anywhere. But the entire point of `Nu` is to indicate that you can’t guarantee the tree is finite. So you’re really looking to blow things up. This form of partiality is unsafe, because it isn’t represented in the type. So while Haskell and Scala libraries tend to provide these operations, there’re not even possible in total languages.

The same technique can be used as above to avoid this problem – using the “environment” monad again. There’s a bit more variation here, as _where_ the function goes can vary. If you can identify the error without the attributes, `AlgebraM (Either error) f (attributes -> Nu g)` is a reasonable choice. It has the benefit of letting `cataM` handle the threading of the monad for you. However, if the error is only identified _with_ the attributes, then you’re stuck with `Algebra f (attributes -> Either error (Nu g))`, which means you have to manage the monad yourself.

## Be Aware

Even though `((->) attributes)` is itself a monad, you can’t use `AlgebraM (Reader attributes) f (Nu g)`. While both result in `attributes -> Nu g` after folding, `AlgebraM` (with `cataM`) will result in every node being passed the same value for `attributes` that you pass at the top level. This is simply a consequence of the general way a monad can be extracted over the fold. To be able to control the values for `attributes`, the function must be managed explicitly in the algebra, which is what `Algebra f (attributes -> Nu g)` requires.

## Just another word for function

“Folding to a function” is an implementation of a technique from Knuth called “[attribute grammars](https://en.wikipedia.org/wiki/Attribute_grammar)”. In a attribute grammar, a formal grammar may have “inherited” and “synthesized” attributes attached to help it calculate a value. Inherited attributes are inherited from the parent and synthesized attributes are passed from the children. In recursion schemes, that looks like
```haskell
Algebra f (inherited -> (synthesized, a))
```
Any attribute grammar should be able to be modeled this way. As a result, I often say “attribute grammar” when I mean a fold with a carrier that’s a function (which is intended to pass some attribute toward the leaves).
