# Functional Programming — MT2023 Concept Reference
*Geraint Jones, University of Oxford. Notes compiled from all 16 lectures.*

---

## Table of Contents
1. [Types and Function Application](#1-types-and-function-application)
2. [Lists and List Comprehensions](#2-lists-and-list-comprehensions)
3. [Standard List Functions in Detail](#3-standard-list-functions-in-detail)
4. [Pattern Matching and Data Types](#4-pattern-matching-and-data-types)
5. [Maybe — Safe Optional Values](#5-maybe--safe-optional-values)
6. [Either — Error Handling and Choice](#6-either--error-handling-and-choice)
7. [Strictness, Laziness, and Bottom](#7-strictness-laziness-and-bottom)
8. [Higher-Order Functions](#8-higher-order-functions)
9. [Fold (foldr) — Every Application](#9-fold-foldr--every-application)
10. [Unfold — Every Application](#10-unfold--every-application)
11. [Fold Fusion](#11-fold-fusion)
12. [Left Fold (loop / foldl)](#12-left-fold-loop--foldl)
13. [Scan](#13-scan)
14. [Proof by Induction](#14-proof-by-induction)
15. [Types and Trees](#15-types-and-trees)
16. [Rose Trees and Bush Trees](#16-rose-trees-and-bush-trees)
17. [Functor and fmap](#17-functor-and-fmap)
18. [Efficiency: Accumulator Pattern](#18-efficiency-accumulator-pattern)
19. [Sorting Algorithms — Full Detail](#19-sorting-algorithms--full-detail)
20. [Sudoku: Standard Functions in Practice](#20-sudoku-standard-functions-in-practice)
21. [Dynamic Programming and Tabulation](#21-dynamic-programming-and-tabulation)
22. [Key Laws and Identities Cheatsheet](#22-key-laws-and-identities-cheatsheet)

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
Partial application is free: `map (2*)` applies `(2*)` to every element.

### Composition
```haskell
(f . g) x = f (g x)
-- type: (.) :: (b -> c) -> (a -> b) -> (a -> c)
```
Composition is **associative**: `f . (g . h) = (f . g) . h`.  
The identity function `id x = x` satisfies `f . id = id . f = f`.

### Sections
An infix operator applied to one argument is a **section**:
```haskell
(2*)    -- = \x -> 2 * x
(+3)    -- = \x -> x + 3
(`div` 10) -- = \x -> x `div` 10
(10 `div`) -- = \x -> 10 `div` x
```

---

## 2. Lists and List Comprehensions

A list of type `[T]` is a sequence of elements of type `T`.

```haskell
[3, 1, 4, 1, 5]  :: [Int]
['a'..'z']        :: [Char]   -- = "abcdefghijklmnopqrstuvwxyz"
[1..10]           :: [Int]    -- [1,2,3,4,5,6,7,8,9,10]
[2,4..12]         :: [Int]    -- [2,4,6,8,10,12]
[[1],[2,3],[4,5,6]] :: [[Int]]
```

### List Comprehensions
```haskell
map f xs    = [ f x | x <- xs ]
filter p xs = [ x   | x <- xs, p x ]
concat xss  = [ x   | xs <- xss, x <- xs ]

-- Multiple generators = nested loops:
pairs = [ (x,y) | x <- [1..3], y <- [1..3] ]
-- = [(1,1),(1,2),(1,3),(2,1),(2,2),(2,3),(3,1),(3,2),(3,3)]

-- Guard filters combinations:
pythagorean = [ (a,b,c) | c <- [1..100], b <- [1..c], a <- [1..b],
                           a^2 + b^2 == c^2 ]
```

Parts after `|`: generators (`x <- xs`) introduce variables; boolean guards filter.

### Operators on Lists
```haskell
(:)   :: a -> [a] -> [a]       -- cons: prepend element
(++)  :: [a] -> [a] -> [a]     -- append: cost = O(length left arg)
(!!)  :: [a] -> Int -> a       -- index: xs !! 0 = head xs
```

---

## 3. Standard List Functions in Detail

### map
```haskell
map :: (a -> b) -> [a] -> [b]
map f []     = []
map f (x:xs) = f x : map f xs
```

**Examples:**
```haskell
map (*2)      [1,2,3]     = [2,4,6]
map show      [1,2,3]     = ["1","2","3"]
map null      [[],[1],[]]  = [True,False,True]
map (map (+1)) [[1,2],[3]] = [[2,3],[4]]      -- map of map
map fst       [(1,'a'),(2,'b')] = [1,2]
map reverse   ["abc","de"] = ["cba","ed"]
```

**As a fold:**
```haskell
map f = fold ((:) . f) []
-- i.e. fold (\x ys -> f x : ys) []
```

**Laws:**
```
map id      = id
map f . map g   = map (f . g)       -- fusion
map f (xs ++ ys) = map f xs ++ map f ys
```

**Parametricity ("theorem for free"):**  
For any `f :: a -> b` and any polymorphic function `fun :: [a] -> [a]`:
```
map f . fun  =  fun . map f
```
This holds for `reverse`, `tail`, `init`, `tails`, `inits`, etc.

---

### filter
```haskell
filter :: (a -> Bool) -> [a] -> [a]
filter p []     = []
filter p (x:xs) | p x       = x : filter p xs
                | otherwise = filter p xs
```

**Examples:**
```haskell
filter even    [1..10]      = [2,4,6,8,10]
filter (>3)    [1..5]       = [4,5]
filter null    [[],[1],[]]  = [[]]
filter isUpper "Hello World" = "HW"
filter (not . null) [[],[1],[2,3]] = [[1],[2,3]]
```

**As a fold:**
```haskell
filter p = fold (\x ys -> if p x then x:ys else ys) []
```

**Laws:**
```
filter p . filter q = filter (\x -> p x && q x)
filter p . map f    = map f . filter (p . f)   -- only if injective
length (filter p xs) + length (filter (not . p) xs) = length xs
```

**With list comprehension:**
```haskell
filter p xs = [x | x <- xs, p x]
```

---

### concat
```haskell
concat :: [[a]] -> [a]
concat []       = []
concat (xs:xss) = xs ++ concat xss
```

**Examples:**
```haskell
concat [[1,2],[3],[4,5,6]] = [1,2,3,4,5,6]
concat ["Hello"," ","World"] = "Hello World"
concat (map (\x -> [x,x]) [1,2,3]) = [1,1,2,2,3,3]
concat [[]]    = []
concat []      = []
```

**As a fold:**
```haskell
concat = fold (++) []
```

**concatMap** (= concat . map):
```haskell
concatMap f = concat . map f
           = fold ((\x ys -> f x ++ ys)) []
-- Example:
concatMap (\x -> [x, -x]) [1,2,3] = [1,-1,2,-2,3,-3]
```

---

### take and drop
```haskell
take :: Int -> [a] -> [a]
take 0 _      = []
take _ []     = []
take n (x:xs) = x : take (n-1) xs

drop :: Int -> [a] -> [a]
drop 0 xs     = xs
drop _ []     = []
drop n (_:xs) = drop (n-1) xs
```

**Examples:**
```haskell
take 3 [1..10]        = [1,2,3]
take 0 [1..10]        = []
take 5 [1,2]          = [1,2]       -- stops at end of list
take 3 (repeat 'x')   = "xxx"       -- works on infinite lists!
drop 3 [1..6]         = [4,5,6]
drop 0 [1,2,3]        = [1,2,3]
drop 10 [1,2,3]       = []
```

**Together:**
```haskell
take n xs ++ drop n xs = xs    -- fundamental identity
splitAt n xs = (take n xs, drop n xs)
```

**take as unfold:**
```haskell
take = unfold (\(n,xs) -> n==0 || null xs)
               (\(n,xs) -> head xs)
               (\(n,xs) -> (n-1, tail xs))
```

---

### takeWhile and dropWhile
```haskell
takeWhile :: (a -> Bool) -> [a] -> [a]
takeWhile p []     = []
takeWhile p (x:xs) | p x       = x : takeWhile p xs
                   | otherwise = []

dropWhile :: (a -> Bool) -> [a] -> [a]
dropWhile p []     = []
dropWhile p (x:xs) | p x       = dropWhile p xs
                   | otherwise = x:xs
```

**Examples:**
```haskell
takeWhile (<4)  [1..7]       = [1,2,3]
takeWhile even  [2,4,6,1,8]  = [2,4,6]    -- stops at first odd
takeWhile (const True) [1..] = [1,2,3,...]  -- infinite!
takeWhile null  [[],[],[1]]  = [[],[]]

dropWhile (<4)  [1..7]       = [4,5,6,7]
dropWhile even  [2,4,5,6]    = [5,6]
dropWhile (>0)  [-1,2,3]     = [-1,2,3]   -- predicate fails immediately
```

**Relationship:**
```haskell
takeWhile p xs ++ dropWhile p xs = xs
span p xs = (takeWhile p xs, dropWhile p xs)
```

---

### span and splitAt
```haskell
span :: (a -> Bool) -> [a] -> ([a],[a])
span p xs = (takeWhile p xs, dropWhile p xs)

splitAt :: Int -> [a] -> ([a],[a])
splitAt n xs = (take n xs, drop n xs)
```

**Examples:**
```haskell
span (<4)     [1,2,3,4,5] = ([1,2,3],[4,5])
span even     [2,4,1,6]   = ([2,4],[1,6])
span (const False) [1,2]  = ([],[1,2])
span (const True)  [1,2]  = ([1,2],[])

splitAt 2 [1..5]  = ([1,2],[3,4,5])
splitAt 0 [1..5]  = ([],[1,2,3,4,5])
splitAt 9 [1..5]  = ([1,2,3,4,5],[])
```

**Use in algorithms:**
```haskell
-- Quicksort uses span-like partition:
partition p xs = (filter p xs, filter (not . p) xs)

-- Sudoku's 'by' uses splitAt to chunk a list:
by :: Int -> [a] -> [[a]]
by n [] = []
by n xs = take n xs : by n (drop n xs)
-- = unfold null (take n) (drop n)
```

---

### zip and zipWith
```haskell
zip :: [a] -> [b] -> [(a,b)]
zip (x:xs) (y:ys) = (x,y) : zip xs ys
zip _      _      = []

zipWith :: (a -> b -> c) -> [a] -> [b] -> [c]
zipWith f (x:xs) (y:ys) = f x y : zipWith f xs ys
zipWith _ _      _      = []

-- zip = zipWith (,)
```

**Examples:**
```haskell
zip [1,2,3] "abc"         = [(1,'a'),(2,'b'),(3,'c')]
zip [1..] "hello"         = [(1,'h'),(2,'e'),(3,'l'),(4,'l'),(5,'o')]
zip [1,2] [10,20,30]      = [(1,10),(2,20)]    -- stops at shorter

zipWith (+)  [1,2,3] [10,20,30] = [11,22,33]
zipWith (*)  [1..5]  [1..5]     = [1,4,9,16,25]
zipWith (:)  [1,2,3] [[],[],[]] = [[1],[2],[3]]
zipWith max  [1,5,3] [4,2,6]   = [4,5,6]
```

**Matrix transpose via zipWith:**
```haskell
-- Transpose: turn rows into columns
transpose :: [[a]] -> [[a]]
transpose []       = []
transpose (xs:xss) = zipWith (:) xs (transpose xss)
-- or simply: transpose = foldr (zipWith (:)) (repeat [])

-- Example:
transpose [[1,2,3],[4,5,6]] = [[1,4],[2,5],[3,6]]
```
This is exactly how `cols` works in the Sudoku solver.

**unzip:**
```haskell
unzip :: [(a,b)] -> ([a],[b])
unzip = foldr (\(x,y) (xs,ys) -> (x:xs, y:ys)) ([],[])
```

---

### replicate, repeat, iterate, cycle
```haskell
replicate :: Int -> a -> [a]
replicate n x = take n (repeat x)

repeat :: a -> [a]
repeat x = x : repeat x   -- infinite list

iterate :: (a -> a) -> a -> [a]
iterate f x = x : iterate f (f x)   -- x, f x, f(f x), ...

cycle :: [a] -> [a]
cycle xs = xs ++ cycle xs   -- infinite repetition of xs
```

**Examples:**
```haskell
replicate 3 'a'      = "aaa"
replicate 0 True     = []
take 5 (repeat 0)    = [0,0,0,0,0]
take 6 (iterate (*2) 1) = [1,2,4,8,16,32]
take 5 (iterate (+3) 0) = [0,3,6,9,12]
take 7 (cycle [1,2,3])  = [1,2,3,1,2,3,1]
```

**iterate as unfold:**
```haskell
iterate f = unfold (const False) id f
```

---

### group and sort
```haskell
group :: Eq a => [a] -> [[a]]
-- maximal runs of equal adjacent elements
group []     = []
group (x:xs) = let (same, rest) = span (== x) xs
               in (x:same) : group rest

sort :: Ord a => [a] -> [a]   -- from Data.List
```

**Examples:**
```haskell
group [1,1,2,3,3,3,1] = [[1,1],[2],[3,3,3],[1]]
group "Mississippi"   = ["M","i","ss","i","ss","i","pp","i"]
group [1,2,3]         = [[1],[2],[3]]

sort [3,1,4,1,5,9] = [1,1,3,4,5,9]
sort "functional"  = "aacfilnnotu"
```

**Frequency count using group . sort:**
```haskell
frequencies :: Ord a => [a] -> [(a,Int)]
frequencies = map (\xs -> (head xs, length xs)) . group . sort

-- most common word example from lecture 01:
mostCommon :: Ord a => [a] -> a
mostCommon = head . last . sortBy (comparing length) . group . sort
```

---

### head, tail, last, init, (!!)
```haskell
head :: [a] -> a      -- partial! undefined on []
tail :: [a] -> [a]    -- partial! undefined on []
last :: [a] -> a      -- partial! undefined on []
init :: [a] -> [a]    -- partial! undefined on []
(!!) :: [a] -> Int -> a  -- partial! undefined if index out of range
```

**Safe alternatives via Maybe:**
```haskell
safeHead :: [a] -> Maybe a
safeHead []    = Nothing
safeHead (x:_) = Just x

safeTail :: [a] -> Maybe [a]
safeTail []     = Nothing
safeTail (_:xs) = Just xs

safeIndex :: [a] -> Int -> Maybe a
safeIndex []     _ = Nothing
safeIndex (x:_)  0 = Just x
safeIndex (_:xs) n = safeIndex xs (n-1)
```

---

### null, length, reverse, sum, product
```haskell
null   :: [a] -> Bool
null []    = True
null (_:_) = False

length :: [a] -> Int
length = fold (const (1+)) 0   -- = fold (\_ n -> n+1) 0

reverse :: [a] -> [a]
reverse = fold (\x ys -> ys ++ [x]) []  -- naive O(n²)
-- Efficient: loop (flip (:)) []         -- O(n)

sum     = fold (+) 0
product = fold (*) 1
```

---

## 4. Pattern Matching and Data Types

New types are introduced by `data` declarations. Functions are defined by **pattern matching on constructors**.

```haskell
data Bool  = False | True
data Maybe a = Nothing | Just a
data Either a b = Left a | Right b

not :: Bool -> Bool
not False = True
not True  = False
```

### Recursive Data Types
```haskell
data List a = Nil | Cons a (List a)   -- isomorphic to [a]
data Nat    = Zero | Succ Nat
```

Functions on recursive types are naturally defined by recursion:
```haskell
f :: [a] -> ...
f []     = ...          -- base case
f (x:xs) = ... x ... (f xs) ...   -- recursive case
```

### Guards, where, let
```haskell
abs x | x < 0    = -x
      | otherwise = x

-- where: local definitions, in scope for all guards
classify x | x < 0    = "negative"
           | x > 0    = "positive"
           | otherwise = "zero"
  where -- no definitions here in this example

-- where vs let:
f x = let y = x + 1    -- let is an expression
      in y * y

g x = y * y            -- where is a declaration
  where y = x + 1
```

### Offside Rule
In Haskell, layout matters. Definitions at the same indentation level are in the same block.

```haskell
-- These two are equivalent:
f x = g x             f x = g x
  where g y = y + 1   where { g y = y + 1 }
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
type Matrix a = [[a]]
```

---

## 5. Maybe — Safe Optional Values

`Maybe a` represents a computation that may fail to return a value.

```haskell
data Maybe a = Nothing | Just a

-- The deconstructor (= fold for Maybe):
maybe :: b -> (a -> b) -> Maybe a -> b
maybe nothing just Nothing  = nothing
maybe nothing just (Just x) = just x
```

### Constructors and Pattern Matching
```haskell
safeHead :: [a] -> Maybe a
safeHead []    = Nothing
safeHead (x:_) = Just x

safeLookup :: Eq k => k -> [(k,v)] -> Maybe v
safeLookup _ []         = Nothing
safeLookup k ((k',v):rest)
  | k == k'   = Just v
  | otherwise = safeLookup k rest

safeDiv :: Int -> Int -> Maybe Int
safeDiv _ 0 = Nothing
safeDiv x y = Just (x `div` y)
```

### Using `maybe` to extract and transform
```haskell
-- maybe default f mx  =  if mx == Nothing then default else f (fromJust mx)

fromMaybe :: a -> Maybe a -> a
fromMaybe def = maybe def id

withDefault :: Int -> Maybe Int -> Int
withDefault d = maybe d (*2)   -- double if Just, otherwise use default

-- Examples:
maybe 0    (*2)   Nothing   = 0
maybe 0    (*2)   (Just 5)  = 10
maybe []   (:[]) (Just 'a') = "a"
fromMaybe  (-1)  Nothing    = -1
fromMaybe  (-1)  (Just 42)  = 42
```

### Chaining Maybe computations
```haskell
-- Manual chaining with pattern matching:
safeSqrt :: Double -> Maybe Double
safeSqrt x | x < 0    = Nothing
            | otherwise = Just (sqrt x)

safeLog :: Double -> Maybe Double
safeLog x | x <= 0   = Nothing
          | otherwise = Just (log x)

-- Chain: compute log of sqrt
logSqrt :: Double -> Maybe Double
logSqrt x = case safeSqrt x of
              Nothing -> Nothing
              Just y  -> safeLog y

-- Or use >>= (bind) from Monad:
logSqrt' x = safeSqrt x >>= safeLog
```

### Maybe as a Functor
```haskell
instance Functor Maybe where
  fmap f Nothing  = Nothing
  fmap f (Just x) = Just (f x)

-- Applying a function only if the value exists:
fmap (*2) Nothing  = Nothing
fmap (*2) (Just 5) = Just 10
fmap show (Just 42) = Just "42"
```

### Selectors and Discriminators
```haskell
isNothing :: Maybe a -> Bool
isNothing Nothing = True
isNothing _       = False

isJust :: Maybe a -> Bool
isJust = not . isNothing

fromJust :: Maybe a -> a      -- partial! crashes on Nothing
fromJust (Just x) = x
```

### Common patterns
```haskell
-- Safe first element of each list:
map safeHead [[1,2],[],[3]] = [Just 1, Nothing, Just 3]

-- Filter out Nothing values:
catMaybes :: [Maybe a] -> [a]
catMaybes = concat . map (maybe [] (:[]))
-- catMaybes [Just 1, Nothing, Just 3] = [1,3]

-- Apply function that may fail to each element:
mapMaybe :: (a -> Maybe b) -> [a] -> [b]
mapMaybe f = catMaybes . map f
```

---

## 6. Either — Error Handling and Choice

`Either a b` represents two possibilities: a `Left a` (typically an error) or a `Right b` (a success value).

```haskell
data Either a b = Left a | Right b

-- The deconstructor (= fold for Either):
either :: (a -> c) -> (b -> c) -> Either a b -> c
either left right (Left x)  = left x
either left right (Right y) = right y
-- Note: either Left Right = id   (fundamental identity)
```

### Left = error / failure, Right = success
```haskell
safeDiv :: Int -> Int -> Either String Int
safeDiv _ 0 = Left "Division by zero"
safeDiv x y = Right (x `div` y)

safeSqrt :: Double -> Either String Double
safeSqrt x
  | x < 0    = Left ("Cannot take sqrt of " ++ show x)
  | otherwise = Right (sqrt x)
```

### Using `either` to process results
```haskell
-- either f g (Left x)  = f x
-- either f g (Right y) = g y

either show (*2)  (Left "err")  = "err"    -- applies show to error
either show (*2)  (Right 5)     = 10       -- applies (*2) to value

-- Process a list of results:
results :: [Either String Int]
results = [Right 1, Left "bad", Right 3]

-- Extract only successes:
rights :: [Either a b] -> [b]
rights = concat . map (either (const []) (:[]))

-- Extract only errors:
lefts :: [Either a b] -> [a]
lefts = concat . map (either (:[]) (const []))
```

### Either as a Functor
```haskell
instance Functor (Either a) where
  fmap f (Left x)  = Left x     -- errors pass through unchanged
  fmap f (Right y) = Right (f y)

fmap (*2) (Left "oops") = Left "oops"
fmap (*2) (Right 5)     = Right 10
```

### Left and Right as injections
```haskell
-- Left :: a -> Either a b    (inject into left)
-- Right :: b -> Either a b   (inject into right)

-- Tagging elements with their origin:
tagLeft  xs = map Left  xs   -- :: [a] -> [Either a b]
tagRight ys = map Right ys   -- :: [b] -> [Either a b]

interleaveTagged :: [a] -> [b] -> [Either a b]
interleaveTagged xs ys = map Left xs ++ map Right ys
```

### Selectors and Discriminators
```haskell
isLeft :: Either a b -> Bool
isLeft (Left _) = True
isLeft _        = False

isRight :: Either a b -> Bool
isRight = not . isLeft

fromLeft :: Either a b -> a    -- partial
fromLeft (Left x) = x

fromRight :: Either a b -> b   -- partial
fromRight (Right y) = y
```

### Either vs Maybe
| Concept | `Maybe a` | `Either e a` |
|---------|-----------|--------------|
| Success | `Just a` | `Right a` |
| Failure | `Nothing` | `Left e` |
| Error info | None | Yes (`e`) |
| Deconstructor | `maybe def f` | `either errF okF` |

---

## 7. Strictness, Laziness, and Bottom

**Bottom** (⊥) is the value of a non-terminating or erroring computation.

A function `f` is **strict** if `f ⊥ = ⊥`. Pattern matching is strict (it must inspect the value to decide which case applies).

**Haskell is lazy by default**: expressions are only evaluated when needed.

### Information Ordering
`x ⊑ y` means "`y` contains at least as much information as `x`".  
`⊥ ⊑ x` for all `x`. For lists: `(x:xs) ⊑ (y:ys)` iff `x ⊑ y` and `xs ⊑ ys`.

### Key strictness facts
- `(++)` is **strict in its left argument**: `⊥ ++ ys = ⊥`
- `(++)` is **not strict in its right argument**: `[1,2] ++ ⊥ = 1:2:⊥ ≠ ⊥`
- `map f` is not strict in elements: `head (map f (1:⊥)) = f 1`
- `fold` is strict in the list argument (needs to know if list is `[]` or `x:xs`)
- Constructors (`:`, `(,)`, `Just`) are **never strict**: `Just ⊥ ≠ ⊥`

### Forcing evaluation (BangPatterns extension)
```haskell
-- Use ! to make an argument strict
loop' s (!n) []     = n
loop' s (!n) (x:xs) = loop' s (s n x) xs
```

---

## 8. Higher-Order Functions

### flip and const
```haskell
flip   :: (a -> b -> c) -> (b -> a -> c)
flip f x y = f y x
-- flip (:) = \xs x -> x:xs   (useful for revcat = loop (flip (:)) [])

const  :: a -> b -> a
const x _ = x
-- const 1 = \_ -> 1   (useful for length = fold (const (1+)) 0)
```

### uncurry and curry
```haskell
uncurry :: (a -> b -> c) -> (a,b) -> c
uncurry f (x,y) = f x y

curry :: ((a,b) -> c) -> a -> b -> c
curry f x y = f (x,y)

-- uncurry (+) (3,4) = 7
-- map (uncurry (+)) [(1,2),(3,4)] = [3,7]
```

### ($) and (&)
```haskell
($) :: (a -> b) -> a -> b
f $ x = f x     -- low-precedence application; avoids parentheses

-- map ($ 3) [(+1),(+2),(+3)] = [4,5,6]
-- map ($ xs) [head,last,tail] -- apply multiple functions to one list
```

### on
```haskell
on :: (b -> b -> c) -> (a -> b) -> a -> a -> c
(f `on` g) x y = f (g x) (g y)

-- sortBy (compare `on` length) ["ab","a","abc"] = ["a","ab","abc"]
-- groupBy ((==) `on` fst) [(1,'a'),(1,'b'),(2,'c')]
```

---

## 9. Fold (foldr) — Every Application

**The fundamental pattern of recursion on lists.**

### Definition
```haskell
fold :: (a -> b -> b) -> b -> [a] -> b
fold cons nil []     = nil
fold cons nil (x:xs) = cons x (fold cons nil xs)
-- = foldr in standard Haskell
```

`fold c n` replaces each `(:)` with `c` and `[]` with `n`:

```
x1 : x2 : x3 : []    -->    x1 `c` (x2 `c` (x3 `c` n))
```

**Fundamental identity:** `fold (:) [] = id`

### How to derive the fold for a function
Given `f xs = fold cons nil xs`, solve for `cons` and `nil`:
1. **nil**: `nil = f []` — evaluate `f` on the empty list
2. **cons**: from `cons x (f xs) = f (x:xs)`, substitute `f xs` by some `y` to write `cons x y = ...`

---

### Comprehensive table of fold applications

| Function | `fold c n` | `c` | `n` |
|----------|-----------|-----|-----|
| `id` | `fold (:) []` | `(:)` | `[]` |
| `(++ ys)` | `fold (:) ys` | `(:)` | `ys` |
| `sum` | `fold (+) 0` | `(+)` | `0` |
| `product` | `fold (*) 1` | `(*)` | `1` |
| `and` | `fold (&&) True` | `(&&)` | `True` |
| `or` | `fold (||) False` | `(\|\|)` | `False` |
| `all p` | `fold (\x b -> p x && b) True` | — | `True` |
| `any p` | `fold (\x b -> p x || b) False` | — | `False` |
| `length` | `fold (const (1+)) 0` | `\_ n->n+1` | `0` |
| `concat` | `fold (++) []` | `(++)` | `[]` |
| `maximum` | `fold1 max` | `max` | first elem |
| `minimum` | `fold1 min` | `min` | first elem |
| `map f` | `fold ((:).f) []` | `\x ys->f x:ys` | `[]` |
| `filter p` | `fold (\x ys -> if p x then x:ys else ys) []` | — | `[]` |
| `reverse` | `fold (\x ys -> ys++[x]) []` (naive) | — | `[]` |
| `null` | `fold (\_ _ -> False) True` | `const (const False)` | `True` |
| `head` | — | not a fold (partial) | — |
| `isort` | `fold insert []` | `insert` | `[]` |
| `cp` (cartesian product) | `fold (\xs yss -> [x:ys | x<-xs, ys<-yss]) [[]]` | — | `[[]]` |
| `cols` (transpose) | `fold (zipWith (:)) (repeat [])` | `zipWith (:)` | `repeat []` |

---

### Each application with explanation

#### sum, product, and, or
```haskell
sum     = fold (+) 0         -- 0 is identity for +
product = fold (*) 1         -- 1 is identity for *
and     = fold (&&) True     -- True is identity for &&
or      = fold (||) False    -- False is identity for ||

-- Derivation of sum:
-- nil  = sum [] = 0
-- cons x y = sum (x:xs) where y = sum xs = x + y
```

#### all and any
```haskell
all p = fold (\x b -> p x && b) True
any p = fold (\x b -> p x || b) False

-- Equivalent to:
all p xs = and (map p xs) = fold (&&) True (map p xs)
any p xs = or  (map p xs) = fold (||) False (map p xs)

-- Examples:
all even [2,4,6] = True
all even [2,3,6] = False
any odd  [2,4,5] = True
```

#### length
```haskell
length = fold (const (1+)) 0
-- const (1+) x n = (1+) n = n+1, ignores x
-- Derivation:
-- nil  = length [] = 0
-- cons x n = length (x:xs) where n = length xs = 1 + n
```

#### concat, concatMap
```haskell
concat    = fold (++) []
-- (++) appends each list to the accumulated result

concatMap f = fold (\x ys -> f x ++ ys) []
-- or: concatMap f = concat . map f
```

#### map f
```haskell
map f = fold ((:) . f) []
-- i.e. fold (\x ys -> f x : ys) []
-- Derivation:
-- nil  = map f [] = []
-- cons x ys = map f (x:xs) where ys = map f xs
--           = f x : map f xs = f x : ys = ((:).f) x ys
```

#### filter p
```haskell
filter p = fold (\x ys -> if p x then x:ys else ys) []
-- Derivation:
-- nil  = filter p [] = []
-- cons x ys = filter p (x:xs) where ys = filter p xs
--           = if p x then x:ys else ys
```

#### reverse (naive and efficient)
```haskell
-- Naive (O(n²)):
reverse = fold (\x ys -> ys ++ [x]) []

-- Efficient (O(n)) — uses loop (left fold):
reverse = loop (flip (:)) []

-- Derivation of revcat spec = reverse xs ++ ys:
-- revcat ys [] = ys
-- revcat ys (x:xs) = revcat (x:ys) xs
-- So: reverse xs = revcat [] xs = loop (flip (:)) [] xs
```

#### null
```haskell
null = fold (const (const False)) True
-- cons x _ = False (a non-empty list is not null)
-- nil = True
```

#### isort — insertion sort as fold
```haskell
isort :: Ord a => [a] -> [a]
isort = fold insert []
  where
    insert :: Ord a => a -> [a] -> [a]
    insert x []     = [x]
    insert x (y:ys) | x <= y    = x : y : ys
                    | otherwise = y : insert x ys
-- insert 3 [1,2,4,5] = [1,2,3,4,5]
-- insert 0 [1,2,3]   = [0,1,2,3]
```

#### cp — Cartesian product as fold
```haskell
-- cp [[1,2],[3,4],[5]] = [[1,3,5],[1,4,5],[2,3,5],[2,4,5]]
-- cp: produces all ways of picking one element from each list

cp :: [[a]] -> [[a]]
cp = fold (\xs yss -> [x:ys | x <- xs, ys <- yss]) [[]]

-- Derivation:
-- nil  = cp [] = [[]]   (one way to pick nothing: the empty combination)
-- cons xs yss = cp (xs:rest) where yss = cp rest
--             = [x:ys | x <- xs, ys <- yss]

-- Example step by step:
-- cp [[1,2],[3,4]] 
-- = fold (\xs yss -> [x:ys | x<-xs, ys<-yss]) [[]] [[1,2],[3,4]]
-- = (\[1,2] yss -> [x:ys | x<-[1,2], ys<-yss]) (cp [[3,4]])
-- = [x:ys | x<-[1,2], ys<-[[3],[4]]]
-- = [[1,3],[1,4],[2,3],[2,4]]
```
The Sudoku solver uses `cp` to expand all possible digit assignments.

#### cols — matrix transpose as fold
```haskell
-- cols [[1,2,3],[4,5,6],[7,8,9]] = [[1,4,7],[2,5,8],[3,6,9]]

cols :: [[a]] -> [[a]]
cols = fold (zipWith (:)) (repeat [])

-- Derivation:
-- nil = cols [] = repeat []  (infinitely many empty columns)
-- cons xs yss = cols (xs:rest) where yss = cols rest
--             = zipWith (:) xs yss
--             = [x:col | (x,col) <- zip xs yss]

-- Example:
-- cols [[1,2],[3,4]]
-- = zipWith (:) [1,2] (cols [[3,4]])
-- = zipWith (:) [1,2] (zipWith (:) [3,4] (repeat []))
-- = zipWith (:) [1,2] [[3],[4]]
-- = [[1,3],[2,4]]
```
The Sudoku solver uses `cols` to check column constraints.

---

### foldBTree
```haskell
data BTree a = Leaf a | Fork (BTree a) (BTree a)

foldBTree :: (a -> b) -> (b -> b -> b) -> BTree a -> b
foldBTree leaf fork (Leaf x)   = leaf x
foldBTree leaf fork (Fork l r) = fork (foldBTree leaf fork l)
                                       (foldBTree leaf fork r)

size    = foldBTree (const 1) (+)          -- count leaves
depth   = foldBTree (const 0) (\l r -> 1 + max l r)
flatten = foldBTree (:[]) (++)             -- list of leaves, left-to-right
fmapBT f = foldBTree (Leaf . f) Fork      -- map over BTree
```

---

## 10. Unfold — Every Application

**The dual of fold.** While fold consumes a list by replacing constructors, unfold **produces** a list by repeatedly applying deconstructors.

### Definition
```haskell
unfold :: (b -> Bool) -> (b -> a) -> (b -> b) -> b -> [a]
unfold stop head tail = rec
  where rec x | stop x    = []
               | otherwise = head x : rec (tail x)
```
Parameters:
- `stop`: termination predicate — return `[]` when `True`
- `head`: extract the next output element from state `x`
- `tail`: advance the state

### Relationship to fold
`unfold p h t . fold c n = id`  
provided `p n = True`, `p (c x y) = False`, `h (c x y) = x`, `t (c x y) = y`.

---

### Comprehensive table of unfold applications

| Function | `unfold stop head tail` | stop | head | tail |
|----------|------------------------|------|------|------|
| `iterate f x` | `unfold (const False) id f` | never | `id` | `f` |
| `take n xs` | `unfold (\(n,xs)->n==0\|\|null xs)` | n=0 or [] | `head . snd` | `(n-1, tail xs)` |
| `digits n` | `unfold (==0) (`mod`10) (`div`10)` | n=0 | `mod 10` | `div 10` |
| `ssort xs` | `unfold null minimum deleteMin` | empty | `minimum` | remove min |
| `by n xs` | `unfold null (take n) (drop n)` | empty | `take n` | `drop n` |
| `countdown` | `unfold (==0) id (subtract 1)` | 0 | `id` | `-1` |

---

### Each application with explanation

#### iterate f x
```haskell
iterate f x = x : iterate f (f x)
           = unfold (const False) id f x
-- Produces: x, f x, f(f x), f(f(f x)), ...
-- Examples:
take 5 (iterate (*2) 1) = [1,2,4,8,16]
take 4 (iterate reverse "ab") = ["ab","ba","ab","ba"]
iterate (subtract 1) 10 -- = [10,9,8,7,...,1,0,-1,-2,...]
```

#### take as unfold
```haskell
take n xs = unfold (\(n,xs) -> n==0 || null xs)
                   (\(n,xs) -> head xs)
                   (\(n,xs) -> (n-1, tail xs))
                   (n, xs)
```

#### digits extraction
```haskell
-- Extract decimal digits in reverse order:
digits :: Int -> [Int]
digits = unfold (==0) (`mod` 10) (`div` 10)
-- digits 123 = [3,2,1]   (least significant first)
-- digits 0   = []

-- Digits in correct order:
digits' n = reverse (digits n)
-- digits' 123 = [1,2,3]
```

#### ssort — selection sort as unfold
```haskell
ssort :: Ord a => [a] -> [a]
ssort = unfold null minimum deleteMin
  where deleteMin xs = delete (minimum xs) xs

-- Each step: stop if empty, output minimum, remove minimum
-- ssort [3,1,4,1] = 1 : ssort [3,4,1]
--                 = 1 : 1 : ssort [3,4]
--                 = 1 : 1 : 3 : ssort [4]
--                 = 1 : 1 : 3 : 4 : ssort []
--                 = [1,1,3,4]
```

#### by — chunk list into groups of n
```haskell
by :: Int -> [a] -> [[a]]
by n = unfold null (take n) (drop n)

-- by 3 [1..9]  = [[1,2,3],[4,5,6],[7,8,9]]
-- by 3 [1..10] = [[1,2,3],[4,5,6],[7,8,9],[10]]
-- by 2 "abcde" = ["ab","cd","e"]
```
Used in the Sudoku solver to split the grid into 3×3 boxes.

#### unfoldBTree
```haskell
unfoldBTree :: (b -> Bool) -> (b -> a) -> (b -> b) -> (b -> b) -> b -> BTree a
unfoldBTree single value left right = rec
  where rec x | single x  = Leaf (value x)
               | otherwise = Fork (rec (left x)) (rec (right x))

-- Build a balanced binary tree from a list:
build = unfoldBTree (null . tail) head
                    (\xs -> take (length xs `div` 2) xs)
                    (\xs -> drop (length xs `div` 2) xs)
-- build [1,2,3,4] = Fork (Fork (Leaf 1) (Leaf 2))
--                        (Fork (Leaf 3) (Leaf 4))
```

#### countdown
```haskell
countdown :: Int -> [Int]
countdown = unfold (==0) id (subtract 1)
-- countdown 5 = [5,4,3,2,1]

-- Equivalently: [n, n-1, ..., 1]
-- = reverse [1..n]
```

---

## 11. Fold Fusion

**The most powerful law for folds.** Lets you replace a composition `f . fold g a` with a single fold, eliminating intermediate data structures.

### Statement
If `f` is strict, `b = f a`, and `h x (f y) = f (g x y)` for all `x`, `y`, then:

```
f . fold g a  =  fold h b
```

Or equivalently: `h x . f = f . g x` (the cons-case commutes).

### Proof sketch (by induction)
- **Base**: `(f . fold g a) [] = f a = b = fold h b []` ✓
- **Step**: `(f . fold g a) (x:xs)`
  `= f (g x (fold g a xs))`
  `= h x (f (fold g a xs))`   (by the commuting condition)
  `= h x ((f . fold g a) xs)`
  `= h x (fold h b xs)`       (IH)
  `= fold h b (x:xs)` ✓

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

### Example: map f = fold ((:).f) []
Using fusion with `g = (:)`, `a = []`, `b = map f [] = []`:
- Check: `h x (map f xs) = map f (x:xs) = f x : map f xs`
- So `h x ys = f x : ys = ((:).f) x ys` ✓

### Example: (++ ys) = fold (:) ys
Using fusion with `g = (:)`, `a = []`, `b = [] ++ ys = ys`:
- Check: `h x (zs ++ ys) = (x:zs) ++ ys = x:(zs++ys)`
- So `h = (:)` ✓

### Example: scan c n = fold h [n]
(See §13 below)

---

## 12. foldr vs foldl — Left and Right Folds

### The core difference: where does the bracket go?

Given a list `[x1, x2, x3, x4]` and an operator `⊕` with identity `e`:

```
foldr (⊕) e  [x1,x2,x3,x4]  =  x1 ⊕ (x2 ⊕ (x3 ⊕ (x4 ⊕ e)))
                                          ←  right-associative: works right-to-left

foldl (⊕) e  [x1,x2,x3,x4]  =  ((((e ⊕ x1) ⊕ x2) ⊕ x3) ⊕ x4)
                                   left-associative: works left-to-right  →
```

The **element type** of the combining function differs:
```haskell
foldr :: (a -> b -> b) -> b -> [a] -> b   -- element on the LEFT, accumulator on the RIGHT
foldl :: (b -> a -> b) -> b -> [a] -> b   -- accumulator on the LEFT, element on the RIGHT
```

### Definitions

```haskell
-- foldr: the course calls this simply "fold"
foldr :: (a -> b -> b) -> b -> [a] -> b
foldr f z []     = z
foldr f z (x:xs) = f x (foldr f z xs)    -- recurse first, then apply f

-- foldl: the course calls this "loop"
foldl :: (b -> a -> b) -> b -> [a] -> b
foldl f z []     = z
foldl f z (x:xs) = foldl f (f z x) xs   -- apply f immediately, pass result forward
```

### Side-by-side concrete trace

```
foldr (-) 0 [1,2,3]
= 1 - (foldr (-) 0 [2,3])
= 1 - (2 - (foldr (-) 0 [3]))
= 1 - (2 - (3 - 0))
= 1 - (2 - 3)
= 1 - (-1)
= 2

foldl (-) 0 [1,2,3]
= foldl (-) (0-1) [2,3]
= foldl (-) (-1) [2,3]
= foldl (-) ((-1)-2) [3]
= foldl (-) (-3) [3]
= (-3) - 3
= -6
```

The results differ because `-` is not associative.

### When they give the same result

`foldr (⊕) e xs = foldl (⊕) e xs` when `(⊕)` is:
1. **Associative**: `(a ⊕ b) ⊕ c = a ⊕ (b ⊕ c)`
2. **e is both left and right identity**: `e ⊕ x = x ⊕ e = x`

```haskell
-- These are equal for any xs:
foldr (+) 0 xs = foldl (+) 0 xs    -- (+) associative, 0 is identity
foldr (*) 1 xs = foldl (*) 1 xs    -- (*) associative, 1 is identity

-- These are NOT equal in general:
foldr (-) 0 xs ≠ foldl (-) 0 xs    -- (-) not associative
foldr (++) [] xs ≠ foldl (++) [] xs -- subtle: equal on finite lists but differ on ⊥
```

### Which to use when?

| Situation | Use | Reason |
|-----------|-----|--------|
| Building a list (e.g. `map`, `filter`) | `foldr` | works on infinite lists; lazy |
| Reducing to a number/bool (e.g. `sum`) | `foldl'` | avoids thunk build-up |
| Need the original list order preserved | `foldr` | visits left-to-right naturally |
| Computing `reverse` | `foldl` | natural left-to-right accumulation |
| Infinite list input | `foldr` only | `foldl` will loop forever |

```haskell
-- foldr can work on infinite lists because it is lazy:
foldr (\x _ -> x) undefined [1,2,3..] = 1    -- stops after first step

-- foldl CANNOT work on infinite lists:
foldl (+) 0 [1..] = ⊥   -- never terminates
```

### Examples using foldl

```haskell
-- reverse: naturally left-associative
reverse = foldl (flip (:)) []
-- foldl (flip (:)) [] [1,2,3]
-- = foldl (flip (:)) [1] [2,3]
-- = foldl (flip (:)) [2,1] [3]
-- = foldl (flip (:)) [3,2,1] []
-- = [3,2,1]  ✓

-- Running total (left scan result):
foldl (+) 0 [1,2,3,4] = 10   -- same as sum

-- Building a string backwards:
foldl (\acc c -> c:acc) [] "hello" = "olleh"
```

### Space behaviour

`foldl` builds a chain of unevaluated thunks:
```
foldl (+) 0 [1,2,3]
→ foldl (+) (0+1) [2,3]           -- thunk: (0+1)
→ foldl (+) ((0+1)+2) [3]         -- thunk: ((0+1)+2)
→ foldl (+) (((0+1)+2)+3) []      -- thunk: (((0+1)+2)+3)
→ evaluate: 6                     -- forced at the end
```

Use **`foldl'`** (strict left fold) to force evaluation at each step:
```haskell
import Data.List (foldl')

foldl' (+) 0 [1,2,3]              -- evaluates accumulator eagerly: no thunk build-up
-- = foldl' (+) 1 [2,3]
-- = foldl' (+) 3 [3]
-- = foldl' (+) 6 []
-- = 6                            -- O(1) space vs O(n) for foldl

-- The course calls this loop':
loop' s (!n) []     = n
loop' s (!n) (x:xs) = loop' s (s n x) xs
```

### Summary table

| | `foldr f z` | `foldl f z` | `foldl' f z` |
|--|-------------|-------------|--------------|
| Associativity | right `x⊕(y⊕z)` | left `(x⊕y)⊕z` | left `(x⊕y)⊕z` |
| Processes | right-to-left | left-to-right | left-to-right |
| Works on infinite lists | yes (if f lazy) | no | no |
| Space (numeric reduction) | O(n) thunks | O(n) thunks | O(1) |
| Course name | `fold` | `loop` | `loop'` |
| Haskell Prelude | `foldr` | `foldl` | `foldl'` (Data.List) |

---

## 13. Scan — scanr and scanl

### The big picture

A **scan** is to a **fold** what intermediate steps are to a final answer. Instead of returning one result, it returns a list of all the partial results.

```
foldr (+) 0  [1, 2, 3]  =  6           -- single answer
scanr (+) 0  [1, 2, 3]  =  [6, 5, 3, 0]  -- all suffix-fold results
```

```
foldl (+) 0  [1, 2, 3]  =  6           -- single answer
scanl (+) 0  [1, 2, 3]  =  [0, 1, 3, 6]  -- all prefix-fold results
```

---

### scanr — right scan (the course's "scan")

**Specification**: apply `foldr` to every suffix (tail) of the list.

```haskell
scanr :: (a -> b -> b) -> b -> [a] -> [b]
scanr c n = map (foldr c n) . tails

tails :: [a] -> [[a]]
tails []     = [[]]
tails (x:xs) = (x:xs) : tails xs

-- tails [1,2,3] = [[1,2,3], [2,3], [3], []]
```

So:
```
scanr c n [x,y,z]
= map (foldr c n) [[x,y,z], [y,z], [z], []]
= [x`c`(y`c`(z`c`n)),  y`c`(z`c`n),  z`c`n,  n]
```

**Concrete examples:**
```haskell
scanr (+) 0 [1,2,3]   = [6, 5, 3, 0]
-- 1+(2+(3+0))=6,  2+(3+0)=5,  3+0=3,  0

scanr (*) 1 [1,2,3,4] = [24, 24, 12, 4, 1]
-- 1*2*3*4=24, 2*3*4=24, 3*4=12, 4*1=4, 1

scanr (:) [] [1,2,3]  = [[1,2,3],[2,3],[3],[]]
-- same as tails!

scanr max 0 [3,1,4,1,5] = [5,5,5,5,5,0]
-- suffix maxima

scanr (-) 0 [1,2,3]   = [2, -1, 3, 0]
-- 1-(2-(3-0))=1-(-1)=2,  2-(3-0)=2-3=-1,  3-0=3,  0
```

**Key property:**
```haskell
head (scanr c n xs) = foldr c n xs
-- The FIRST element is the fold of the whole list.
last (scanr c n xs) = n
-- The LAST element is always the initial value.
```

**Length:** `length (scanr c n xs) = length xs + 1`

### Efficient scanr via fold fusion

Naive implementation is O(n²) — computing each suffix fold separately.

```haskell
-- Specification (quadratic):
scanr c n = map (foldr c n) . tails

-- Efficient (linear) — derived by fold fusion:
scanr c n = foldr h [n]
  where h x zs = c x (head zs) : zs
-- At each step: prepend c x (next result), passing along previous results.
```

**Derivation intuition:**
- The fold builds the result list from right to left.
- When we process element `x` and the rest has already been scanned to `zs = [r1, r2, ...]`,
  the new head is `c x r1` (since `r1 = foldr c n rest`).

---

### scanl — left scan

**Specification**: apply `foldl` to every prefix (initial segment) of the list.

```haskell
scanl :: (b -> a -> b) -> b -> [a] -> [b]
scanl f z = map (foldl f z) . inits

inits :: [a] -> [[a]]
inits []     = [[]]
inits (x:xs) = [] : map (x:) (inits xs)

-- inits [1,2,3] = [[], [1], [1,2], [1,2,3]]
```

So:
```
scanl f z [x,y,z]
= map (foldl f z) [[], [x], [x,y], [x,y,z]]
= [z,  f z x,  f(f z x)y,  f(f(f z x)y)z]
```

**The first element is always the initial value `z`.**

**Concrete examples:**
```haskell
scanl (+) 0 [1,2,3]   = [0, 1, 3, 6]
-- 0,  0+1=1,  1+2=3,  3+3=6
-- running totals, growing from left

scanl (*) 1 [1,2,3,4] = [1, 1, 2, 6, 24]
-- running products (factorials!)

scanl (flip (:)) [] [1,2,3] = [[], [1], [2,1], [3,2,1]]
-- shows how foldl builds reverse

scanl (-) 0 [1,2,3]   = [0, -1, -3, -6]
-- 0,  0-1=-1,  -1-2=-3,  -3-3=-6

scanl max 0 [3,1,4,1,5] = [0, 3, 3, 4, 4, 5]
-- running maximum
```

**Key property:**
```haskell
last (scanl f z xs) = foldl f z xs
-- The LAST element is the fold of the whole list.
head (scanl f z xs) = z
-- The FIRST element is always the initial value.
```

**Efficient scanl** (already linear — no fusion needed):
```haskell
scanl f z []     = [z]
scanl f z (x:xs) = z : scanl f (f z x) xs
```

---

### scanr vs scanl — side-by-side comparison

```
List:       [1,   2,   3]   initial: 0

scanr (+) 0: [6,   5,   3,   0]    -- suffix sums (right-to-left)
              ^                ^
           fold whole        initial

scanl (+) 0: [0,   1,   3,   6]    -- prefix sums (left-to-right)
              ^                ^
           initial           fold whole
```

| | `scanr f z xs` | `scanl f z xs` |
|--|----------------|----------------|
| Corresponds to | `foldr` on suffixes | `foldl` on prefixes |
| First element | `foldr f z xs` | `z` |
| Last element | `z` | `foldl f z xs` |
| Length | `length xs + 1` | `length xs + 1` |
| Order of results | right-to-left partial results | left-to-right partial results |
| Works on infinite lists | yes (lazy) | no (needs all of `xs`) |
| Course name | `scan` | — |
| Haskell Prelude | `scanr` | `scanl` |

**When results are the same:** Only when `f` is associative, commutative, and `z` is the identity — e.g. `scanr (+) 0 xs` and `scanl (+) 0 xs` contain the same elements but in **reverse order**:
```haskell
scanr (+) 0 [1,2,3] = [6,5,3,0]
scanl (+) 0 [1,2,3] = [0,3,5,6]
-- same numbers, reversed!
-- This holds because (+) is associative and commutative.
```

---

## 14. Proof by Induction

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

### Example proof: map f . map g = map (f . g)
By induction on xs:

**Base** (xs = []):
```
(map f . map g) [] = map f (map g []) = map f [] = []
map (f . g)     [] = []                             ✓
```

**Step** (xs = y:ys), assuming `(map f . map g) ys = map (f.g) ys`:
```
(map f . map g) (y:ys)
= map f (g y : map g ys)
= f (g y) : map f (map g ys)
= (f . g) y : (map f . map g) ys    (definition of (.))
= (f . g) y : map (f . g) ys        (IH)
= map (f . g) (y:ys)                ✓
```

---

## 15. Types and Trees

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
-- Fold-like deconstructor:
either f g (Left x)  = f x
either f g (Right y) = g y
-- Identity: either Left Right = id
```

### Polynomial types (sums of products)
Types built from `data` are **sums** (alternatives `|`) of **products** (tuples/fields).  
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

depth :: BTree a -> Int
depth = foldBTree (const 0) (\l r -> 1 + max l r)
```

Efficient flatten (linear, avoiding quadratic `++`):
```haskell
flatcat :: BTree a -> [a] -> [a]
flatcat (Leaf x)   ys = x : ys
flatcat (Fork l r) ys = flatcat l (flatcat r ys)
-- flatcat = foldBTree (:) (flip (.))
-- flatten t = flatcat t []
```

---

## 16. Rose Trees and Bush Trees

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

-- Examples:
sizeRT :: RTree a -> Int
sizeRT = foldRTree (\_ ns -> 1 + sum ns)

depthRT :: RTree a -> Int
depthRT = foldRTree (\_ ds -> 1 + maximum (0:ds))

flattenRT :: RTree a -> [a]
flattenRT = foldRTree (\x xss -> x : concat xss)
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

## 17. Functor and fmap

### The core intuition: a box you can map over

A **Functor** is any type that acts like a **container** or **context** holding values of some type `a`, and lets you apply a function to those values without changing the structure of the container.

Think of it as: *"apply a function to the contents; leave the box shape intact."*

```
                f :: a -> b
              ────────────────
  Maybe Int ──────fmap────────▶  Maybe String
  Just 42                        Just "42"
  Nothing                        Nothing         (box shape preserved)

  [Int]     ──────fmap────────▶  [String]
  [1,2,3]                        ["1","2","3"]   (list structure preserved)

  RTree Int ──────fmap────────▶  RTree String
  (same tree shape, values changed)
```

The type class:
```haskell
class Functor f where
  fmap :: (a -> b) -> f a -> f b
--         ↑           ↑       ↑
--     function    container  container
--     to apply    of a's     of b's
```

`f` here is **the container type** (`Maybe`, `[]`, `Either String`, …), not a plain function.

---

### The two Functor laws

These say: *fmap should not touch the structure, and fmapping twice equals fmapping the composition.*

```haskell
-- Law 1: fmap id = id
-- Applying the identity function changes nothing.
fmap id (Just 5) = Just 5      ✓
fmap id [1,2,3]  = [1,2,3]     ✓

-- Law 2: fmap f . fmap g = fmap (f . g)
-- Two fmaps in a row = one fmap of the composition.
fmap (+1) (fmap (*2) (Just 3))  = Just 7
fmap ((+1) . (*2)) (Just 3)     = Just 7   -- same result, one pass ✓
```

Haskell does **not enforce** these laws — it is the programmer's responsibility.

---

### Standard instances, explained

#### List — the obvious one
```haskell
instance Functor [] where
  fmap = map

fmap (*2) [1,2,3] = [2,4,6]
```
`map` **is** `fmap` for lists. They are identical.

#### Maybe — "might not be there"
```haskell
instance Functor Maybe where
  fmap f Nothing  = Nothing
  fmap f (Just x) = Just (f x)
```

Rule: if the value is absent, it stays absent. If present, apply `f` to it.

```haskell
fmap (*2)     Nothing   = Nothing
fmap (*2)     (Just 5)  = Just 10
fmap show     (Just 42) = Just "42"
fmap (++ "!") Nothing   = Nothing

-- Practical use: safely transform a result that might have failed
fmap (*10) (safeHead [1,2,3]) = Just 10
fmap (*10) (safeHead [])      = Nothing   -- failure propagates
```

#### Either — "might be an error"
```haskell
instance Functor (Either e) where
  fmap f (Left err) = Left err   -- error passes through unchanged
  fmap f (Right x)  = Right (f x)
```

`fmap` only touches the `Right` (success) value. Errors (`Left`) are ignored — this is the short-circuit behaviour.

```haskell
fmap (*2) (Right 5)      = Right 10
fmap (*2) (Left "oops")  = Left "oops"    -- error unchanged
fmap show (Right 42)     = Right "42"

-- Chain transformations; errors propagate automatically:
fmap (*2) (fmap (+1) (Right 3))  = Right 8
fmap (*2) (fmap (+1) (Left "x")) = Left "x"
```

#### RTree — "values in a tree"
```haskell
instance Functor RTree where
  fmap f (RTree a ts) = RTree (f a) (map (fmap f) ts)
--                              ↑                ↑
--                         apply f to root     recursively fmap subtrees
```

The tree **shape** (branching) is preserved; every value at every node is transformed by `f`.

```haskell
-- Tree:       1
--            / \
--           2   3
--                \
--                 4

fmap (*10) (RTree 1 [RTree 2 [], RTree 3 [RTree 4 []]])
= RTree 10 [RTree 20 [], RTree 30 [RTree 40 []]]
-- Tree:       10
--            /  \
--          20    30
--                 \
--                  40
```

---

### Why `Either e` not `Either`?

`fmap` has type `(a -> b) -> f a -> f b`, so `f` must take **exactly one** type argument. `Either` takes two:

```
Either   :: * -> * -> *    -- two type arguments
Either e :: * -> *         -- partially applied: takes one argument
```

So `Functor (Either e)` makes `Either e` a functor in its **second** argument — that is why only `Right` is affected by `fmap`. The `Left e` part is the "fixed" structure.

---

### fmap derived from fold

For any recursive type with a fold, `fmap` follows naturally — replace constructors with constructors that apply `f` to values:

```haskell
-- Lists:
map f = foldr ((:) . f) []

-- BTree:
fmapBT :: (a -> b) -> BTree a -> BTree b
fmapBT f = foldBTree (Leaf . f) Fork
-- Leaves: apply f, re-wrap in Leaf. Forks: keep Fork unchanged.

-- RTree:
fmapRT :: (a -> b) -> RTree a -> RTree b
fmapRT f = foldRTree (\x bs -> RTree (f x) bs)
```

---

### The power: "lift any function into any context"

```haskell
double :: Int -> Int
double = (*2)

-- The same function works in any Functor:
fmap double (Just 5)   = Just 10
fmap double [1,2,3]    = [2,4,6]
fmap double (Right 5)  = Right 10
fmap double (Left "e") = Left "e"   -- no-op on error
```

Write `double` once; use it inside any container without rewriting it.

### map as a special case
`map` is `fmap` for lists. For any recursive type, `fmap` can be derived from the fold:
```haskell
-- For lists:
map f = fold ((:) . f) []
-- For BTree:
fmapBT f = foldBTree (Leaf . f) Fork
-- For RTree:
fmapRT f = foldRTree (RTree . f) -- :: (a->b) -> RTree a -> RTree b
```

### Summary

| Container `f` | What `fmap f z` does |
|---------------|----------------------|
| `[]` | applies `f` to every element |
| `Maybe` | applies `f` if `Just`, passes `Nothing` through |
| `Either e` | applies `f` to `Right`, passes `Left` through |
| `RTree` | applies `f` to every node value, keeps tree shape |
| `Bush t` | applies `f` to every node value, keeps shape |

**The Functor contract:** `fmap` reaches inside the container, applies your function to every `a`, and returns the same-shaped container full of `b`s.

---

## 18. Efficiency: Accumulator Pattern

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
A left-skewed tree has O(n²) cost from repeated `(++)`.

### Linear flatten via accumulator
```haskell
flatcat :: BTree a -> [a] -> [a]     -- spec: flatcat t ys = flatten t ++ ys
flatcat (Leaf x)   ys = x : ys
flatcat (Fork l r) ys = flatcat l (flatcat r ys)
-- Then: flatten t = flatcat t []
-- flatcat = foldBTree (:) (flip (.))
```

### Fast exponentiation
```haskell
-- Naive: n multiplications
pow x n = if n == 0 then 1 else pow x (n-1) * x

-- O(log n) with accumulator:
power (*) y x 0        = y
power (*) y x n
  | even n = power (*) y (x*x) (n `div` 2)
  | odd  n = power (*) (x*y) x (n-1)
-- Then: pow x n = power (*) 1 x n
```
Works for any associative operation (e.g. matrix multiplication for matrix exponentiation).

---

## 19. Sorting Algorithms — Full Detail

### 1. Insertion Sort — a fold

**Idea**: insert each element into the correct position of the already-sorted accumulator.

```haskell
isort :: Ord a => [a] -> [a]
isort = fold insert []

insert :: Ord a => a -> [a] -> [a]
insert x []     = [x]
insert x (y:ys)
  | x <= y    = x : y : ys    -- x belongs here
  | otherwise = y : insert x ys

-- Example trace:
-- isort [3,1,2]
-- = insert 3 (insert 1 (insert 2 []))
-- = insert 3 (insert 1 [2])
-- = insert 3 [1,2]
-- = [1,2,3]
```

**Complexity:**
- Best case: O(n) — already sorted
- Worst/average: O(n²) — each `insert` may scan the whole list
- Space: O(n)

**Characterisation:** Insertion sort is a **fold** — builds the result incrementally from left to right.

---

### 2. Selection Sort — an unfold

**Idea**: repeatedly extract the minimum element.

```haskell
ssort :: Ord a => [a] -> [a]
ssort = unfold null minimum deleteMin
  where
    deleteMin xs = delete (minimum xs) xs

-- delete removes the first occurrence of an element:
delete :: Eq a => a -> [a] -> [a]
delete _ []     = []
delete x (y:ys) | x == y    = ys
                | otherwise = y : delete x ys

-- Example trace:
-- ssort [3,1,2]
-- = 1 : ssort [3,2]
-- = 1 : 2 : ssort [3]
-- = 1 : 2 : 3 : ssort []
-- = [1,2,3]
```

**Complexity:**
- Best/Worst/Average: O(n²) — `minimum` and `deleteMin` each traverse the list
- Space: O(n)

**Characterisation:** Selection sort is an **unfold** — generates the result element by element.

---

### 3. Quicksort — unfold then fold

**Idea**: partition around a pivot, sort each part recursively.

```haskell
qsort :: Ord a => [a] -> [a]
qsort []     = []
qsort (x:xs) = qsort smaller ++ [x] ++ qsort larger
  where
    smaller = filter (< x) xs
    larger  = filter (> x) xs
-- Note: this version drops duplicates; include equal with smaller or larger as needed

-- Full version preserving duplicates:
qsort' []     = []
qsort' (x:xs) = qsort' [y | y <- xs, y < x]
             ++ [y | y <- x:xs, y == x]
             ++ qsort' [y | y <- xs, y > x]

-- Example trace:
-- qsort [3,1,4,1,5]
-- pivot=3, smaller=[1,1], larger=[4,5]
-- = qsort [1,1] ++ [3] ++ qsort [4,5]
-- = [1,1] ++ [3] ++ [4,5]
-- = [1,1,3,4,5]
```

**Complexity:**
- Best/Average: O(n log n) — balanced partitions
- Worst: O(n²) — already sorted (pivot always min or max)
- Space: O(log n) average, O(n) worst (stack depth)

**Characterisation:** Quicksort = **unfold** (build a QTree partition) followed by **fold** (flatten QTree in order).

```haskell
-- As unfold-then-fold:
data QTree a = QLeaf | QFork (QTree a) a (QTree a)

buildQ :: Ord a => [a] -> QTree a
buildQ = unfoldQTree null (\(x:_) -> x) (\(x:xs) -> filter (<x) xs) (\(x:xs) -> filter (>x) xs)

flattenQ :: QTree a -> [a]
flattenQ = foldQTree [] (\l x r -> l ++ [x] ++ r)

qsort = flattenQ . buildQ
```

---

### 4. Merge Sort — unfold then fold

**Idea**: split in half, sort each half, merge sorted halves.

```haskell
msort :: Ord a => [a] -> [a]
msort []  = []
msort [x] = [x]
msort xs  = merge (msort ls) (msort rs)
  where (ls, rs) = halve xs

halve :: [a] -> ([a],[a])
halve xs = splitAt (length xs `div` 2) xs

merge :: Ord a => [a] -> [a] -> [a]
merge []     ys     = ys
merge xs     []     = xs
merge (x:xs) (y:ys)
  | x <= y    = x : merge xs (y:ys)
  | otherwise = y : merge (x:xs) ys

-- Example trace:
-- msort [3,1,4,2]
-- = merge (msort [3,1]) (msort [4,2])
-- = merge (merge (msort [3]) (msort [1])) (merge (msort [4]) (msort [2]))
-- = merge (merge [3] [1]) (merge [4] [2])
-- = merge [1,3] [2,4]
-- = [1,2,3,4]
```

**Complexity:**
- Best/Average/Worst: O(n log n) — always balanced splits
- Space: O(n) — merge creates a new list
- Note: `length xs` makes `halve` O(n); can use tortoise-hare to avoid

**Characterisation:** Merge sort = **unfold** (build an MTree of singletons/pairs) followed by **fold** (merge bottom-up).

```haskell
-- As unfold-then-fold:
data MTree a = MLeaf a | MFork (MTree a) (MTree a)

buildM :: [a] -> MTree a
buildM [x] = MLeaf x
buildM xs  = MFork (buildM ls) (buildM rs) where (ls,rs) = halve xs

flattenM :: Ord a => MTree a -> [a]
flattenM (MLeaf x)   = [x]
flattenM (MFork l r) = merge (flattenM l) (flattenM r)

msort = flattenM . buildM
```

---

### Complexity Summary

| Algorithm | Best | Average | Worst | Space | Characterisation |
|-----------|------|---------|-------|-------|-----------------|
| Insertion sort | O(n) | O(n²) | O(n²) | O(n) | fold |
| Selection sort | O(n²) | O(n²) | O(n²) | O(n) | unfold |
| Quicksort | O(n log n) | O(n log n) | O(n²) | O(log n) | unfold then fold |
| Merge sort | O(n log n) | O(n log n) | O(n log n) | O(n) | unfold then fold |

---

## 20. Sudoku: Standard Functions in Practice

The Sudoku solver from lectures 7–8 is a rich example of applying every standard list function.

### Grid representation
```haskell
type Grid  = [[Digit]]
type Digit = Char
digits     = ['1'..'9']
blank      = (== '0')
```

### cp — Cartesian product (fold)
```haskell
cp :: [[a]] -> [[a]]
cp = fold (\xs yss -> [x:ys | x <- xs, ys <- yss]) [[]]

-- Used to expand all possible digit choices:
-- expand :: Grid -> [Grid]
-- expand = cp . map choices
-- where choices c = if blank c then digits else [c]
```

### cols — transpose (fold with zipWith)
```haskell
cols :: [[a]] -> [[a]]
cols = fold (zipWith (:)) (repeat [])

-- Used to check column constraints:
-- valid grid means: all rows ok, all cols ok, all boxes ok
-- rows = id
-- cols = transpose
```

### by — chunking with splitAt (unfold)
```haskell
by :: Int -> [a] -> [[a]]
by n = unfold null (take n) (drop n)

-- Used to split grid into 3×3 boxes:
-- boxs :: [[a]] -> [[a]]
-- boxs = map concat . concat . map cols . by 3 . map (by 3)
-- Explanation:
--   map (by 3) :: [[a]] -> [[[a]]]   -- split each row into triples
--   by 3       :: [[[a]]] -> [[[[a]]]] -- group into groups of 3 rows
--   map cols   :: [[[[a]]]] -> [[[[a]]]] -- transpose each group
--   concat . map concat -- flatten
```

### zipWith (:) for column transpose
```haskell
-- transpose via fold:
-- fold (zipWith (:)) (repeat []) [[1,2],[3,4],[5,6]]
-- = zipWith (:) [1,2] (fold (zipWith (:)) (repeat []) [[3,4],[5,6]])
-- = zipWith (:) [1,2] (zipWith (:) [3,4] (zipWith (:) [5,6] (repeat [])))
-- = zipWith (:) [1,2] (zipWith (:) [3,4] [[5],[6]])
-- = zipWith (:) [1,2] [[3,5],[4,6]]
-- = [[1,3,5],[2,4,6]]
```

### filter and map for pruning
```haskell
-- Remove choices that are already fixed in a row/col/box:
remove :: [Digit] -> [Digit] -> [Digit]
remove fixed cs
  | length cs == 1 = cs          -- already fixed; don't remove
  | otherwise      = filter (`notElem` fixed) cs  -- remove singletons

ones :: [[Digit]] -> [Digit]
ones = concat . filter (\cs -> length cs == 1)    -- extract fixed digits

-- pruneRow: prune a row given its fixed digits
pruneRow :: [[Digit]] -> [[Digit]]
pruneRow row = map (remove fixed) row
  where fixed = ones row
```

### span / break for search
```haskell
-- break p = span (not . p)
-- Used to find the first unfixed cell and split around it:
expand1 :: Grid -> [Grid]
expand1 grid = [rows1 ++ [row1 ++ [c] : row2] ++ rows2 | c <- cs]
  where
    (rows1, row:rows2) = break (any ((>1) . length)) grid
    (row1, cs:row2)    = break ((>1) . length) row
```

### repeatedly — iterate until stable
```haskell
repeatedly :: (a -> a) -> a -> a
repeatedly f x | f x == x  = x
               | otherwise = repeatedly f (f x)
-- Used to apply pruning until no more changes:
-- repeatedly prune grid
```

---

## 21. Dynamic Programming and Tabulation

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

**Complexity**: `fibK` with `tabulate` runs in O(n) time and O(n) space — each `fib k` is computed at most once.

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

## 22. Key Laws and Identities Cheatsheet

### Fold laws
```
fold (:) []           = id
fold c n (xs ++ ys)  = fold c n xs `op` fold c n ys
  (when x `op` (y `op` z) = (x `c` y) `op` z and n `op` x = x)
map f                 = fold ((:) . f) []
filter p              = fold (\x ys -> if p x then x:ys else ys) []
concat                = fold (++) []
sum                   = fold (+) 0
product               = fold (*) 1
length                = fold (const (1+)) 0
and                   = fold (&&) True
or                    = fold (||) False
(++ ys)               = fold (:) ys
cp                    = fold (\xs yss -> [x:ys | x<-xs, ys<-yss]) [[]]
cols                  = fold (zipWith (:)) (repeat [])
isort                 = fold insert []
```

### Unfold identities
```
iterate f             = unfold (const False) id f
digits n              = unfold (==0) (`mod`10) (`div`10)   n
by n                  = unfold null (take n) (drop n)
ssort                 = unfold null minimum deleteMin
```

### Fusion law
```
f strict, b = f a, h x (f y) = f (g x y)
⟹  f . fold g a  =  fold h b
```

### Scan theorem
```
head (scan c n xs)  =  fold c n xs
scan c n            =  fold h [n]   where h x zs = c x (head zs) : zs
```

### fold vs loop
```
fold (+) 0  = loop (+) 0        (+ associative, right-strict, 0 is left unit)
fold (++) [] ≠ loop (++) []     (++ not right-strict)
reverse      = loop (flip (:)) []
```

### Map laws
```
map id            = id
map f . map g     = map (f . g)            -- fusion
map f (xs ++ ys)  = map f xs ++ map f ys
fmap id           = id                      (Functor law)
fmap f . fmap g   = fmap (f . g)            (Functor law)
```

### Either identity
```
either Left Right  = id
```

### Maybe vs Either
```
maybe def f Nothing  = def
maybe def f (Just x) = f x
either l r (Left x)  = l x
either l r (Right y) = r y
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

### Standard function complexity
| Function | Time | Note |
|----------|------|------|
| `map f xs` | O(n) | n = length xs |
| `filter p xs` | O(n) | |
| `concat xss` | O(total length) | |
| `(++) xs ys` | O(length xs) | strict in left arg |
| `reverse xs` | O(n²) naive, O(n) accumulator | |
| `sort xs` | O(n log n) | Data.List uses merge sort |
| `take n xs` | O(min n (length xs)) | works on infinite lists |
| `zip xs ys` | O(min (length xs) (length ys)) | |
| `fold f n xs` | O(n) calls to f | |
| `isort xs` | O(n²) | fold of insert |
| `ssort xs` | O(n²) | unfold of minimum |
| `qsort xs` | O(n log n) avg | O(n²) worst |
| `msort xs` | O(n log n) | always |

---

*Course: Functional Programming MT2023, Geraint Jones, University of Oxford*  
*Primary textbook: "Thinking Functionally with Haskell" (Bird)*
