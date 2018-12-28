# Glossary

## A

### algebra

### anamorphism

### apomorphism

## C

### carrier

In a algebra like `f a -> a`, `a` is called the _carrier_.

### catamorphism

### chronomorphism

### coalgebra

## D

### distributive law

### dynamorphism

## E

### Elgot

A variation of generalized algebras that swaps the parameters from `f (w a) -> a` to `w (f a) -> a`. The name is taken from the “Elgot algebra” and is indicated in recursion scheme libraries with an `e` prefix.

## F

### `Fix`

A [fixed-point operator](#fixed-point-operator) implemented using native recursion. Depending on evaluation model of the language, it may be more akin to [Mu](#Mu) (in a strict language like Scala) or [Nu](#Nu) (in a lazy language like Haskell). It is generally unimplementable in total languages, but the clarity of its definition makes it useful for teaching recursion schemes.

### fixed-point operator

Any of a number of type constructors that have the (poly)kind `(k -> k) -> k` (most commonly seen where `k = *`). [Mu](#Mu), [Nu](#Nu), and [Fix](#Fix) are the most-frequently seen.

### fold

### fusion

### futumorphism

## G

### generalized algebra

### generalized fold

### greatest fixed point

## H

### histomorphism

### hylomorphism

## L

### least fixed point

## M

### metamorphism

### Mu (Μ)

### mutumorphism

## N

### natural transformation

Sometimes represented by the Greek lowercase eta (η).

### Nu (Ν)

## P

### pattern functor

### phi (ɸ)

### psi (ψ)

## U
 
### unfold

## Z

### zygomorphism

_zygo-_ is a prefix meaning something like “paired” (think of a _zygote_, which is a cell formed by the pairing of two gametes). And the algebra used for a zygomorphism is paired like that – `f (ann, a) -> a`. It pairs the carrier with an extra value providing context to the algebra.
