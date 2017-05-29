+++
date = "2017-04-11T21:33:09+09:00"
description = ""
draft = false
tags = ["rust"]
title = "Rust面白そう"
updated = "2017-04-11T21:33:09+09:00"

+++

何となく[The Rust Programming Language][rust-book]を読んでみたらかなり面白かった。  
ので気になったとこをメモしておくけど、ざっくりとしか理解してない。  
以下は全て「たぶん」。

[rust-book]: https://doc.rust-lang.org/book

## Ownership, borrowing, lifetime

変数、特にポインタ値が無効な参照を持ってしまう事をコンパイルレベルで防ぐ仕組み。  
ある値の実体を参照する変数は常に1つしか存在できない。
ある変数を別の変数に代入した後で元の変数を参照するとエラーになる (Ownership)。
1つの値を複数から参照したい場合はreference (`&x`) を使う。
referenceはある値のOwnershipを借用 (borrowing) し、その値へのポインタを持つ。
referenceのスコープは元の値のスコープと同じかそれ以下である必要があり、それが満たされていない場合はコンパイルエラーになる。

## Box, reference (&)

ref: [stack overflow](https://stackoverflow.com/questions/27305585/difference-between-pass-by-reference-and-by-box)

reference は常に借用という概念とともにあるっぽい。
要はある値への参照 (ポインタ) であり、参照先の実体よりもライフタイムが長い場合、コンパイル時点でエラーになる。
これにより、参照先が無効となる事を防ぐ。あくまで借用なので、参照先のメモリの所有権を持つ変数が別に存在する。  
`Box`もポインタだが、その参照先は常にheapに作られる (stackは関数内のローカルな値の格納先。heapは永続的) 。
そしてreferenceとは違い借用ではないため、`Box`値を引数で渡すと所有権も移る。
上記stack overflowだと、`Box`値は主に再帰的な構造くらいでしか使わないとある。
ただのポインタ型だから、再帰的なデータのように必要なメモリ量が未知な場合に使えるという事?

- `Box`のreferenceも作れる。referenceのBoxも作れる (たぶん)
- `'static`なreferenceもheapに作られる?

通常はBoxやreferenceを通してポインタを扱うが、row pointerも一応使えるっぽい。ただし`unsafe`な作業になる。

## Misc

- No garbage collector!
- [Cargo][cargo]というパッケージマネージャや[rustup][rustup]というインストーラが最初からついてて便利。
- struct, trait はあるがclass, extendはない。
    - 実際traitがあれば継承は不要。むしろ継承こそtraitのような仕組みを簡略化したもの、という気がする。
    - trait は いわゆる interface で、あるstructは任意の数のtraitを実装できる
    - メソッドを持たないtraitも定義可能 (Goのダックタイピングなinterfaceとの違い)
- モジュール化の仕組みが柔軟。
- `where`のおかげで複雑なGenericsが書きやすそう。
- 他の言語と違い、同名の変数を何回でも宣言できる。mutableな変数を`let`で再宣言してイミュータブルにしたりできる。
  型を変換しただけの値に`id_str`とか別名をつける必要がなくなるのは地味に便利かも。
- テストをテスト対象の関数と同じファイルに書くのが習慣? 
  関数が小さければいいけど、大きくなってきたらファイルは分けたい気がする。

[cargo]: http://doc.crates.io/
[rustup]: https://www.rustup.rs/

#### Trait

基本的に他の言語でいうTraitと同じ機能っぽい。
メソッドのシグネチャやデフォルトメソッドを含むインターフェイスを定義し、structがそれを実装する。
あるTraitが、それを実装するstructに対し、別のTraitの実装を要求する事もできる。
ちょっと特徴的だと思ったのは、Genericsを持ったTraitの実装を、Genericsの型引数ごとに別個に定義できる点。
例えば`foo`メソッドを持つ `Some<T>`というTraitがあったとして、`T`が`String`の場合と`i32`の場合で全然別の実装を提供できる。

#### Iterator

RustにおけるCollection操作は、Iteratorパターンで実装される
(Iteratorは次の値が入っているかもしれない`Option`値を返す`next`メソッドを実装する)。
例えば`map`や`filter`といった関数は普通、Collectionを新たなCollectionに変換する関数だが、
RustではIteratorを新しいIteratorに変換する関数になっている (`iterator adapter`)。
そのため、一般的なCollection操作でありがちな、
`a.map(..).filter(..).map(..)`のように書くと中間Collectionが毎回生成されてしまい無駄、という問題が起きない。
上記はあくまでマッピングやフィルタを行う新しいIteratorを作るだけで、
`collect()`メソッドなどが結果値を要求するまで実際の操作は遅延される。
Haskellならデフォルトで非正格だし、ScalaはStreamのように非正格評価をするAPIを用意して同種の問題を解決しているけど、
RustはそれをIteratorパターンで解決している。
もちろん無限リストも簡単に作れる。その場合、`next()`が永遠に`None`を返さないIteratorになる。
ちなみに、`for`文もIteratorによるループの糖衣構文みたいなものらしい。
