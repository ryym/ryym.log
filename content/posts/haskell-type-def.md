+++
date = "2017-01-11T21:55:00+09:00"
description = "Haskellの型定義用語はOOP出身だと最初混乱する。でもある程度は対応づけられそう。"
draft = false
tags = ["haskell"]
title = "Haskellの型定義をちょっと整理"
updated = "2017-01-11T21:55:00+09:00"

+++

- `class`, `data`, `type`
- 型クラス、型コンストラクタ、値コンストラクタ、インスタンス

---

クラスやインスタンスといった単語はJava出身の身にも馴染み深いが、Haskell (あるいは関数型言語一般?) におけるそれらの用語の使われ方は、
一般的なOOPにおけるそれとは全く違う。  
以下はあまりまとまりのないメモ。

```haskell
-- Type constructor
data Maybe = Nothing | Just a
data Tree a = EmptyTree | Node a (Tree a) (Tree a) deriving (Show)

-- Type synonym (alias)
type Code = String

-- Class
-- Eq, Ord, Show, Read, ...
class YesNo where
   yesno :: a -> bool

-- Type can be an instance of a class
instance YesNo (Maybe a) where
  yesno (Just _) = true
  yesno Nothing = false


yesno $ Just 0 -- True
```

## 型コンストラクタ

**型コンストラクタ**は0個以上の型引数を受け取り、実際の値に割り当て可能な型 (=具体型) を生成する。
その型を持つ値は、**値コンストラクタ**を使って生成できる。値コンストラクタは型定義の右辺で定義する。
上記の例でいえば、`Maybe`が型コンストラクタで、`Just a`が値コンストラクタ。`Just 2`のようにすれば、`Maybe Int`型の値が作れる。
逆にいうと`Maybe`型の値、というものは存在しない。

## 型シノニム

**型シノニム** (`type`)は単に型の別名を定義できるだけ。
ちなみに、Haskellの世界では型も関数なので、`type MapInt = Map k`のように、型引数を部分適用する事もできる!  
また`type Optional a = Maybe a`のように`type`で型引数を受け取る事もできる。

## 型クラス

**型クラス**は、クラスとそのクラスに適用可能な関数のシグネチャを定義する。
関数はシグネチャだけが定義され、実体はその型クラスのインスタンスとなる型がそれぞれ実装する。
これにより、ある型クラスを実装した型の値に対しては、それがどんな型であってもその型クラスに定義された関数を使用できる。
例えば`==`のような等値比較もHaskellでは関数だが、これをIntやStringといった色々な値に使えるのは、それらが皆`Eq`クラスのインスタンスだから。
上記の例では、定義したYesNoクラスをMaybeに実装させ、任意の`Maybe a`型の値をBool値に変換できるようにしている。
Maybeのみならず、Intなど既存の型もYesNoのインスタンスにして`yesno`関数を実装すれば、様々な値をBool値に変換できるようになる。
`Eq`や`Show`などの一般的な型クラスは`deriving (Eq, Show)`のように書くだけで実装を自動抽出してくれる。

### OOPでいうと..?

このように考えると、型クラスはJavaでいうインターフェイスのようなもの、と言える気がする。
その例えでいうと`instance Class Type where..`はJavaの`implements Interface`に相当する事になる。

- 型クラス - Interfaceっぽい
- 型コンストラクタ・値コンストラクタ - ちょっとクラスっぽい

でもHaskellの型の世界 ([代数的データ型][wiki-algebraic-data-type]) の方がずっと柔軟だし、パターンマッチも便利。  
それにWikiにあるように、OOPにおけるコンストラクタはいわば関数なので更なる簡約が可能だが、
値コンストラクタを使った式はまさしく値を表す式であり、それ以上簡約できない、という点も面白い。

[wiki-algebraic-data-type]: https://ja.wikipedia.org/wiki/%E4%BB%A3%E6%95%B0%E7%9A%84%E3%83%87%E3%83%BC%E3%82%BF%E5%9E%8B

### 型クラスを型パラメータの条件にする

例えば`Show`を実装した任意の型を受け取りたい場合は以下のように書く。Genericsの上限境界みたいなものか。

```haskell
-- Showクラスを実装した型のみ受け取れる
tellValue :: (Show a) => a -> string
tellValue v = "The value is " ++ show v
```

### 型クラスがとる型引数について

上記のYesNoでは、型クラスは型引数に具体型を要求している。つまり、`instance YesNo (Maybe a) where..`とは書けても、`instance YesNo Maybe where..`とは書けない。これは単にYesNoクラスの性質の問題で、例えばFunctor型クラスは、「1つの型引数を取る型コンストラクタ」を型引数として受け取る。

```haskell
-- f は値ではなく型コンストラクタ
class Functor f where
  fmap :: (a -> b) -> f a -> f b
```

`fmap`の型定義を見ればわかるように、f型は1つの型引数を受け取って初めて具体型になる。だからインスタンスの定義も以下のようになる。

```haskell
instance Funcotr Maybe where
  fmap f (Just x) = Just (f x)
  fmap f  Nothing = Nothing

instance Functor [] where
  fmap = map
```

Maybeのインスタンス定義を見ると、Justの中身の型については何も定義されていない。
2番目はリストのインスタンス定義で、`[]`は空リストではなくリストの型コンストラクタ。リストの生成は普通`[a]`と書くが、これは`[] a`の糖衣構文。
つまり値コンストラクタも`[]`と定義されている (値コンストラクタが1種類しかない場合、わかりやすいように型コンストラクタと同名にするのがマナーらしい)。
