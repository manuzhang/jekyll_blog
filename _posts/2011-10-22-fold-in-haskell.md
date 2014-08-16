---
author: manuzhang
comments: true
date: 2011-10-22 15:00:55+00:00
layout: post
published: false
slug: fold-in-haskell
title: fold in Haskell
wordpress_id: 59
categories:
- Haskell
tags:
- fold
- functional programming
---

**WARNING: this article needs revision**



**Overview**

`<br />
<strong>fold :: (a -> b -> b) -> b -> ([a] -> b)</strong><br />
<strong> fold f v [] = v</strong><br />
<strong> fold f v (x:xs) = f x (fold f v xs)</strong><br />`



Given a function _f_ of type _a -> b -> b_ and a value _v_ of type _b_, the function _fold f v_ processes a list of type _[a]_ to give type _b_ by replacing the _nil_ constructor [] at the end of the list by the value _v_, and each _cons_ constructor (:) within the list by the function _f_



The constructor (:) can be replaced by a built-in function as well as user-defined function, often defined as a nameless function using the lambda notation.



<!-- more -->

`<br />
<strong>sum :: [a] -> Int</strong><br />
<strong> sum = fold (+) 0</strong></p>
<p><strong>length :: [a] -> Int</strong><br />
<strong> length = fold (x n -> 1 + n) 0</strong></p>
<p><strong>reverse :: [a] -> [a]</strong><br />
<strong> reverse = fold (x xs -> xs ++ [x]) []</strong></p>
<p><strong>map :: (a -> b) -> ([a] -> [b])</strong><br />
<strong> map f = fold (x xs -> f x：xs) []</strong></p>
<p><strong>filter :: (a -> Bool) -> ([a] -> [a])</strong><br />
<strong> filter p = fold (x xs -> if p x then x:xs else xs) []</strong><br />`



**The universal property of fold **

`<br />
<strong>g [] = v</strong><br />
<strong> g (x:xs) = f x (g xs)</strong><br />
<strong> <=></strong><br />
<strong> g = fold f v</strong><br />`



The **universal property** states that for finite lists the function _fold f v_ is not just a solution to its defining equations, but in fact the _unique_ solution



Let's apply the **universal property** to proving the equation







  * (+1).sum = fold (+) 1*



where _f.g x = f(g(x))_



The left-hand composite function sums a list and then increments the result. We can put it in a **Haskell** function

`<br />
<strong>((+1).sum) [] = sum [] + 1 = 0 + 1 = 1</strong><br />
<strong> ((+1).sum) (x:xs) = sum (x:xs) + 1 = (x + sum xs) + 1 = x + (sum xs + 1) = (+) x (((+1).sum) xs)</strong><br />`



Now let's substitute _g_ for _((+1).sum)_ and yield the left half of **universal property** and by moving from left to right we can achieve that v is 1 and f is (+). What do we get?



The right-hand of the equation: _fold (+) 1_



**The fusion property of fold**



Remember the definition of sum:



**`(+1).sum = (+1).fold (+) 0`**

Connect it with our conclusion just now:



**`(+1).sum = fold (+) 1`**

We jump to a new equation:



**`(+1).fold (+) 0 = fold (+) 1`**

You may argue that there is no magic here since the difference between the two hands is whether increment by 1 at first or at last. Nonetheless, let's look at a more general case:

`<br />
<strong>h.fold g w [] = h (fold g w [])</strong><br />
<strong> = h w</strong><br />
<strong> h.fold g w (x:xs) = h (fold g w (x:xs))</strong><br />
<strong> = h (g x (fold g w xs))</strong><br />
<strong> = h.g x (fold g w xs)</strong><br />
<strong> = h.g x y</strong><br />`

Compare the above equation with



`<br />
<strong>g' [] = v</strong><br />
<strong> g' = fold f v</strong><br />
<strong> g' (x:xs) = f x (g' xs)</strong><br />`

Finally, here's the **fusion property**:



`<br />
<strong>h w = v</strong><br />
<strong> h.g x y = f x (g' xs)</strong><br />
<strong> = f x (h y)</strong><br />
<strong> where y = fold g w xs</strong><br />
<strong> =></strong><br />
<strong> h.fold g w = fold f v</strong><br />`

A simple application of fusion shows that:



**`(x a).fold (x) b = fold (x) (b x a)`**

x is an arbitrary infix operator



A more interesting example asserts that the _map_ operator distributes over function composition (.):



`<br />
<strong>(map f).(map g) = map (f.g)</strong><br />
<strong> (map f).(fold (x xs -> (g x):xs)) [] = fold (x xs -> ((f.g) x):xs) []</strong><br />`

Try applying the **fusion property**:

`<br />
<strong>map f [] = []</strong><br />
<strong> (map f).(x xs -> (g x):xs) x y = (x xs -> ((f.g) x):xs) x (map f y)</strong><br />
<strong> where y = fold (x xs -> g x:xs) [] xs</strong><br />
<strong> (map f).((g x):y) = ((f.g) x):(map f y)</strong><br />
<strong> where y = fold map g [] xs</strong><br />`



**Universality as a definition principle**



Let's recall that



**`sum = fold (+) 0`**

but how we have deduced the answer?



First of all, we may think recursively to calculate the sum of a list of numbers:



`<br />
<strong>sum :: [Int] -> Int</strong><br />
<strong> sum [] = 0</strong><br />
<strong> sum (x:xs) = x + sum xs</strong><br />
<strong> = (+) x (sum xs)</strong><br />`

If we substitute _sum_ for _g_, _(+)_ for _f_, ** for _v_, that is exactly what **universal property** states



In more complicated cases, the solution may not be apparent from observation. Considering:



`<br />
<strong>map :: (a -> b) -> ([a] -> [b])</strong><br />
<strong> map f [] = []</strong><br />
<strong> map f (x:xs) = (f x):(map f xs)</strong><br />`

So how to solve the equation _map f = fold g v_? As always, turn to the definition of **fold**:



`<br />
<strong>map f [] = v</strong><br />
<strong> map f (x:xs) = g x (map f xs)</strong><br />`

It is immediate that _v = []_. From the second equation:



`<br />
<strong>map f (x:xs) = g x (map f xs)</strong><br />
<strong> (f x):(map f xs) = g x (map f xs)</strong><br />
<strong> (f x):ys = g x ys</strong><br />
<strong> where ys = map f xs</strong><br />
<strong> g = x ys -> (f x):ys</strong><br />
<strong> => map f = fold (x ys -> (f x):ys) []</strong><br />`



