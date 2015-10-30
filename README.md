Algebraic
=========

The goal of this experiment is to explore ways to construct functions
which are richer than the typical Haskell functions (of type `(->)`). In
particular, functions which are known to be bijective, or injective, or
surjective, or total, or partial, etc.

The foundation is this type, from [Data.Algebraic.Function](Data/Algebraic/Function.hs)

```Haskell
data F g h s t = F {
      to :: g s t
    , from :: h t s
    }
```

If `g` and `h` are arrows, then `F g h` is a category, so we can use it as
we would the typical function type `(->)`. Judiciously choosing types for
`g` and `h` gives rise to many familiar mathematical notions:

```Haskell
-- Total functions always produce a value...
type TotalFunction = F (Kleisli Identity) (EmptyArrow)

-- ... but partial functions do not.
-- Notice how the choice of EmptyArrow means we don't have to give any
-- information for the 'from' part of the definition.
type PartialFunction = F (Kleisli Maybe) (EmptyArrow)

-- Bijections can always be inverted...
type TotalBijection = F (Kleisli Identity) (Kleisli Identity)

-- ... injections can be inverted too, but not everything in the codomain
-- has a preimage.
type TotalInjection = F (Kleisli Identity) (Kleisli Maybe)

-- Surjections always give at least one preimage.
type TotalSurjection = F (Kleisli Identity) (Kleisli NonEmpty)
```

Functions of different types can be composed in such a way that the maximal
amount of information is preserved. For instance, if you compose a total
bijection with a total injection, you will get a total injection. This
is all supported by a very convenient ordering on the arrows in
question:

```
                    Kleisli Identity
                ^                      ^
               /                        \
              /                          \
             /                            \
          Kleisli Maybe          Kleisli NonEmpty
            ^         ^          ^         ^
            |          \        /          |
            |          Kleisli []          |
            |                              |
            \              ^               /
             \             |              /
              \                          /
                      EmptyArrow

```

In this diagram, there is a line from `x` to `y` if and only if there is
an arrow homomorphism from `y` to `x`. Notice the one in the middle:
`Kleisli []`. This is just a normal function: every image has 0 or more
preimages.

Really, we're concerned with the greatest-lower-bounds in this ordering,
which are captured by the type family `GLB` and the related class
`WitnessGLB`, instances of which are used to give:

```Haskell
fcompose
    :: forall f1 g1 f2 g2 s t u .
       ( Composable f1 f2
       , Composable g1 g2
       )
    => F f2 g2 u t
    -> F f1 g1 s u
    -> F (GLB f2 f1) (GLB g2 g1) s t
```

By using `(<.>)` as a replacement for the typical categorical composition
`(.)`, we allow GHC to infer the types of our functions.

```Haskell
plus :: Int -> F Total Bijection Int Int
plus i = F (arr ((+) i))
           (arr (\x -> x - i))

isPositive :: F Total Surjection Int Bool
isPositive = F (arr (\x -> x > 0))
               (Kleisli $ \b -> if b
                   then (1 :| [2..])
                   else fmap negate (0 :| [1..])
               )

boolNot :: F Total Bijection Bool Bool
boolNot = F (arr not)
            (arr not)

-- GHC infers the type
--
--   example :: F (Kleisli Identity)
--                (Kleisli NonEmpty)
--                Int
--                Bool
--
-- meaning we have a total surjection from Int to Bool, as witnessed by
--
--   image :: Int -> Bool
--   image = runIdentity . runKleisli (to example)
--
--   preimages :: Bool -> NonEmpty Int
--   preimages = runKleisli (from example)
-- 
example = boolNot <.> isPositive <.> plus 5
```

Constructing functions
======================

So we can build rich functions using a category-like interface; big deal.
But things get interesting when we mix in algebraic datatypes.
By giving types for sums and products, as we do in
[Data.Algebraic.Product](Data/Algebraic/Product.hs) and
[Data.Algebraic.Sum](Data/Algebraic/Sum.hs), we can construct rich functions
componentwise. That's to say, if you give a product of `F`s, then you have
an `F` on a corresponding product or sum:

```Haskell
productF :: (F f1 g1 s1 t1 :*: ... :*: F fn gn sn tn)
         -> F (GLBFold [f1,..,fn])
              (GLBFold [g1,..,gn])
              (s1 :*: ... :*: sn)
              (t1 :*: ... :*: tn)

sumF :: (F f1 g1 s1 t1 :*: ... :*: F fn gn sn tn)
     -> F (GLBFold [f1,..,fn])
          (GLBFold [g1,..,gn])
          (s1 :+: ... :+: sn)
          (t1 :+: ... :+: tn)
```

These are not the types as they appear in GHCi, since these functions are
implemented via type classes, but that's what they essentially are.
