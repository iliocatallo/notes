# A structural approach to `Free`

## Introduction
This note presents `Free` from a sole structural standpoint. More specifically, we reduce the derivation of `Free` to the problem of designing a data type for trees without predetermined internal nodes. The reader should be familiar with recursive data types, type constructors, as well as the functor and monad abstraction. The code is written in Haskell.

## A data type for trees
Let us consider the following encoding of a tree:

```haskell
data Tree a = Leaf a | Branch (Tree a) (Tree a)
```

Note that `Tree` has an immediate `Monad` instance:

```haskell
instance Monad Tree where
  return x = Leaf x
  (Leaf x) >>= g = g x
  (Branch l r) >>= g = Branch (l >>= g) (r >>= g)
```

According to the above, `(>>=)` replaces each leaf node with whatever tree the function `g` returns. That is, `(>>=`) implements adjoining between trees. As an example, we can duplicate each leaf node by means of the `duplicateLeaf` function:

```haskell
duplicateLeaf :: a -> Tree a
duplicateLeaf x = Branch (Leaf x) (Leaf x)

t1 = Branch (Leaf 9) (Leaf 7)

-- t2 = Branch (Branch (Leaf 9) (Leaf 9)) (Branch (Leaf 7) (Leaf 7))
t2 = t1 >>= duplicateLeaf
```

## Arbitrary internal nodes
At present, `Branch` is the only possible structural form for internal nodes. The most immediate way to support alternative forms for internal nodes would be to extend the definition of `Tree` with additional data constructors. For instance, we may introduce a `Branch'` constructor to account for the case of an internal node with three child nodes:

```haskell
data Tree a = Leaf a
            | Branch (Tree a) (Tree a)
            | Branch' (Tree a) (Tree a) (Tree a)
```

However, one may wonder if a more general solution exists. That is, if it is possible to provide a definition of `Tree` that permits internal nodes of arbitrary structures without the need of explicitly encoding them in the definition of the data type.

As an example, we might be interested in internal nodes with either one, two or three child nodes. Let us enumerate these three alternatives in a dedicated data type, which we parameterize on the type of its child nodes.

```haskell
data Node e = Unary e | Binary e e | Ternary e e e
```

This is to say that a value of type `Node e` represents an internal node whose child nodes are of type `e`. Depending on the use case at hand, we may come up with different definitions for the internal nodes, such as:

```haskell
data Node' e = Empty | Unary e
```

Nonetheless, we would have a type constructor `f` (e.g., `Node`) parameterized on the type `e` of its child nodes. It is therefore a matter of altering the definition of `Tree` so as to account for whatever type constructor `f` happens to represent our internal nodes. A revised definition may read:

```haskell
data Tree f a = Leaf a | Branch (f e)
```

`Branch` now holds a single field, namely, the actual internal node of type `f e` (e.g., `Node e`). Still, the above is not valid Haskell, in that the type parameter `e` is only mentioned on the right-hand side of the definition. However, we know that `e` denotes the type of the child nodes. Since child nodes are themselves trees, that is, they are of type `Tree f a`, we conclude that the final definition should read:

```haskell
data Tree f a = Leaf a | Branch (f (Tree f a))
```

It is useful to contrast the original definition with what we have just derived:

```haskell
data Tree a = Leaf a | Branch (Tree a) (Tree a) -- original
data Tree f a = Leaf a | Branch (f (Tree f a))  -- parameterized on f
```

As shown, we are still in the presence of a recursive data type; the only difference being that internal nodes have two explicit child nodes (of type `Tree a`) in the original definition, while arbitrarily many child nodes (of type `Tree f a` ) in the second.

We are now in the condition of instantiating trees with different internal nodes. For instance:

```haskell
t1 :: Tree Node Integer
t1 = Branch (Unary (Leaf 5))

t2 :: Tree Node Integer
t2 = Branch (Binary (Branch (Unary (Leaf 7))) (Leaf 8))
```

As shown, both `t1` and `t2` are trees whose internal nodes are determined by `Node`, and whose leafs retain a value of type `Integer`.

## Recovering monadic facilities
The changes we implemented in `Tree` call for changes also in the corresponding `Monad` instance. For convenience, let us repeat the original definition:

```haskell
instance Monad Tree where
  return x = Leaf x
  (Leaf x) >>= g = g x
  (Branch l r) >>= g = Branch (l >>= g) (r >>= g)
```

The first straightforward adaptation is to include `f` in the context. In addition, observe that no changes are required to either `return` or the first match for `(>>=)`, as they both have to do with leaf nodes.

```haskell
instance Monad (Tree f) where
  return x = Leaf x
  (Leaf x) >>= g = g x
  (Branch y) >>= g = ...
```

The `Branch` case proves more challenging. In principle, we would like to apply `(>>= g)` to every child node the internal node at hand happens to contain. In this case however, we are only provided with `y`, which represents the entirety of the internal node, effectively masking its specific structure. As a consequence, we can no longer explicitly recurse over its child nodes, as we used to do in the original implementation.

We therefore need an operation on `y` and `(>>= g)` that would result in the application of `(>>= g)` to every child node within `y`. In other words, we would like to *map* every child node within `y` to whatever `(>>= g)` returns. Fortunately, there already exists an abstraction for expressing the *mappability* of a given type, named `Functor`.

Sticking to the example, the `Functor` instance for `Node e` may read:

```haskell
instance Functor Node where
  fmap f (Unary e) = Unary (f e)
  fmap f (Binary e1 e2) = Binary (f e1) (f e2)
  fmap f (Ternary e1 e2 e3) = Ternary (f e1) (f e2) (f e3)
```

Moving the mapping operation within the type functor `f` (e.g., `Node` ) should appear natural, as it is where we confined the explicit knowledge on the shape of the internal nodes. The intricacy of the mapping is hidden from the calling site by means of `fmap`.

The instance we just introduced is so immediate that GHC can automatically derive it for us, without further interventions.

```haskell
{-# LANGUAGE DeriveFunctor #-}

data Node e = Unary e | Binary e e | Ternary e e e deriving Functor
```

With the `Functor` instance in place, we can complete the specification of the `Monad` instance for `Tree` as:

```haskell
instance Functor f => Monad (Tree f) where
  return x = Leaf x
  (Leaf x) >>= g = g x
  (Branch y) >>= g = Branch (fmap (>>= g) y)
```

Note how we managed to recover the intended behavior: leaf nodes are replaced with whatever tree the function `g` happens to return; leaf nodes are reached by applying `(>>= g)` to the child nodes of every internal node. To show this, let us adapt the `duplicateLeaf` example we introduced at the beginning of our discussion:

```haskell
duplicateLeaf :: a -> Tree Node a
duplicateLeaf x = Branch (Binary (Leaf x) (Leaf x))

t3 = Branch (Binary (Leaf 9) (Leaf 7))
t4 = t3 >>= duplicateLeaf
-- t4 = Branch (Binary (Branch (Binary (Leaf 9) (Leaf 9)))
--                     (Branch (Binary (Leaf 7) (Leaf 7))))
```

## Introducing `Free`
The derivation of `Free` is just one renaming away. In particular, it amounts to renaming the data type from `Tree` to `Free`, and its data constructors from  `Branch` to `Free`, and from `Leaf` to `Pure`. The final data type is therefore defined as follows:

```haskell
data Free f a = Pure a | Free (f (Free f a))
```

## Conclusions and future work
The above derivation highlights `Free`'s affinity with well-known data types for representing trees. In particular, `Free` appears as a generalization that enables expressing trees with undetermined internal nodes. This intuition serves as a valuable starting point for getting acquainted with `Free`. Further elaborations are however necessary; most importantly, additional discussion on how to lift values of the inner type constructor `f` (e.g., `Node`) to values of type `Free f a` (e.g., `Free Node a`), with the aim of enabling `do` notation. Likely, a treatment of such a topic would benefit from the introduction of some semantics, such as qualifying the inner type constructor as a way of encoding some Domain-Specific Language (DSL). For this reason, it is somewhat out of the scope of this note.
