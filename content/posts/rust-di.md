+++
date = "2018-08-26T16:31:11+09:00"
description = ""
draft = true
tags = ["rust"]
title = "Rust の decoupled DI"
updated = "2018-08-26T16:31:11+09:00"
+++

## 概要

- Rust で良い感じに DI したい。
- <https://keens.github.io/blog/2017/12/01/rustnodi/> の方法が良い感じ。
- この記事で紹介されている方法を自分なりに整理し、もうひと工夫してみる。

## はじめに

最近 Rust に触るのがすごく楽しいです。
今は試しに Rust で Web アプリを書いているところで、
その際 Rust ではどんな風に DI するのが良いかが気になりました。
そこで調べてみると大いに参考になる以下の記事が見つかったので、
これをベースに自分なりの DI 方法を考えてみました。

<https://keens.github.io/blog/2017/12/01/rustnodi/>

もちろんこの記事にもあるように、依存関係を抽象する方法はいくつもありうるのですが、
今回書く内容はこの記事で紹介されている DI 方法を前提としています。

## ベースとなる DI 方法

まず前述の記事で紹介されている DI 方法を自分なりに整理してみます。要点をまとめると、

- 1. メインのロジックは trait のメソッドにデフォルト実装として持たせる。
- 2. 依存関係の実体へアクセスする getter メソッドは実装なしの trait メソッドとしておく。
- 3. (必要なら) 1つの具体的な型に trait の実装を集約し、所有権の問題を回避する (後述)。

こうすると 2 の getter メソッドだけ trait を`impl`する側で実装できるので、
テストの際はこの部分をモックに差し替えて 1 の trait のデフォルト実装をテストできます。

例えば次のコードは、ある API を提供するための trait 定義の例です。

```rust
/// 何かしらの API クライアントを提供する。
trait HaveClient {
    type Client: Client;
    fn client(&self) -> &Self::Client;
}

/// API を用いて Foo に関する処理を提供する。
trait FooApi: HaveClient {
    // (Future などは無し)
    fn fetch_foo(&self, id: u32) -> Result<Foo> {
        let res = self.client().fetch(format!("/foo/{}", id))?;
        Ok(Foo::from(res))
    }
}
```

`FooApi` trait は`fetch_foo`のデフォルト実装を提供していますが、
実際の client を取得する部分に関しては`HaveClient` trait に依存しています。
そのためテストではこの`HaveClient`の実装を差し替える事ができます。

```rust
#[test]
fn test_foo_api() {
    struct Mock {
        client: MockClient
    }

    impl HaveClient for Mock {
        type Client = MockClient;
        fn client(&self) -> &Self::Client {
            &self.client
        }
    }

    impl FooApi for Mock {}

    let m = Mock {
        client: MockClient {},
    };
    let expected = Foo { id: 1, is_foo: true };
    assert_eq!(m.fetch_foo(1).unwrap(), expected);
}
```

次にモックではない方の実装はどうなるでしょうか。
こちらでも`Client`を実装する struct と
`HaveClient`および`FooApi`を実装する struct の2つがあれば良さそうです。
しかしそれぞれに具体的な型を用意する代わりに、
`HaveClient`と`Client`を1つの struct に実装してしまう事もできます。

```rust
struct Hub { /* some state */ }

impl Hub {
    /* ... */
}

impl Client for Hub {
    /* ... */
}

impl HaveClient for Hub {
    type Client = Self;
    fn client(&self) -> &Self::Client {
        &self
    }
}

impl FooApi for Hub {}
```

1つの struct で実装する方法には、所有権に悩まされないメリットがあります。
例えば`FooApi`の他に`BarApi`など同じ client を使うサービスが他にもあり、
かつそれぞれが同じ client のインスタンスを使いたい場合 (キャッシュを共有したい、 API の呼び出し制限管理をしたいなど)、
どこかにある client の参照をそれぞれが持ったり、client を`Rc`/`Arc`で持つ必要が出てきます。
しかし`Client`も`FooApi`も`BarApi`も全て同じ struct (`Hub`) に実装してしまえば、
そもそも同じインスタンスなので所有権について考えなくても良くなるわけです。

そしてサービスを使う側は具体的な型である`Hub`について知る必要はなく、
欲しい型だけをジェネリクスで要求すれば済みます。

```rust
fn use_foo_api<T: FooApi>(api: &T) -> Result<Something> {
    let foo = api.fetch_foo(opts.foo_id)?;
    // ...
}

fn main() {
    let hub = Hub::create();
    // ...
    let result = use_foo_api(&hub);
    // ...
}
```

以上が trait のデフォルト実装を利用した DI の方法です。
わかりやすいし、前述のように`FooApi`をテストするのも簡単です。

## 問題: trait の依存ツリーのせいでテストがしづらい

ただ、このままだと1つ面倒な点がありました。
それは`FooApi`**に**依存しているコードをテストしたい場合です。

例えば前述の例でいう`use_foo_api`関数のテストを考えてみます。
幸い引数はジェネリクスになってるので、`Hub`ではなくても`FooApi`を実装した型の値なら何でも渡せます。
それなら適当な struct を定義して`FooApi`のモックを実装すればおしまいかと思いきや、
これは上手くいきません。

```rust
#[test]
fn test_use_foo_api() {
    struct MockApi {}

    // COMPILE ERROR!
    impl FooApi for MockApi {
       fn fetch_foo(&self) -> Result<Foo> {
           /* some fake implementation */
       }
    }

    let m = MockApi {};
    let result = use_foo_api(&m).unwrap();
}
```

なぜなら`FooApi`は、それを実装する型が`HaveClient`も実装している事を要求しているからです。

```rust
trait FooApi: HaveClient {
    /* ... */
}
```

したがって、`FooApi`を実装するためにはまず`HaveClient`を実装しなければいけません。

しかし、これはあまりやりたくありません。
`use_foo_api`のテストでは`FooApi`自体をモックに差し替えるのだから client は必要ないはずです。
我慢して実装するにしても、依存するサービスが増えれば連鎖的にそれらの依存先 (の依存先の依存先...) も全て実装する必要がある事になります。
不便だし、使う側に trait 間の依存関係が漏れてしまうのがイマイチです。

サービスを使う側には内部的な依存関係を隠して、インターフェイスだけを提供する事はできないでしょうか？

## 解決策

そこでこれを実現するため、各サービス trait の定義と実装にもうひと手間加えてみます。

まず次のように、`FooApi`のメソッドと、`FooApi`が依存する trait の定義を別々の trait にしてみます。

```rust
// 純粋にインターフェイスだけを定義する trait
trait IsFooApi {
    fn fetch_foo(&self, id: u32) -> Result<Foo>;
}

// FooApi の依存関係を定義する trait
trait FooApi: HaveClient {}
```

先程と違い、`IsFooApi`はデフォルトの実装を持っていない点に注意してください。
そしてサービスを使う側は、`FooApi`の代わりに`IsFooApi`を要求するようにしてみます。

```rust
fn use_foo_api<T: IsFooApi>(api: &T) -> Result<Something> {
    let foo = api.fetch_foo(100)?;
    // ...
}
```

最後に、`FooApi`を実装する全ての型が自動的に`IsFooApi`を実装するようにします。
`IsFooApi`の実装はここで行います。

```rust
impl<T: FooApi> IsFooApi for T {
    fn fetch_foo(&self, id: u32) -> Result<Foo> {
        let res = self.client().fetch(format!("/foo/{}", id))?;
        Ok(Foo::from(res))
    }
}
```

`Hub`の実装は先程と変わりません。

```rust
struct Hub { /* some state */ }

impl Hub { /* ... */ }

impl Client for Hub { /* ... */ }

impl HaveClient for Hub { /* ... */ }

impl FooApi for Hub {}
```

するとどうなるでしょう？

`Hub`は`FooApi`を実装しているので、自動的に`IsFooApi`も実装します。
そのため先程と同じように`use_foo_api`関数には`Hub`を渡す事ができます。
一方`use_foo_api`の方は`FooApi`ではなく`IsFooApi`のみを要求しています。
こちらは`HaveClient`への依存がないので、
`use_foo_api`をテストしたければ、単純に`IsFooApi`だけを実装したモックを渡せるようになります。

```rust
#[test]
fn test_use_foo_api() {
    struct MockApi {}

    // OK!
    impl IsFooApi for MockApi {
       fn fetch_foo(&self, id: u32) -> Result<Foo> {
           /* some fake implementation */
       }
    }

    let m = MockApi {};
    let result = use_foo_api(&m).unwrap();
}
```

ポイントは以下です:

- サービスの実装を持たせる型 (例でいう`Hub`) には`FooApi`を実装する。
- サービスを使う側は`FooApi`ではなく`IsFooApi`のみを要求する。

これでインターフェイスと依存関係を分離できました。

これはサービス間の依存の疎結合化にも有効です。
例えば`FooApi`に依存する別のサービスがある場合も、
`FooApi`の代わりに`IsFooApi`を要求すれば、`FooApi`の依存関係に関しては知らずにすみます。
そしてそのサービスもまた同じ方法でインターフェイスと依存関係を別の trait にすれば、
そのサービスに更に依存するサービスもまた同じ恩恵を受けられます。

```rust
trait IsFooService {
    fn do_something(&self, args: Args) -> Result<Something>;
}

// FooService は IsFooApi に依存する。
trait FooService: IsFooApi {}

impl <T: FooService> IsFooService for T {
    fn do_something(&self, args: Args) -> Result<Something> {
        let foo = self.fetch_foo(args.foo_id)?;
        // ...
    }
}

impl FooApi for Hub {}
impl FooService for Hub {}
```

つまりこの方法で各サービスを実装していけば、
互いのインターフェイスにのみ依存してロジックを構築していく事ができるはずです。

また、ここまでの例では`IsFooApi`のように、実装する型が直接そのサービスになる方式でしたが、
実際にはサービスのインスタンスを取得する形式の方が一般的だと思うので、そちらの例も載せます。

```rust
trait IsFooApi {
    fn fetch_foo(&self, id: u32) -> Result<Foo>;
}

trait FooApi: HaveClient {}
impl<T: FooApi> IsFooApi for T { /* implement */ }

// FooApi のインスタンスを提供する trait.
// type Api の型境界は FooApi ではなく IsFooApi なので、
// HaveFooApi のモック実装は簡単にできる。
trait HaveFooApi {
    type Api: IsFooApi;
    fn foo_api(&self) -> &Self::Api;
}

trait IsFooService {
    fn do_something(&self) -> String;
}

trait FooService: HaveFooApi {}

impl<T: FooService> IsFooService for T {
    fn do_something(&self) -> String {
        let api = self.foo_api();
        let foo = api.fetch_foo(5).unwrap();
        format!("foo: {:?}", foo)
    }
}

// FooService のインスタンスを提供する trait.
trait HaveFooService {
    type Svc: IsFooService;
    fn foo_service(&self) -> &Self::Svc;
}
```

`HaveFooApi`や`HaveFooService`の実装に関しては、
内部にサービスのインスタンスを保持する必要があるので具体的な struct (or enum) が実装する形になります。

## behavior trait と implementor trait

前述した`IsFooApi`系 trait と`FooApi`系 trait について、
自分は便宜的に前者を behavior trait 、 後者を implementor trait と呼んでいます。
前者が動作のインターフェイスだけを定義するのに対し、
後者は実際の実装を依存関係の定義とともに提供するからです。

```rust
mod behavior {
    // 日付に関するメソッドを提供する。
    pub trait IsCalendar { /* ... */ }

    // スケジュール情報を提供する。
    pub trait CanFindSchedule { /* ... */ }

    // DB コネクションを提供する。
    pub trait HaveDb { /* ... */ }
}

mod implementor {
    // IsCalendar の implementor。
    pub trait Calendar: HaveDb {}
    impl<T: Calendar> IsCalendar for T { /* implement */ }

    // CanFindSchedule の implementor。
    pub trait FindSchedule: IsCalendar + HaveDb {}
    impl<T: FindSchedule> CanFindSchedule for T { /* implement */ }
}

/// サービスを使う側
mod service_user {
    struct Hub {
        pool: DbPool,
    }

    // 状態を持たない implementor は impl するだけで実装できる。
    impl Calendar for Hub {}
    impl FindSchedule for Hub {}
    // ...

    // こちらは実装が必要。
    impl HaveDb for Hub {
        /* fn db_conn(&self) -> Result<PooledConnection> {...} */
    }

    // サービスを使う側は implementor ではなく behavior を要求する。
    fn use_services<S>(svc: &S) -> Result<Something>
    where
        S: CanFindSchedule + HaveFooService,
    {
        // Use services.
    }
}
```

(このサンプルコードでは見やすさのため behavior trait と implementor trait でモジュールを分けましたが、
実際には両者は常に対応するペアになるので、
各ペアをセットでモジュールにした方が管理しやすいと思います。)

## まとめ

以下の方法で疎結合な DI を実現できました。

- インターフェイス定義と依存関係定義を別の trait にする (behavior & implementor)
- サービスを使う側は behavior trait にのみ依存する。
- サービスを実装する側は implementor trait を実装する。
- implementor trait を実装している型は必ず対応する behavior trait も実装しているので、
behavior trait を要求される場面で使用できる。
