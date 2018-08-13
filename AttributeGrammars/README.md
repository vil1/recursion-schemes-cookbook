# How do I Pass Data Toward the Leaves?

https://functionalprogramming.slack.com/archives/C0433UN0M/p1532450806000153

## Just another word for function

Attribute grammars are a useful notion, but in recursion schemes, they generally boil down to `Algebra f (s -> a)`. That is, a fold to a function. Having access to this function in your algebra lets you pass data _toward_ the leaves even though the fold is traversing _from_ the leaves.
