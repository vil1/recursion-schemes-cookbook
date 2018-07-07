# Working With Schemas

Schemas are tree-like objects that unambiguously and completely describe a data structure. Schemas are useful whenever we need to move data in and out across the boudnaries of our applications. A generic way to use and take advantage of schemas regardless of their concrete representation (there is a bunch of competing schema formats out there like [Avro](), [Thrift](), [Protobuf](), etc) is therefore much needed.

### The SchemaF Pattern-Functor

We obviously need a pattern-functor whose structure reflects the general structure of schemas. Of course you'll want to tailor this pattern to fit your business problem, keeping only the bits of the general structure you need, but you'll probably end up with something that looks like the following:

```scala
sealed trait SchemaF[A]

// struct aka record aka object aka dictionnary
final case class StructF[A](fields: Map[String, A]) extends SchemaF[A]

final case class ArrayF[A](element: A) extends SchemaF[A]

final case class UnionF[A](alternatives: List[A]) extends SchemaF[A]

// suppose we have a `Type` ADT that represents simple types as Int, String, etc
final case class ValueF[A](tpe: Type) extends SchemaF[A]
```

You might not care about unions at all and therefore omit the `UnionF` case, or you might prefer to have a specific case for every simple type (like `final case class IntF[A]() extends SchemaF[A]`) and so on. 

In any case, it's always a good idea to provide a `Traverse` instance for your pattern right away, rather than just a mere `Functor`. It'll allow you to use all the "monadic-flavored" schemes out of the box.

## Working With Schemas Only

A lot of work can be done using only a schema or to put it differently, a lot of interesting functions can be built by applying some f-algebras to our `SchemaF` pattern.

We can produce functions that convert schemas between different formats (Avro, Thrift, Protobuf, etc), that produce data-validator for a schema, that ensure compatibility across different versions of a schema and so on.

### Knowing Where You Are

It's sometimes needed to know the position of a given schema "node" within the global schema. For example, you want the data-validator you're building to add a precise path to the error messages it produces.

Such information is not present in our pattern-functor (and it shouldn't be in yours either). This means that we need a way to *label* each "node" of our schema with its path first.

First we need to notice that the only way to build such paths is to go top-down from the "root" of our schema, which means that we're looking for a coalgebra. 

Provided we can compute such path, what we want is to label each element of a given schema with its path. The `matryoshka.patterns.EnvT` pattern-functor allows just that. Given a label type `E` and a (pattern-)functor `F`, `EnvT[E, F, ?]` is the pattern-functor that has the exact same structure as `F` but with each "node" labelled with a value of type `E`.

``` scala
final case class EnvT[E, F[_], A](run: (E, F[A]))
```

So we want to build a `Coalgebra[EnvT[Path, SchemaF, ?], A]` but we still need to find the right carrier (the `A` type variable). We will surely need our recursive `T` on `SchemaF`, but we also need to *carry along* the path from the root, so we'll use `(Path, T)` as our carrier. Now we're ready to implement our coalgebra.

``` scala
def labelWithPath[T](implicit T: Recursive.Aux[T, SchemaF]): Coalgebra[EnvT[Path, SchemaF, ?], (Path, T)] = { 
  case (path, t) =>
    t.project match {
      case StructF(fields) =>
        EnvT((path, StructF(fields.map{ case (name, tt) =>
          name -> (path / name, tt)
        })))
      case schema => EnvT((path, schema))
    }
}
```

For every struct field, we *push down* a new path that is the current path to which we append the name of that field. The other case have no influence on the path so we simply push the current one.

We need to do that by hand because the name of a field is not visible from the field itself, so we have no way to write a function of type `(Path, SchemaF[T]) => Path` that would actually build a correct path.

If we were able to write such function, we could use the `attributeTopDown` function from the `Recursive` trait. For example, labelling each element of a schema with its depth would look like:

``` scala
def labelWithDepth[T, U](t: T)(implicit T: Recursive.Aux[T, SchemaF],
                                        U: Corecursive.Aux[U, EnvT[Int, SchemaF, ?]]): U = 
  T.attributeTopDown[U, Int](t, 0) {
    case (depth, _) => depth + 1
  }
```

### Remembering What You Did Before

The main strength of recursion schemes is that you get to process a "tree" one "node" at a time, which means that your algebra is oblivious of the result it produced before (elswhere on the "tree"). 

Lets be a bit more precise. The carrier of any f-algebra carries precisely the result of the previous execution of the said f-algebra, so we always rembember what we did just before. But what if we need to remember what we did on a totally unrelated part of the tree?

Concretely, say we want to serialize any `T` recursive on `SchemaF` to `org.apache.avro.Schema`. It's definitely doable since we can express every possible `SchemaF` case in terms of `avro.Schema` structure-wise. The problem is that the Avro API mandates that, when building a `Schema`, we register every record (the Avro representation of our `StructF`) under a name that is unique across the whole `Schema`. This is because Avro is primarily meant as a binary representation of some classes in a codebase, but that's not the case there, so these mandatory unique names are irrelevant to us. All we're interested in is the *structure* of our `SchemaF`, so if we're able to deterministically derive a name from an arbitrary `SchemaF` we're good to go, provided that we can remember the name we've already registered.

So we need to write an algebra that works in a *context* were it can *register* and *lookup* facts, that is a context that's able to maintain an *updatable state*, that is the `State` monad.

In conclusion, we want to write an `AlgebraM[State[Registry, ?], SchemaF, avro.Schema]`. In the `Registry` managed by the state monad, we'll store a mapping from name to `Schema` of all the partial records we've already built. That way we'll be able to avoid duplicate names.

``` scala
import org.apache.avro.Schema

type Registry = Map[String, Schema]

def name(structure: Map[String, Schema]): String                         = ???
def buildRecordSchema(name: String, fields: Map[String, Schema]): Schema = ???

def toAvroAlg: AlgebraM[State[Registry, ?], SchemaF, Schema] = {

  case StructF(fields) =>
    State { registry =>
      val n = name(fields)
      if (registry.contains(name)) // recalling what we did in the past
        (registry, registry(name))
      else {
        val schema = buildRecordSchema(name, fields)
        (
          registry + (name -> schema), // memorizing what we just did 
          schema
        ) 
      }
    }

}


def toAvro[T](t: T)(implicit T: Recursive.Aux[T, SchemaF]): Schema = 
  t.cataM(toAvroAlg).run(Map.empty)._2
```

As a bonus, since we reuse a record we've already registered when we encounter a `StructF` that has the exact same structure (provided that `name` is injective) we're guaranttied to build the most compact `Schema` we can.

## Working With Schemas And Data
