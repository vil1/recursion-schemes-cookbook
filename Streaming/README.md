# Streaming

“I want `anaM`.”

You probably don’t.

```haskell
anaM :: (Corecursive t f, Traversable f) => (a -> m (f a)) -> a -> m t
```

This looks like a reasonable enough type, and it‘s not hard to implement in most languages. But when you think through what it has to do, you can see the problem.

Say you have `expandFoo :: Natural -> Either String (Foo Natural)` as your algebra. Let’s first look at it with (non-monadic `ana)`.

```haskell
ana (Compose <<< expandFoo) :: a -> Nu (Compose (Either String) Foo)
```

This is unproblematic. However, we don’t have a nice `Either String (Nu Foo)` like we were hoping for. There’s an `Either` before each node, and we need to handle them one at a time. The only way to pull them out is to walk the whole structure, `sequenceA`ing each `Foo (Either String (Nu Foo))` to `Either String (Foo (Nu Foo)))` and `embed`ding it. But [what do we know about `Nu`?](../Mu-Nu/README.md) The values may be infinitely large. That’s a problem if we want to walk the whole structure.

That’s roughly what `anaM` does (in one pass, though). So, it’s necessarily partial – it may never complete.

How do we get around this?

There are two approaches, and I recommend you explore them in this order

## 1. use a fold instead

When you have an input with a `Recursive` instance, you can sometimes convert your `Coalgebra` to an `Algebra` (or, even better, to a `NaturalTransformation`). We lucked out – our example above uses `Natural`, which has `Recursive Natural Maybe`. So, after thinking hard for a while, perhaps we manage to get to `natToFoo :: Maybe (Mu Foo) -> Either String (Mu Foo)`.
```haskell
cataM natToFoo :: Natural -> Either String (Mu Foo)
```
This has to do the same traversal as we did above, but since we know `Natural` is finite, we can walk the structure without worrying about non-termination … which is exactly what `cata` already does, so we can `sequenceA` in that same pass.

Notice that we also have `Mu Foo` instead of `Nu Foo` in the result. This isn’t guaranteed, but often if you can fold to something instead of unfold to it, you get to keep the finitary proof that you started with.

But, this approach doesn’t always work, so …

## 2. try streaming

Doing this within a recursion scheme library requires writing a bit of your own machinery _for now_. But, the annoying type we had before – `Nu (Compose (Either String) Foo)` is very similar to what you see in effectful stream libraries. The types you often see look more like this, though
```haskell
type Stream m a = Nu (Coproduct m (XNor a))
```
- Where `Compose m f` means something like “always an `m` followed by an `f`”, `Coproduct m f` mean something like “either an `m` or an `f` at each step”. So, you might get five `Right`s in a row before seeing the next `Foo`.
- The `Either String` is generalized to an arbitrary functor.
- `Foo :: * -> *` is replaced by `XNor a :: * -> *`, which is the pattern functor for `List`-like things, since most streaming is done over a sequence of things.

But recursion schemes aren’t restricted in those ways – you can stream arbitrary tree structures, exploring only the branches you need to. For the time being, you should be able to do a lot of similar stuff with an effectful streaming library (say, `conduits` in Haskell or FS2 in Scala).
