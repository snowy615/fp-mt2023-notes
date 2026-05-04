# Functional Programming — MT2023 Concept Reference
*Geraint Jones, University of Oxford. Notes compiled from all 16 lectures.*

---

## Table of Contents
1. [Types and Function Application](#1-types-and-function-application)
2. [Lists and List Comprehensions](#2-lists-and-list-comprehensions)
3. [Pattern Matching and Data Types](#3-pattern-matching-and-data-types)
4. [Strictness, Laziness, and Bottom](#4-strictness-laziness-and-bottom)
5. [Higher-Order Functions](#5-higher-order-functions)
6. [Fold (foldr)](#6-fold-foldr)
7. [Unfold](#7-unfold)
8. [Fold Fusion](#8-fold-fusion)
9. [Left Fold (loop / foldl)](#9-left-fold-loop--foldl)
10. [Scan](#10-scan)
11. [Proof by Induction](#11-proof-by-induction)
12. [Types and Trees](#12-types-and-trees)
13. [Rose Trees and Bush Trees](#13-rose-trees-and-bush-trees)
14. [Functor and fmap](#14-functor-and-fmap)
15. [Efficiency: Accumulator Pattern](#15-efficiency-accumulator-pattern)
16. [Sorting Algorithms as Folds and Unfolds](#16-sorting-algorithms-as-folds-and-unfolds)
17. [Dynamic Programming and Tabulation](#17-dynamic-programming-and-tabulation)
18. [Key Laws and Identities Cheatsheet](#18-key-laws-and-identities-cheatsheet)

---

## 1. Types and Function Application

Every Haskell function has a type. The notation `f :: X -> Y` means applying `f` to something of type `X` gives something of type `Y`.

```haskell
sin     :: Float -> Float
add     :: (Int, Int) -> Int
logBase :: Float -> (Float -> Float)   -- curried: logBase 10 :: Float -> Float
(.)     :: (b -> c) -> (a -> b) -> (a -> c)
```

**Function application** binds to the left: `f x y` means `(f x) y`.  
**Operators** (symbols) bind less tightly: `p q + r s` means `(+) (p q) (r s)`.

### Currying
Every multi-argument function is really a chain of single-argument functions.  
`logBase 10` is a perfectly good value of type `Float -> Float`.

### Composition
```haskell
(f . g) x = f (g x)
-- type: (.) :: (b -> c) -> (a -> b) -> (a -> c)
```
Composition is **associative**: `f . (g . h) = (f . g) . h`.  
The identity function `id x = x` satisfies `f . id = id . f = f`.

---

## 2. Lists and List Comprehensions

A list of type `[T]` is a sequence of elements of type `T`.

```haskell
[3, 1, 4, 1, 5]  :: [Int]
['a'..'z']        :: [Char]   -- = "abcdefghijklmnopqrstuvwxyz"
[1..10]           :: [Int]    -- [1,2,3,4,5,6,7,8,9,10]
[[1],[2,3],[4,5,6]] :: [[Int]]
```

### List Comprehensions
```haskell
map f xs    = [ f x | x <- xs ]
filter p xs = [ x   | x <- xs, p x ]
concat xss  = [ x   | xs <- xss, x <- xs ]
```
Parts after `|`: generators (`x <- xs`) introduce variables; boolean expressions are guards.

### Key List Functions
```haskell
null    :: [a] -> Bool
head    :: [a] -> a           -- partial: undefined on []
tail    :: [a] -> [a]         -- partial: undefined on []
last    :: [a] -> a
(++)    :: [a] -> [a] -> [a]  -- concatenation, cost proportional to length of left arg
length  :: [a] -> Int
reverse :: [a] -> [a]
take    :: Int -> [a] -> [a]
drop    :: Int -> [a] -> [a]
zip     :: [a] -> [b] -> [(a,b)]   -- stops at shorter list
zipWith :: (a -> b -> c) -> [a] -> [b] -> [c]
span    :: (a -> Bool) -> [a] -> ([a],[a])   -- = (takeWhile p xs, dropWhile p xs)
splitAt :: Int -> [a] -> ([a],[a])           -- = (take n xs, drop n xs)
group   :: Eq a => [a] -> [[a]]   -- maximal runs of equal elements
sort    :: Ord a => [a] -> [a]
```

---

## 3. Pattern Matching and Data Types

New types are introduced by `data` declarations. Functions are defined by **pattern matching on constructors**.

```haskell
data Bool  = False | True
data Maybe a = Nothing | Just a
data Either a b = Left a | Right b

not :: Bool -> Bool
not False = True
not True  = False

-- Either's deconstructor (= its fold, since Either is not recursive)
either :: (a -> c) -> (b -> c) -> Either a b -> c
either left right (Left x)  = left x
either left right (Right y) = right y
-- Note: either Left Right = id
```

### Recursive Data Types
```haskell
data List a = Nil | Cons a (List a)   -- isomorphic to [a]
data Nat    = Zero | Succ Nat
```

Functions on recursive types are naturally defined by recursion:
```haskell
f :: List a -> ...
f Nil        = ...
f (Cons x xs) = ... x ... (f xs) ...
```

### Record Syntax and newtype
```haskell
data Pair a = Pair { first :: a, second :: a }   -- generates selectors automatically
newtype Value a = Value a   -- strict wrapper; zero runtime cost
```

### Type Aliases
```haskell
type String = [Char]
type Word   = [Char]
```

---

## 4. Strictness, Laziness, and Bottom

**Bottom** (⊥) is the value of a non-terminating or erroring computation.

A function `f` is **strict** if `f ⊥ = ⊥`. Pattern matching is strict (it must inspect the value to decide which case applies).

**Haskell is lazy by default**: expressions are only evaluated when needed.

### Information Ordering
`x ⊑ y` means "`y` contains at least as much information as `x`".  
`⊥ ⊑ x` for all `x`. For lists: `(x:xs) ⊑ (y:ys)` iff `x ⊑ y` and `xs ⊑ ys`.

### Key strictness facts
- `(++)` is **strict in its left argument**: `⊥ ++ ys = ⊥`
- `(++)` is **not strict in its right argument**: `[1,2] ++ ⊥ = 1:2:⊥ ≠ ⊥`
- `map f` is not strict: `map f ⊥ = ⊥`, but `head (map f (1:⊥)) = f 1`
- `fold` is strict in the list argument
- Constructors (`:`, `(,)`, `Just`) are **never strict**

### Forcing evaluation (BangPatterns extension)
```haskell
-- Use ! to make an argument strict
loop' s (!n) []     = n
loop' s (!n) (x:xs) = loop' s (s n x) xs
```

---

## 5. Higher-Order Functions

### map
```haskell
map :: (a -> b) -> [a] -> [b]
map f []     = []
map f (x:xs) = f x : map f xs
-- Laws:
-- map id = id
-- map f . map g = map (f . g)
-- map f (xs ++ ys) = map f xs ++ map f ys
```

### filter
```haskell
filter :: (a -> Bool) -> [a] -> [a]
filter p []     = []
filter p (x:xs) | p x       = x : rest
                | otherwise = rest
  where rest = filter p xs
```

### flip and const
```haskell
flip   :: (a -> b -> c) -> (b -> a -> c)
flip f x y = f y x

const  :: a -> b -> a
const x _ = x
```

### uncurry and curry
```haskell
uncurry :: (a -> b -> c) -> (a,b) -> c
uncurry f (x,y) = f x y

curry :: ((a,b) -> c) -> a -> b -> c
curry f x y = f (x,y)
```

### iterate
```haskell
iterate :: (a -> a) -> a -> [a]
iterate f x = x : iterate f (f x)
-- = unfold (const False) id f x
```

---

## 6. Fold (foldr)

**The fundamental pattern of recursion on lists.**

### Definition
```haskell
fold :: (a -> b -> b) -> b -> [a] -> b
fold cons nil []     = nil
fold cons nil (x:xs) = cons x (fold cons nil xs)
-- = foldr in standard Haskell (fold = foldr :: (a->b->b) -> b -> [a] -> b)
```

`fold c n` replaces each `(:)` with `c` and `[]` with `n`:

```
x1 : x2 : x3 : []    -->    x1 `c` (x2 `c` (x3 `c` n))
```

**Note**: `fold (:) [] = id`

### Standard functions as folds
```haskell
sum     = fold (+) 0
product = fold (*) 1
concat  = fold (++) []
map f   = fold ((:) . f) []       -- or: fold (\x ys -> f x : ys) []
(++ bs) = fold (:) bs              -- append bs to the right
```

### How to find the fold for a function
Given `f xs = fold cons nil xs`, solve for `cons` and `nil`:
- `nil`: set `nil = f []` (use the definition of `fold` on `[]`)
- `cons`: given `cons x (fold cons nil xs) = f (x:xs)`, substitute `fold cons nil xs` by `f xs` (for a smaller argument) to find `cons x ys = ...`

**Example** — showing `map f = fold ((:) . f) []`:
```
nil  = map f [] = []
cons x ys = map f (x:xs)     where ys = map f xs (by assumption)
           = f x : map f xs  = f x : ys = ((:) . f) x ys
```

### foldBTree
```haskell
data BTree a = Leaf a | Fork (BTree a) (BTree a)

foldBTree :: (a -> b) -> (b -> b -> b) -> BTree a -> b
foldBTree leaf fork (Leaf x)   = leaf x
foldBTree leaf fork (Fork l r) = fork (foldBTree leaf fork l)
                                       (foldBTree leaf fork r)
-- size = foldBTree (const 1) (+)
-- flatten = foldBTree (:[]) (++)
```

---

## 7. Unfold

**The dual of fold.** While fold consumes a list by replacing constructors, unfold **produces** a list by repeatedly applying deconstructors.

### Definition
```haskell
unfold :: (b -> Bool) -> (b -> a) -> (b -> b) -> b -> [a]
unfold null head tail = rec
  where rec x | null x    = []
               | otherwise = head x : rec (tail x)
```
Parameters: `null` tests for termination, `head` extracts the next element, `tail` advances the state.

### Example
```haskell
unfold (==0) (`mod`10) (`div`10) 123456 = [6,5,4,3,2,1]
```

### Relationship to fold
`unfold null head tail . fold cons nil = id`  
provided `null nil = True`, `null (cons x y) = False`, `head (cons x y) = x`, `tail (cons x y) = y`.

### iterate as unfold
```haskell
iterate f = unfold (const False) id f
```

### take as unfold
```haskell
take = unfold (\(n,xs) -> n==0 || null xs)
               (\(n,xs) -> head xs)
               (\(n,xs) -> (n-1, tail xs))
```

### unfoldBTree
```haskell
unfoldBTree :: (b -> Bool) -> (b -> a) -> (b -> b) -> (b -> b) -> b -> BTree a
unfoldBTree single value left right = rec
  where rec x | single x  = Leaf (value x)
               | otherwise = Fork (rec (left x)) (rec (right x))

-- Build a balanced tree from a list:
build = unfoldBTree (null . tail) head
                    (\xs -> take (length xs `div` 2) xs)
                    (\xs -> drop (length xs `div` 2) xs)
```

---

## 8. Fold Fusion

**The most powerful law for folds.** Lets you replace a composition `f . fold g a` with a single fold, eliminating intermediate data structures.

### Statement
If `f` is strict, `b = f a`, and `h x . f = f . g x` (i.e. `h x (f y) = f (g x y)` for all `x`, `y`), then:

```
f . fold g a  =  fold h b
```

### Proof sketch (by induction)
- Base: `(f . fold g a) [] = f a = b = fold h b []` ✓ (since `b = f a`)
- Step: `(f . fold g a) (x:xs) = f (g x (fold g a xs))`
  - By IH: `= f (g x (fold g a xs))`
  - `= (h x . f) (fold g a xs)` (by the commutativity condition)
  - `= h x ((f . fold g a) xs)` 
  - `= h x (fold h b xs)` (IH)
  - `= fold h b (x:xs)` ✓

### Conditions summary
| Condition | Meaning |
|-----------|---------|
| `f` is strict | `f ⊥ = ⊥` |
| `b = f a` | the nil-case matches |
| `h x (f y) = f (g x y)` | the cons-case commutes |

### Proving f is a fold via fusion
Any strict function `f` satisfying `f (x:xs) = h x (f xs)` is a fold:
```
f = f . id = f . fold (:) [] = fold h (f [])
```
(using fusion with `g = (:)`, `a = []`)

### Example: map f = fold ((:).f) []
Using fusion on `map f . fold (:) []`:
- `b = map f [] = []`
- `h x ys = map f (x:xs)` where `ys = map f xs` = `f x : ys = ((:).f) x ys`

### Example: scan c n = fold h [n]
(See §10 below)

---

## 9. Left Fold (loop / foldl)

### Definition
**Right fold** (fold/foldr) is right-associative:
```
fold (⊕) e [x0, x1, ..., xn] = x0 ⊕ (x1 ⊕ (... ⊕ (xn ⊕ e)...))
```

**Left fold** (loop/foldl) is left-associative:
```
loop (⊕) e [x0, x1, ..., xn] = (...((e ⊕ x0) ⊕ x1) ⊕ ... ⊕ xn)
```

```haskell
loop :: (b -> a -> b) -> b -> [a] -> b
loop s n []     = n
loop s n (x:xs) = loop s (s n x) xs
-- = foldl in standard Haskell
```

Alternatively: `loop s n = fold (flip s) n . reverse`

### When fold = loop
`fold (⊕) e = loop (⊗) e`  
when `(⊕)` is right-strict, `e ⊗ x = x ⊕ e`, and there exists `(⊙)` with  
`a ⊗ (b ⊙ c) = (a ⊗ b) ⊙ c` and `(a ⊙ b) ⊕ c = a ⊙ (b ⊕ c)`.

The simple case: when `(⊕) = (⊗) = (⊙)` is **right-strict**, **associative**, and `e` is a **left unit**.
```haskell
sum     = fold (+) 0  = loop (+) 0    -- (+) is associative, 0 is left unit
product = fold (*) 1  = loop (*) 1
concat  = fold (++) [] ≠ loop (++) []  -- (++) is NOT associative: xs++⊥ = ⊥ but ⊥++xs = ⊥
```

### Space: strict accumulator (loop')
`loop` builds up a chain of unevaluated thunks. Use `loop'` (with `!`) to force evaluation:
```haskell
loop' s (!n) []     = n
loop' s (!n) (x:xs) = loop' s (s n x) xs
-- or use foldl' from Data.List
```

---

## 10. Scan

### Specification
```haskell
scan :: (a -> b -> b) -> b -> [a] -> [b]
scan c n = map (fold c n) . tails
-- where tails xs = all suffix segments of xs in decreasing length order
```

```haskell
tails :: [a] -> [[a]]
tails []     = [[]]
tails (x:xs) = (x:xs) : tails xs
```

So `scan c n [x,y,z] = [fold c n [x,y,z], fold c n [y,z], fold c n [z], fold c n []]`  
= `[x `c` (y `c` (z `c` n)), y `c` (z `c` n), z `c` n, n]`

**Naive implementation** is quadratic: ~½n² applications of `c`.

### Efficient implementation via fold fusion
Applying fusion to `(map (fold c n)) . tails`:

`tails = fold g [[]]` where `g x yss = (x : head yss) : yss`

Then fuse `map (fold c n)` with this fold:
```haskell
scan c n = fold h [n]
  where h x zs = c x (head zs) : zs
```
This is **linear**: n applications of `h`, each calling `c` once.

```haskell
-- In standard Haskell: scanr = scan
scan c n = fold h [n] where h x zs = c x (head zs) : zs
-- Equivalently: scan c n xs = fold h [n] xs
```

### Key property
`head (scan c n xs) = fold c n xs`

---

## 11. Proof by Induction

### Induction on natural numbers
To prove `P(n)` for all natural numbers `n`, prove:
1. `P(0)` (base case)
2. For all `n`, if `P(n)` then `P(n+1)` (inductive step)

### Induction on finite lists
To prove `P(xs)` for all **finite** lists `xs`:
1. `P([])` (base case)
2. For all `x` and finite `xs`, if `P(xs)` then `P(x:xs)` (step)

### Induction on partial lists
A **partial list** has a tail that is `⊥` or a smaller partial list.  
To prove `P(xs)` for all **partial** lists:
1. `P(⊥)`
2. For all partial `xs`, if `P(xs)` then `P(x:xs)`

### Infinite lists
Haskell's `[a]` includes infinite lists. To prove `P(xs)` for **all** lists (including infinite):
- Prove it for all partial lists, and
- Show `P` is **chain complete** (true for every element of an ascending chain → true for the limit).

Equations between Haskell-definable expressions are chain complete.  
**Conjunction** ('and') of chain-complete properties is chain complete.  
**Disjunction** ('or') and existential quantification are NOT guaranteed chain complete.

### The Take Lemma
`xs = ys` if and only if `take n xs = take n ys` for all natural numbers `n`.

Useful for proving equalities involving infinite lists: reduce to proving `take (n+1) LHS = take (n+1) RHS` by induction on `n`.

### Equational reasoning format
```
  expression1
= { justification }
  expression2
= { justification }
  expression3
```

---

## 12. Types and Trees

### Algebraic data types
Every `data` declaration introduces:
- A **type constructor** (e.g. `Either`)
- **Data constructors** (e.g. `Left`, `Right`) — always invertible (injective)
- **Deconstructors/selectors** (e.g. `left`, `right`) — partial functions

```haskell
data Either a b = Left a | Right b
-- Deconstructors:
left  (Left x)  = x
right (Right y) = y
-- Discriminator:
isLeft (Left _)  = True
isLeft (Right _) = False
-- Single fold-like deconstructor: either (defined above)
```

### Maybe
```haskell
data Maybe a = Nothing | Just a

maybe :: b -> (a -> b) -> Maybe a -> b
maybe nothing just Nothing  = nothing
maybe nothing just (Just x) = just x
```

### Polynomial types (sums of products)
Types built from `data` are **sums** (alternatives `|`) of **products** (tuples of constructors).  
Constructors are **never strict**: `Pair ⊥ ⊥ ≠ ⊥`.  
Pattern matching on constructors **is** strict.

### newtype vs data
```haskell
newtype Value a = Value a   -- strict constructor, zero runtime cost, erased at compile time
data    Box   a = Box a     -- lazy constructor (Box ⊥ ≠ ⊥)
```

### Binary trees
```haskell
data BTree a = Leaf a | Fork (BTree a) (BTree a)

foldBTree :: (a -> b) -> (b -> b -> b) -> BTree a -> b
foldBTree leaf fork (Leaf x)   = leaf x
foldBTree leaf fork (Fork l r) = fork (foldBTree leaf fork l)
                                       (foldBTree leaf fork r)

flatten :: BTree a -> [a]
flatten = foldBTree (:[]) (++)

size :: BTree a -> Int
size = foldBTree (const 1) (+)
```

Efficient flatten (linear, avoiding quadratic `++`):
```haskell
flatcat :: BTree a -> [a] -> [a]
flatcat t ys = flatten t ++ ys     -- specification
-- Derive by calculation:
flatcat (Leaf x)   ys = x : ys
flatcat (Fork l r) ys = flatcat l (flatcat r ys)
-- So: flatcat = foldBTree (:) (flip (.))
-- And: flatten t = flatcat t []
```

---

## 13. Rose Trees and Bush Trees

### Rose Tree
A rose tree has nodes with **any number of children** (a list of subtrees).

```haskell
data RTree a = RTree a [RTree a]
-- A node is a value paired with a (possibly empty) list of subtrees.
-- RTree is never empty.
```

**fmap on RTree** (making it a Functor):
```haskell
instance Functor RTree where
  fmap f (RTree a ts) = RTree (f a) (map (fmap f) ts)
```

**Fold on RTree**:
```haskell
foldRTree :: (a -> [b] -> b) -> RTree a -> b
foldRTree node (RTree x ts) = node x (map (foldRTree node) ts)
```

**Use case**: Tries (prefix trees) are a special form of rose tree used for efficient tabulation:
```haskell
data Trie a b = Trie b (Mapping a (Trie a b))
type Copse a b = Mapping a (Trie a b)
```

### Bush Tree
A generalisation parameterised by its child container:
```haskell
data Bush t a = Bush a (t (Bush t a))
-- Bush [] a  ≅  RTree a   (children in a list)
-- Bush Maybe a  = non-empty list type
```

```haskell
instance Functor t => Functor (Bush t) where
  fmap f (Bush x ts) = Bush (f x) (fmap (fmap f) ts)

foldBush :: Functor t => (a -> t b -> b) -> Bush t a -> b
foldBush bush (Bush x ts) = bush x (fmap (foldBush bush) ts)
```

---

## 14. Functor and fmap

A **Functor** is a type constructor `f` with a mapping operation that respects identity and composition.

```haskell
class Functor f where
  fmap :: (a -> b) -> f a -> f b
-- Laws (not enforced by Haskell, but required):
-- fmap id = id
-- fmap f . fmap g = fmap (f . g)
```

### Standard instances
```haskell
instance Functor [] where
  fmap = map

instance Functor Maybe where
  fmap f Nothing  = Nothing
  fmap f (Just x) = Just (f x)

instance Functor RTree where
  fmap f (RTree a ts) = RTree (f a) (map (fmap f) ts)
```

### map as a special case
`map` is `fmap` for lists. For any recursive type, once the fold is defined, `fmap` can be derived from it:
```haskell
-- For lists:
map f = fold ((:) . f) []
-- For BTree:
fmap f = foldBTree (Leaf . f) Fork
```

### The general pattern
For any functor `t`, `fmap (foldBush bush) = fmap` applied to the children, which is why the fold and the functor are interleaved in `foldBush` and `foldRTree`.

---

## 15. Efficiency: Accumulator Pattern

### Quadratic reverse (naive)
```haskell
reverse []     = []
reverse (x:xs) = reverse xs ++ [x]   -- O(n²): each (++) is O(length left arg)
```

### Linear reverse via accumulator
Introduce `revcat ys xs = reverse xs ++ ys` as a specification, then derive:
```haskell
revcat ys []     = ys
revcat ys (x:xs) = revcat (x:ys) xs
-- So:
reverse xs = revcat [] xs
-- Equivalently:
reverse = loop (flip (:)) []
```

### Quadratic tree flatten (naive)
`flatten (Fork ls rs) = flatten ls ++ flatten rs`  
A left-skewed tree has O(n log n) or O(n²) cost from repeated `(++)`.

### Linear flatten via accumulator
```haskell
flatcat :: BTree a -> [a] -> [a]     -- spec: flatcat t ys = flatten t ++ ys
flatcat (Leaf x)   ys = x : ys
flatcat (Fork l r) ys = flatcat l (flatcat r ys)
-- Then: flatten t = flatcat t []
```

### Fast exponentiation
```haskell
-- Naive: n multiplications
pow x n = if n == 0 then 1 else pow x (n-1) * x

-- O(log n) with accumulator:
power (*) y x 0        = y
power (*) y x n | even n = power (*) y (x*x) (n `div` 2)
                | odd  n = power (*) (x*y) x (n-1)
-- Then: pow x n = power (*) 1 x n
```
Works for any associative operation (e.g. matrix multiplication).

---

## 16. Sorting Algorithms as Folds and Unfolds

### Insertion sort (fold)
```haskell
isort :: Ord a => [a] -> [a]
isort = fold insert []
  where
    insert x []     = [x]
    insert x (y:ys) | y >= x    = x : y : ys
                    | otherwise = y : insert x ys
```

### Selection sort (unfold)
```haskell
ssort :: Ord a => [a] -> [a]
ssort = unfold null minimum deleteMin
  where deleteMin xs = delete (minimum xs) xs
```
`ssort` is an unfold: it repeatedly selects the minimum and removes it.

### Quicksort (unfold then fold)
```haskell
qsort :: Ord a => [a] -> [a]
qsort []     = []
qsort (x:xs) = qsort [y | y <- xs, y < x] ++
               [y | y <- xs, y == x] ++  -- or x: if no duplicates
               qsort [y | y <- xs, y > x]
-- Can be expressed as: qsort = flatten . build
-- where build :: Ord a => [a] -> QTree a   (an unfold to a QTree)
-- and   flatten :: QTree a -> [a]           (a fold from a QTree)
```

### Merge sort (unfold then fold)
```haskell
msort :: Ord a => [a] -> [a]
msort []  = []
msort [x] = [x]
msort xs  = merge (msort ls) (msort rs) where (ls, rs) = halve xs

halve xs  = splitAt (length xs `div` 2) xs

merge :: Ord a => [a] -> [a] -> [a]
merge (x:xs) (y:ys) | x <= y    = x : merge xs (y:ys)
                    | otherwise = y : merge (x:xs) ys
merge []     ys     = ys
merge xs     []     = xs
-- msort = flatten . build
-- where build :: [a] -> MTree a   (an unfold)
-- and   flatten :: Ord a => MTree a -> [a]   (a fold using merge)
```

---

## 17. Dynamic Programming and Tabulation

### Fixed points
For a recursive function `f`, extract the **kernel** `fK` (the non-recursive body):
```haskell
fib 0 = 0
fib 1 = 1
fib n = fib (n-1) + fib (n-2)

fibK :: (Int -> Integer) -> (Int -> Integer)
fibK f 0 = 0
fibK f 1 = 1
fibK f n = f (n-1) + f (n-2)
-- fib = fibK fib  ==>  fib is a fixed point of fibK
```

### Tabulate
Replace recursive calls with table lookups:
```haskell
tabulate :: ((Int -> a) -> (Int -> a)) -> (Int -> a)
tabulate kernel = fun
  where fun = (tab !!)
        tab = map (kernel fun) [0..]
-- tabulate fibK = fib, but computing each value only once
```
Uses Haskell's lazy evaluation: `tab` is an infinite list built once; entries are computed on demand and shared.

### Association list tabulation
For functions over lists (e.g. Countdown `results`):
```haskell
data Mapping a b = Mapping [(a,b)]

toMapping :: [a] -> (a -> b) -> Mapping a b
toMapping xs f = Mapping [(x, f x) | x <- xs]

getMapping :: Eq a => Mapping a b -> a -> b
getMapping (Mapping m) x = head [b | (a,b) <- m, a == x]
```

### Trie tabulation
For prefix-closed domains (subsequences), a **Trie** (rose tree structure) is more efficient than an association list:
```haskell
data Trie a b = Trie b (Mapping a (Trie a b))

getTrie :: Eq a => Trie a b -> [a] -> b
getTrie (Trie y _) []     = y
getTrie (Trie _ m) (x:xs) = m `getMapping` x `getTrie` xs
```
Lookup is O(length of key) instead of O(size of table).

---

## 18. Key Laws and Identities Cheatsheet

### Fold laws
```
fold (:) []            = id
fold c n (xs ++ ys)   = fold c n xs `op` fold c n ys
  (when x `op` (y `op` z) = (x 'c' y) `op` z and n `op` x = x)
map f                  = fold ((:) . f) []
sum                    = fold (+) 0
product                = fold (*) 1
concat                 = fold (++) []
(++ ys)                = fold (:) ys
```

### Fusion law
```
f strict, b = f a, h x . f = f . g x
⟹  f . fold g a  =  fold h b
```

### Scan theorem
```
head (scan c n xs) = fold c n xs
scan c n           = fold h [n]   where h x zs = c x (head zs) : zs
```

### fold vs loop
```
fold (+) 0  = loop (+) 0        (+ associative, right-strict, 0 is left unit)
fold (++) [] ≠ loop (++) []     (++ not associative across ⊥)
reverse      = loop (flip (:)) []
```

### Map laws
```
map id     = id
map f . map g  = map (f . g)
map f (xs ++ ys) = map f xs ++ map f ys
fmap id    = id                          (Functor law)
fmap f . fmap g = fmap (f . g)           (Functor law)
```

### Induction schemes summary
| To prove P(xs) for... | Base case | Step |
|-----------------------|-----------|------|
| all finite lists | P([]) | P(xs) ⟹ P(x:xs) |
| all partial lists | P(⊥) | P(xs) ⟹ P(x:xs) |
| all lists (incl. infinite) | P(⊥) + chain completeness | P(xs) ⟹ P(x:xs) |
| all natural numbers | P(0) | P(n) ⟹ P(n+1) |

### Unfold–fold identity
```
unfold p h t . fold c n = id
  provided: p n = True, p (c x y) = False, h (c x y) = x, t (c x y) = y
```

### Efficiency patterns
| Problem | Naive | Efficient | Trick |
|---------|-------|-----------|-------|
| `reverse` | O(n²) | O(n) | accumulator `revcat`: `loop (flip (:)) []` |
| `flatten` BTree | O(n log n) | O(n) | accumulator `flatcat`: `foldBTree (:) (flip (.))` |
| `fib n` | O(2ⁿ) | O(n) | tabulation |
| `x^n` | O(n) | O(log n) | fast exponentiation with accumulator |

---

*Course: Functional Programming MT2023, Geraint Jones, University of Oxford*  
*Primary textbook: "Thinking Functionally with Haskell" (Bird)*
