+++
date = "2018-08-26T16:31:11+09:00"
description = ""
draft = false
tags = ["rust"]
title = "Rust で DI する時の小技"
updated = "2019-07-10T20:54:15+09:00"
+++


最近 [Rust][rust] に触るのがすごく楽しいです。
で、書いているうちに Rust ではどんな風に DI するのが良いか気になったので、
試したり調べたりした事を簡単にまとめておきます。

[rust]: https://www.rust-lang.org/en-US/

Rust の DI に関しては次の記事がとても参考になりました。  

<https://keens.github.io/blog/2017/12/01/rustnodi/>

なお、以降では DI の対象となるコンポーネントをざっくり「サービス」と呼んでいます。

## 概要

- Rust で DI するには。
    - struct ベースで DI する方法
    - trait ベースで DI する方法
- trait ベースだと他の trait への依存関係があるとモックしづらい。どうするか？

## struct ベースの DI

- サービスのインターフェイスを trait で定義し、 実装は struct で行う。
- struct はフィールドに依存関係を持ち、インスタンス生成時に実体を受け取る。
    - Java や Ruby でいうクラスなら、コンストラクタで依存関係を受け取る感じ。
- 依存を差し替えられるよう、フィールドの型にはジェネリクスを使う。
- 単純でわかりやすい。

### サンプル

以下の例では`SvcB`が`SvcA`に依存しています。
そのため`SvcB`を実装する型は`SvcA`を実装する型をフィールドに持ち、それを使います。

```rust
pub trait SvcA {
    fn a(&self) -> String;
}

pub trait SvcB {
    fn b(&self) -> String;
}

pub struct ImplA {}

impl SvcA for ImplA {
    fn a(&self) -> String {
        "impl-a".to_owned()
    }
}

pub struct ImplB<A: SvcA> {
    a: A,
}

impl<A: SvcA> SvcB for ImplB<A> {
    fn b(&self) -> String {
        format!("a: {}, b: {}", self.a.a(), "impl-b")
    }
}

#[test]
fn test_b() {
    struct MockA {}
    impl SvcA for MockA {
        fn a(&self) -> String {
            "mock-a".to_owned()
        }
    }

    let b = ImplB { a: MockA {} };
    assert_eq!(b.b(), "a: mock-a, b: impl-b");
}

pub fn use_b<B: SvcB>(b: B) -> String {
    format!("[use] {}", b.b())
}
```

### 考慮事項

- 依存が増えると型パラメータも増える。
    - 参照だけを持つ形だとライフタイムパラメータも必要になる。
- Rust には所有権があるので、複数のサービスに同じサービスのインスタンスを持たせようとすると面倒になりうる。

シンプルではありますが、依存の数だけ型パラメータを持たなければいけないのは辛いですね。

## trait ベースの DI

- trait のデフォルト実装にロジックを置く。
- trait だとフィールドは使えないので、代わりに依存する trait を要求する。
- 1つの struct に複数のサービス trait を集約して実装することもできる。

### サンプル

こちらはクラスベースの思考から離れ、オブジェクトというよりは型でロジックを分離する方法です。
以下は先程と同じ`SvcA`と`SvcB`を trait ベースで再実装したものです。

```rust
pub trait SvcA {
    fn a(&self) -> String {
        "svc-a".to_owned()
    }
}

// SvcB requires SvcA.
pub trait SvcB: SvcA {
    fn b(&self) -> String {
        format!("a: {}, b: {}", self.a(), "svc-b")
    }
}

pub struct Hub {}
impl SvcA for Hub {}
impl SvcB for Hub {}

#[test]
fn test_b() {
    struct Mock {}
    impl SvcA for Mock {
        fn a(&self) -> String {
            "mock-a".to_owned()
        }
    }
    impl SvcB for Mock {}

    let b = Mock {};
    assert_eq!(b.b(), "a: mock-a, b: svc-b");
}

pub fn use_b<B: SvcB>(b: B) -> String {
    format!("[use] {}", b.b())
}
```

### 考慮事項

- 場合によってはこっちの方がシンプル。
- trait を`impl`するだけで実装が得られる。
- trait とそれを実装する struct を 1:1 で作らなくても良くなる。
- 依存する trait をモック実装すればテストできる。
- 1つの struct に実装を集約して同じインスタンスを使い回せば、所有権に悩まされずに済むかも。

struct ベースの方法と違い、依存が増減する度に型パラメータを変更する必要がないので、
個人的にはこちらをメインに使う方が良さそうな気がしています。
ただこの方法では、1つの struct がたくさんのサービス trait を実装していく形にはなります (この例でいう`Hub`)。
この調子で他のサービスも実装していくと`Hub`がどんどん肥大化する感じはしますが、
実際にはそれをそのまま使うわけではなく、`use_b`での使用例のようにその時必要な型としてだけ使うようにすれば、
その点はあまり問題なさそうに思えます。

ちなみにもし複数のサービスが同名のメソッドを持っていても、呼び分けるのは簡単なので問題ありません。

```rust
trait A1 {
    fn a(&self) -> i32 { 0 }
}
trait A2 {
    fn a(&self) -> i32 { 1 }
}

struct Hub {}
impl A1 for Hub {}
impl A2 for Hub {}

fn main() {
    let hub = Hub {};

    // hub.a() と同じ
    let a1 = A1::a(&hub);
    let a2 = A2::a(&hub);
    println!("{}, {}", a1, a2);
}
```

## trait ベースの方法の問題点

しかし、 trait ベースの方法には1つ問題があります。  
サービス自体をテストする場合は、依存する trait をモックに差し替えればすみました。
ではサービスを使う側のテストはどうでしょうか？

例えば先程の例の`use_b`を、実際の`Hub`ではなくモックを使ってテストする事を考えてみます。幸い引数の型はジェネリクスになっているので、`SvcB`を実装している型の値であれば何でも渡せます。
それなら適当な struct を定義して`SvcB`を実装すれば良いだけかと思いきや、これは上手くいきません。

```rust
pub fn use_b<B: SvcB>(b: B) -> String {
    format!("[use] {}", b.b())
}

#[test]
fn test_use_b_by_mock() {
    struct Mock {}

    // COMPILE ERROR
    impl SvcB for Mock {
        fn b(&self) -> String {
            "mock-b".to_owned()
        }
    }
    
    assert_eq!(use_b(Mock {}), "[use] mock-b");
}
```

というのも`SvcB`は、自身を実装する型が`SvcA`も実装している事を要求するからです。

```rust
trait SvcB: SvcA {
    /* ... */
}
```

しかしこれは困ります。モック実装で`SvcB`のインターフェイスを満たすだけなら`SvcA`は必要ないはずです。
それに例えば`SvcA`が更にまた別の trait に依存していた場合、
それもまた同じように実装しなければいけません。
本来なら`use_b`は`SvcB`のインターフェイスだけ知っていれば良いのに、
いざテストを書こうとすると trait 間の依存関係まで漏れてしまいます。

## 解決策

これはインターフェイスの定義と依存関係の定義を別の trait にすると解決できます。

先程の例で続けると、まず`SvcA`と`SvcB`の定義にもうひと手間加えて以下のようにしてみます。

```rust
// インターフェイス
pub trait IsSvcA {
    fn a(&self) -> String;
}

// 依存関係
pub trait SvcA {}

// インターフェイス
pub trait IsSvcB {
    fn b(&self) -> String;
}

// 依存関係
pub trait SvcB: IsSvcA {}
```

先程とは違い、`IsSvcA`と`IsSvcB`はインターフェイスだけを定義していて、
デフォルト実装はなくなっています。
また`SvcB`は`SvcA`ではなく`IsSvcA`に依存しています。

ではデフォルト実装ではなくなったロジックはどこに置くのかというと、
次のようにします。

```rust
impl<T: SvcA> IsSvcA for T {
    fn a(&self) -> String {
        "svc-a".to_owned()
    }
}

impl<T: SvcB> IsSvcB for T {
    fn b(&self) -> String {
        format!("a: {}, b: {}", self.a(), "svc-b")
    }
}
```

これにより、`SvcX`を`impl`すると自動的に`IsSvcX`も`impl`され、
メソッドの実装も与えられます。

そして最後に、サービスを使う側は`SvcB`ではなく`IsSvcB`に依存するようにします。

```rust
pub fn use_b<B: IsSvcB>(b: B) -> String {
    format!("[use] {}", b.b());
}
```

するとどうでしょう。`use_b`が依存する`IsSvcB`はもう依存を持たなくなったので、
`IsSvcB`を単体でモックに実装してテストできるようになります。

```rust
#[test]
fn test_use_b_by_mock() {
    struct Mock {}

    // OK
    impl IsSvcB for Mock {
        fn b(&self) -> String {
            "mock-b".to_owned()
        }
    }
    
    assert_eq!(use_b(Mock {}), "[use] mock-b");
}
```

ではモックではない実装の方はどうなるでしょうか。
こちらは先程と同じように`SvcB`を実装すれば自動的に`IsSvcB`も実装されるので、
そのまま`use_b`に渡す事ができます。
ただし`SvcB`は`IsSvcA`に依存しているので、`SvcA`も忘れずに実装します。

```rust
struct Hub {}
impl SvcA for Hub {}
impl SvcB for Hub {}

#[test]
fn test_use_b() {
    let svc = Hub {};
    assert_eq!("[use] a: svc-a, b: svc-b", use_b(svc));
}
```

つまり先程は`SvcB`という 1 つの trait だったものを、
実装する側と使う側向けの 2 つの trait に分割する事で、
サービスの依存関係に関しては実装する側のみが知るようにしています。
この方法であれば trait ベースでサービスを定義しつつ、
サービスを使う側はインターフェイスにだけ依存できるようになります。

ポイントは以下です:

- サービスの実装を持たせる型 (例でいう`Hub`) には`SvcB`を実装する。
- サービスを使う側は`SvcB`ではなく`IsSvcB`のみを要求する。

ここまでのコード: [Playground](https://play.rust-lang.org/?gist=fcf761189223f2e6480271c2186abb69&version=stable&mode=debug&edition=2015)

## インターフェイスと依存関係定義の分離

先程の`SvcB`のように、あるサービスがまた他のサービスを使う側になる事もあるため、
インターフェイスと依存関係定義の分離は、サービス同士を疎結合に保つのにも有効です。
また先程までの例では1つの struct (`Hub`) が複数のサービスとして直接振る舞う形でしたが、
そうではなく getter でサービスのインスタンスを取得して使うようなパターンもありえます。
その場合は以下のような感じでしょうか。

```rust
pub trait IsSvcA {
    fn a(&self) -> String;
}

pub trait SvcA {}

impl<T: SvcA> IsSvcA for T {
    fn a(&self) -> String {
        "svc-a".to_owned()
    }
}

// Provide A service.
pub trait HaveSvcA {
    type A: IsSvcA; // Not SvcA
    fn get_svc_a(&self) -> &Self::A;
}

pub trait IsSvcB {
    fn b(&self) -> String;
}

// SvcB depends on HaveSvcA instead of IsSvcA.
pub trait SvcB: HaveSvcA {}

impl<T: SvcB> IsSvcB for T {
    fn b(&self) -> String {
        let a = self.get_svc_a();
        format!("a: {}, b: {}", a.a(), "svc-b")
    }
}

// Provide B service.
pub trait HaveSvcB {
    type B: IsSvcB; // Not SvcB
    fn get_svc_b(&self) -> &Self::B;
}

pub fn use_b<S: HaveSvcB>(svc: S) -> String {
    let b = svc.get_svc_b();
    format!("[use] {}", b.b())
}
```

`HaveSvcB`の associated type `B`が要求する型は`SvcB`ではなく`IsSvcB`になっています。
こうする事で、`HaveSvcB`を使う他のコードは`IsSvcB`にのみ依存する形になり、
`SvcB`自体の依存関係 (`HaveSvcA`) については知らなくてすみます。

なお`HaveSvcA`や`HaveSvcB`は実際のインスタンスを返さないといけないので、
これらは具体的な型の方で実装します。

```rust
pub struct Hub {}

impl SvcA for Hub {}
impl SvcB for Hub {}

impl HaveSvcA for Hub {
    type A = Self;
    fn get_svc_a(&self) -> &Self::A {
        &self
    }
}

impl HaveSvcB for Hub {
    type B = Self;
    fn get_svc_b(&self) -> &Self::B {
        &self
    }
}

#[test]
fn test_use_b() {
    let svc = Hub {};
    assert_eq!(use_b(svc), "[use] a: svc-a, b: svc-b");
}
```

[Playground](https://play.rust-lang.org/?gist=701cdbd3d07ec6a4f2daf19559d62abd&version=stable&mode=debug&edition=2015)

ちなみにここでいう`IsX`や`HaveX`系 trait と `SvcX`系 trait について、
自分は便宜的に前者を interface trait, 後者を implementation trait と呼んでいます。
前者が動作のインターフェイスだけを定義するのに対し、
後者は実際の実装を依存関係の定義とともに提供するというニュアンスです。

## まとめ

- 単純にやるなら struct ベースでもいい。
- trait ベースでやるとより柔軟。
- 以下のようにすると、 trait ベースの DI をより疎結合にできる。
    - インターフェイス定義と依存関係定義を別の trait にする (interface trait, implementation trait)。
    - implementation trait のみが実際に必要となる依存関係を持つ。
    - implementation trait を実装すると interface trait も自動で実装されるようにする。
    - サービスを使う側は interface trait にのみ依存する。
    - サービスを提供する側は implementation trait を実装する。

### 追記

実際に趣味コードでこの方法を使ってみて思ったのですが、 trait のネーミングは逆の方が良いかもしれません。

今までの例:

- interface trait - `IsSvcA`
- implementation trait - `SvcA`

逆にすると:

- interface trait - `SvcA`
- implementation trait - `IsSvcA`

というのも外部で使われるのは interface trait の方なので、このネーミングルールで公開してしまうと、
`Is`始まりという不自然なネーミングが内部の設計
(interface trait / implementation trait という分離) を外に匂わせる形になるからです。
であれば、外部で使う方をより普通の`SvcA`のようにし、
内部の implementation trait を`IsSvcA`のようなネーミングルールで統一する方が良いのではないか、と思いました。
逆になると`Is`という接頭ルールはあまり直感的じゃない気もしますが。

