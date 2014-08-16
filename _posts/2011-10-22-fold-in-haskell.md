---
layout: post
title: fold in Haskell
comments: true
---

## Overview


```haskell
fold :: (a -> b -> b) -> b -> ([a] -> b)
fold f v [] = v
fold f v (x:xs) = f x (fold f v xs)
```

Given a function `f` of type `a -> b -> b` and a value `v` of type `b`, the function `fold f v` processes a list of type `[a]` to give type `b` by replacing the `nil` constructor [] at the end of the list by the value `v`, and each `cons` constructor `:` within the list by the function `f`. The constructor `:` can be replaced by a built-in function as well as user-defined function, often defined as a nameless function using the lambda notation.

```haskell
sum :: [a] -> Int
sum = fold (+) 0

length :: [a] -> Int
length = fold (x n -> 1 + n) 0

reverse :: [a] -> [a]
reverse = fold (x xs -> xs ++ [x]) []

map :: (a -> b) -> ([a] -> [b])
map f = fold (x xs -> f x：xs) []

filter :: (a -> Bool) -> ([a] -> [a])
filter p = fold (x xs -> if p x then x:xs else xs) []
```

## The universal property of fold 

```haskell
g [] = v
g (x:xs) = f x (g xs)
<=>
g = fold f v
```

The **universal property** states that for finite lists the function `fold f v` is not just a solution to its defining equations, but in fact the **unique** solution. Let's apply the **universal property** to prove the equation `(+1).sum = fold (+) 1` where `f.g x = f(g(x))`.
The left-hand composite function sums a list and then increments the result. We can put it in a **Haskell** function

```haskell
((+1).sum) [] = sum [] + 1 = 0 + 1 = 1
((+1).sum) (x:xs) = sum (x:xs) + 1 = (x + sum xs) + 1 = x + (sum xs + 1) = (+) x (((+1).sum) xs)
```

Now let's substitute `g` for `((+1).sum)` and yield the left half of **universal property** and by moving from left to right we can achieve that `v` is `1` and `f` is `+`. What do we get?

The right-hand of the equation: `fold (+) 1`



## The fusion property of fold

Remember the definition of sum: `(+1).sum = (+1).fold (+) 0`
Connect it with our conclusion just now: `(+1).sum = fold (+) 1`
We jump to a new equation: `(+1).fold (+) 0 = fold (+) 1`

You may argue that there is no magic here since the difference between the two hands is whether increment by 1 at first or at last. Nonetheless, let's look at a more general case:

```haskell
h.fold g w [] = h (fold g w [])
 = h w
h.fold g w (x:xs) = h (fold g w (x:xs))
 = h (g x (fold g w xs))
 = h.g x (fold g w xs)
 = h.g x y
```

Compare the above equation with

```haskell
g' [] = v
g' = fold f v
g' (x:xs) = f x (g' xs)
```

Finally, here's the **fusion property**:

```haskell
h w = v
h.g x y = f x (g' xs)
 = f x (h y)
 where y = fold g w xs
 =>
 h.fold g w = fold f v
```

A simple application of fusion shows that:

`(x a).fold (x) b = fold (x) (b x a)`

x is an arbitrary infix operator

A more interesting example asserts that the `map` operator distributes over function composition `.`:

```haskell
(map f).(map g) = map (f.g)
(map f).(fold (x xs -> (g x):xs)) [] = fold (x xs -> ((f.g) x):xs) []
```

Try applying the **fusion property**:

```haskell
map f [] = []
 (map f).(x xs -> (g x):xs) x y = (x xs -> ((f.g) x):xs) x (map f y)
 where y = fold (x xs -> g x:xs) [] xs
 (map f).((g x):y) = ((f.g) x):(map f y)
 where y = fold map g [] xs
```

## Universality as a definition principle

Let's recall that `sum = fold (+) 0` but how we have deduced the answer?

First of all, we may think recursively to calculate the sum of a list of numbers:

```haskell
sum :: [Int] -> Int
 sum [] = 0
 sum (x:xs) = x + sum xs
 = (+) x (sum xs)
```

If we substitute `sum` for `g`, `(+)` for `f`, `**` for `v`, that is exactly what **universal property** states
In more complicated cases, the solution may not be apparent from observation. Considering:

```haskell
map :: (a -> b) -> ([a] -> [b])
 map f [] = []
 map f (x:xs) = (f x):(map f xs)
```

So how to solve the equation `map f = fold g v`? As always, turn to the definition of **fold**:

```haskell
map f [] = v
map f (x:xs) = g x (map f xs)
```

It is immediate that `v = []`. From the second equation:

```haskell
map f (x:xs) = g x (map f xs)
 (f x):(map f xs) = g x (map f xs)
 (f x):ys = g x ys
 where ys = map f xs
 g = x ys -> (f x):ys
 => map f = fold (x ys -> (f x):ys) []
```

