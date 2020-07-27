+++
title = "几个 Haskell Functor/Monad 实例的演算推导"
date = 2020-07-27T23:03:00+08:00
lastmod = 2020-07-27T23:40:38+08:00
tags = ["haskell", "functor", "monad"]
categories = ["haskell"]
draft = false
+++

最近学习 Java, 看到 Java 8 以后引进了 lambda, 然后就顺理成章的引入了 Stream 式的函数式编程方式，默认惰性求值，甚至引进了 Optional interface. 这真是没想到，Java 这个浓眉大眼的也早就叛变了。于是翻出了下面这篇旧文。

学习 haskell 的过程中，functor/monad 的运算和应用是个坎。

光是从这几个的类簇（类型类）的定义来看，都太抽象了。 `learn you a great haskell` 是很好的入门书。对几种常见的 functor(monad) 都有介绍。但是在介绍它们的运算中符号使用得太灵活。同样的 `f` 有时是 functor, 有时是 function. 为了好好掌握，自己作了一下演算对比总结。


## Definitions {#definitions}

以下是 (applicative)functor/monad 的定义。除些还有几个用于它们身上的操作符，我们将在下一节中用几个实例推演它们是如何实现这些定义的，还有这些运算符在这些实例上运算的实际效果是什么

```haskell
class Functor f where
  fmap :: (a -> b) -> f a -> f b
```

```haskell
class (Functor fr) => Applicative fr where
  fmap :: (a -> b) -> fr a -> fr b
  pure :: a -> fr a
  (<*>) :: fr (a -> b) -> fr a -> fr b
```

```haskell
class Monad m where
  return :: a -> ma

  (>>=) :: ma -> (a -> m b) -> m b

  (>>) :: m a -> m b -> m b
  m x >> m y = m x >>= \_ -> m y

  fail :: String -> m a
  fail msg = error msg
```

```haskell
(<$>) :: (Functor fr) => (a -> b) -> fr a -> fr b
fn (<$>) x = fmap fn x
(<$>) = `fmap`
```

```haskell
liftA2 :: (Applicative fr) => (a -> b -> c) -> fr a -> fr b -> fr c
liftA2 fn x y = fn <$> x <*> y = fmap fn x <*> y
```

```haskell
sequenceA :: (Applicative fr) => [fr a] -> fr [a]
sequenceA [] = pure []
sequenceA (x:xs) = (:) <$> x <*> sequenceA xs
sequenceA = foldr (liftA2 (:)) (pure [])
```


## Instances {#instances}


### Maybe {#maybe}

```haskell
instance Applicative Maybe where
  -- functor
  fmap :: (a -> b) -> Maybe a -> Maybe b
  fmap f (Just x) = Just (f x)
  fmap Nothing = Nothing

  -- applicative functor
  pure :: a -> Just a
  pure = Just

  (<*>) :: Maybe(a -> b) -> Maybe a -> Maybe b
  Nothing <*> _ = Nothing
  (Just f) <*> Nothing = fmap f Nothing = Nothing
  (Just f) <*> (Just x) = fmap f (Just x) = Just (f x)
  (Just f) <*> something = fmap f something

  liftA2 :: (a -> b -> c) -> Maybe a -> Maybe b -> Maybe c
  liftA2 fn (Maybe x) (Maybe y)
    = fn <$> (Maybe x) <*> (Maybe y)
    = fmap fn (Maybe x) <*> (Maybe y)
    = Maybe (fn x) <*> Maybe y
    = Maybe (fn x y)

  sequenceA :: [Maybe a] -> Maybe [a]
  sequenceA :: (Maybe x):maybes = fmap (:) (Maybe x) <*> sequenceA maybes = Maybe (x:) <*> maybes
  sequenceA = foldr (liftA2 (:)) (Just [])

  -- monad
  return x = Just x

  (>>=) :: Maybe a -> (a -> Maybe b) -> Maybe b
  Nothing >>= f = Nothing
  Just x >>= f = f x

  foo :: Maybe String
  foo = Just 3 >>= \x -> Just "!" >>= \y -> Just (show x ++ y)
    = Just 3 >>= (\x -> Just "!" >>= (\y -> Just (show x ++ y)))
    = Just "!" >>= (\y -> Just (show 3 ++ y))
    = Just (show 3 ++ "!")
    = "3!"

  fail _ = Nothing
```


### (Either e) {#either-e}

```haskell
data Either a b = Left a | Right b

-- functor
instance Functor (Either a) where
  fmap :: (b -> c) -> Either a b -> Either a c
  fmap f (Right x) = Right (f x)
  fmap f (Left x) = Left x

-- monad
 instance (Error e) => Monad (Either e) where
   return :: a -> (Either e) a
   return x = Right x

   (>>=) :: (Either e a) -> (a -> Either e a) -> (Either e a)
   Right x >>= f = f x
   Left err >>= f = Left err
   fail msg = Left (strMsg msg)
```


### List {#list}

```haskell
instance Applicative [] where
  fmap :: (a -> b) -> [a] -> [b]
  fmap = map
  fmap [] = []
  fmap fn (x:xs) = (fn x):(fmap fn xs)

  pure :: a -> [a]
  pure x = [x]

  (<*>) :: [(a -> b)] -> [a] -> [b]
  fs <*> xs = [f x | f <- fs, x <- xs]

  liftA2 :: (a -> b -> c) -> [a] -> [b] -> [c]
  liftA2 fn xs ys
    = fn <$> xs <*> ys
    = fmap fn xs <*> ys
    = map fn xs <*> ys
    = [fn x y | x <- xs, y <- ys]

  sequenceA :: [[a]] -> [[a]]
  sequenceA (xs:xss) = map (:) x <*> sequenceA xss
  sequenceA = foldr (liftA2 (:)) ([[]])

instance Monad [] where
  return x = [x]

  (>>=) :: [a] -> (a -> [b]) -> [b]
  xs >>= f = concat (map f xs)

  foo :: [(Int, String)]
  foo = [1, 2] >>= \n -> ['a', 'b'] >>= \ch -> return (n, ch)
    = [1, 2] >>= (\n -> ['a', 'b'] >>= (\ch -> return (n, ch)))
    = concat [['a', 'b'] >>= (\ch -> return (1, ch)),
      ['a', 'b'] >>= (\ch -> return (2, ch))]
    = concat [concat [[(1, 'a')], [(1, 'b')]],
      concat [[(2, 'a')], [(2, 'b')]]]
    = concat [[(1, 'a'), (1, 'b')], [(2, 'a'), (2, 'b')]]
    = [(1, 'a'), (1, 'b'), (2, 'a'), (2,'b')]

  fail _ = []
```


### IO {#io}

```haskell
instance Applicative IO where
  fmap :: (a -> b) -> IO a -> IO b
  fmap f action = do
    result <- action
    return (f action)

  pure :: a -> IO a
  pure = return
  pure x = return x

  (<*>) :: IO(a -> b) -> IO a -> IO b
  a <*> b = do
    f <- a
    x <- b
    return (f x)

  liftA2 :: (a -> b -> c) -> IO a -> IO b -> IO c
  liftA2 fn (IO x) (IO y)
    = fn <$> (IO x) (IO y)
    = fmap fn (IO x) <*> (IO y)
    = IO (fn x) <*> (IO y)
    = IO (fn x y)

  sequenceA :: [IO a] -> IO [a]
  sequenceA = foldr (liftA2 (:)) (IO [])
```


### function {#function}

```haskell
instance Applicative ((->) r) where
  fmap :: (a -> b) -> (r -> a) -> (r -> b)
  fmap f g = (\x -> f(g(x)))
  fmap = (.)

  pure :: a -> (r -> a)
  pure x = (\_ -> x)

  (<*>) :: (r -> (a -> b)) -> (r -> a) -> (r -> b)
  f <*> g = \x -> f x (g x)

instance Monad ((->) r) where
  return :: r -> (r -> a)
  return x = \_ -> x

  (>>=) :: (r -> a) -> (a -> (r -> b)) -> r -> b
  h >>= fn = \w -> fn (h w) w
```


### ZipList {#ziplist}

```haskell
instance Applicative ZipList where
  fmap :: (a -> b) -> ZipList a -> ZipList b

  pure :: a -> ZipList [a]
  pure x = ZipList (repeat x)

  (<*>) :: ZipList[a -> b] -> ZipList [a] -> ZipList [b]
  ZipList fs <*> ZipList xs = ZipList (zipWith (\f x -> f x) fs xs)

  liftA2 :: (a -> b -> c) -> ZipList a -> ZipList b -> ZipList c
    = fn <$> ZipList xs <*> ZipList ys
    = fmap fn ZipList xs <*> ZipList ys
    = ZipList (map fn xs) <*> ZipList ys
    = ZipList (zipWith (\f x -> f x) (map fn xs) ys)

  sequenceA :: [ZipList a] -> ZipList [a]
  sequenceA ((ZipList xs):zls) = ZipList (map (:) xs) <*> sequenceA zls
  sequenceA = foldr (liftA2 (:)) (ZipLits [])
```


## Laws {#laws}

最后，以下是 (applicative) functor/monad 必须遵守的法则，Haskell 编译器无法自动检查。需要手动验证，这里就不展开了。

```haskell
-- functor
fmap id == id
fmap (f . g) = fmap f . fmap g

-- applicative functor
pure id <*> v = v
pure (.) <*> u <*> v <*> w = u <*> (v <*> w)
pure fn <*> pure x = pure (fn x)
u <*> pure y = pure ($ y) <*> u

-- Monad
return x >> f = f x
m >>= return = m
(m >>= f) >> g = m >>= (\x -> f x >>= g)
```


## <span class="org-todo todo TODO">TODO</span> Is Promise of Javascript a Monad? {#is-promise-of-javascript-a-monad}
