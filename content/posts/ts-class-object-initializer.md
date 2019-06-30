+++
date = "2019-06-29T19:05:39+09:00"
description = ""
draft = false
tags = ["typescript"]
title = "TypeScript のクラスを Object 形式で初期化したい"
updated = "2019-06-29T19:05:39+09:00"
languages = ["typescript"]
+++

## TL;DR

```typescript
// こうじゃなくて...
const alice = new User(1, 38, 'alice', 'alice@example.com');

// こうしたい。
const alice = new User({
  id: 1,
  age: 38,
  name: 'alice',
  email: 'alice@example.com',
});

// けど言語レベルでは未サポート。こんな感じの方法が現実的か。
class User {
  id!: number;
  age!: number;
  name!: string;
  email!: string;
  createdAt: Date = new Date();

  constructor(props: SemiPartial<Fields<User>, 'createdAt'>) {
    Object.assign(this, props);
  }
}
```

## クラスを Object 形式で初期化したい

[TypeScript][typescript] のクラスには [Parameter properties][ts-parameter-properties] という記法があります。
これを使うと、クラスのプロパティをコンストラクタの引数と一緒に定義できて便利です。

[typescript]: https://www.typescriptlang.org/
[ts-parameter-properties]: https://www.typescriptlang.org/docs/handbook/classes.html#parameter-properties

```typescript
class User {
  // id, number はクラスのプロパティになる。
  constructor(readonly id: number, public name: string) {}
}

const alice = new User(100, 'alice');
console.log(alice.id, alice.name);
```

しかしこれは、たくさんのプロパティを初期化するのには向いていません。
使う側は引数の順番を覚えないといけないし、同じ型のプロパティが複数あると順番を間違えても気づきにくいです。

```typescript
class User {
  constructor(
    id: number,
    email: string,
    givenName: string,
    familyName: string,
  ) {}
}

// Wrong argument order!
const harry = new User(100, 'harry', 'potter', 'harry@example.com');
console.log(harry.familyName);
//=> "harry@example.com"
```

そのためこういう時には引数に Object を使い、値とプロパティ名を対応付けつつ初期化したいところです。
しかし Parameter properties は Object 形式には対応していないため、単純にやると途端にコード量が増えてしまいます。

```typescript
class User {
  id: number;
  email: string;
  givenName: string;
  familyName: string;

  constructor(
    props: {
      id: number,
      email: string,
      givenName: string,
      familyName: string,
    }
  ) {
    this.id = props.id;
    this.email = props.email;
    this.givenName = props.givenName;
    this.familyName = props.familyName;
  }
}
```

これをもう少し簡単に実現する方法はないものでしょうか。

同じ要望を持つ人は少なくないらしく、 TypeScript のリポジトリにはこの件に関する Issue が上がっています。

https://github.com/microsoft/TypeScript/issues/3895

クローズされているので対応される可能性は低そうですが、いくつかのワークアラウンドが紹介されています。
ここではこの Issue で紹介されている方法を踏まえつつ、現時点の TypeScript でやる場合の方法を考慮事項とともにまとめます。

- TypeScript のバージョン: 3.5.2

## 試み 1. 動的に値を代入する

上記のコードでは、プロパティの定義、引数の定義、値の代入で計3回もプロパティを列挙する必要があります。
まずは[`Object.assign`][object-assign]を使って値の代入だけでも省略してみましょう。

[object-assign]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign

```typescript
// 簡潔さのため、以降の例ではプロパティを id, name のみとします。
class User {
  id: number;
  name: string;

  constructor(
    props: {
      id: number,
      name: string;
    }
  ) {
    // これ1行で済む。
    Object.assign(this, props);
  }
}

const alice = new User({id: 1, name: 'alice'});
```

これでプロパティが増減しても、値の代入部分は書き換えずに済むようになりました。

ところが、この方法ではコンパイルが通らないケースがあります。
それはコンパイラオプションで[`strictPropertyInitialization`][strict-class-initialization]が有効になっている場合です。

[strict-class-initialization]: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-7.html#strict-class-initialization

### `strictPropertyInitialization`の考慮

このオプションを有効にすると、プロパティの初期化が正しく行われている事をコンパイラがチェックしてくれます。
そのため、プロパティへの直接的な代入がない上記のコードではエラーになってしまいます。
コンパイラが`Object.assign`を認識してくれれば良いのですが、現状ではされません。

しかしこのためだけにコンパイラの型チェックをゆるめるのは避けたいところです。
それをせずにこのエラーを回避するには、プロパティに`!`を追加してコンパイラにチェック不要である事を示す必要があります。

```typescript
class User {
  id!: number;
  name!: string;

  constructor(props: {/* omit */}) {
    Object.assign(this, props);
  }
}
```

ちょっとダサいですが、`constructor`の引数さえ正しくチェックされればプロパティの初期化し忘れは防げるはずなので、
実質的には問題ないでしょう。

とはいえ、似たようなプロパティ定義をクラスと constructor で繰り返すのはやはりコードを書く・読むコストを増やします。
そこで constructor の記述を省略するために、次の方法が思いつきます。

## 試み 2. 引数の型にクラスを使う

```typescript
class User {
  id!: number;
  name!: string;

  constructor(props: User) {
    Object.assign(this, props)
  }
}

const alice = new User({id: 1, name: 'alice'});
```

単純にコンストラクタの引数の型に自分自身を使っています。
これで引数型も簡潔になるし、ちゃんとコンパイルは通るものの、この方法はこの例くらいシンプルなケースでしか使えません。
以下の大きな問題があるからです:

- `User`がメソッドを持っていると、`props`もそれを持つ事を要求される。
- private なプロパティがあるとコンパイルできなくなる。

引数として`User`型の値を渡す必要があるので、`User`がメソッドを持っていたら当然それらも`props`に要求されてしまいます。
更に問題なのは private プロパティで、こちらもやはり要求されます。
その上通常の Object には private プロパティを定義できないので、 Object 形式で渡す事は実質不可能になります。

```typescript
class User {
  id!: number;
  private name!: string;

  constructor(props: User) {
    Object.assign(this, props);
  }
}

// Error: 'name' is missing
new User({id: 1});

// Error: 'name' is private in type 'User'
new User({id: 1, name: 'alice'});
```

これは実用的ではありません。

## 試み 3. メソッドと private プロパティを除外する

前述の問題を回避するために、メソッドと private プロパティを除外した型を引数で受け取るようにしましょう。
[Mapped Types][mapped-types] と [Conditional Types][conditional-types] を使えば以下のように書けます:

```typescript
// User からメソッド・ private プロパティを除いた型。
type UserFields = Fields<User>;

// T の FieldNames のみの Object 型を作る。
type Fields<T> = { [P in FieldNames<T>]: T[P] };

// T のメソッド (正確には値が関数であるプロパティ) 以外の名前のみを列挙する。
type FieldNames<T> = {
  [P in keyof T]: T[P] extends (...args: any[]) => any ? never : P
}[keyof T];
```

[mapped-types]: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-1.html#mapped-types
[conditional-types]: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-8.html#conditional-types

このコードではメソッドの除外しかしていないように見えますが、
`keyof`で列挙されるクラスのプロパティに private なものは含まれないので、
`keyof T`した時点で private プロパティも除外されています。

これでやりたい事が実現できそうです。

```typescript
class User {
  id!: number;
  name!: string;
  private unused: boolean;

  constructor(props: Fields<User>) {
    Object.assign(this, props);
    this.unused = false;
  }

  greet(): string {
    return `Hi, I'm ${this.name}.`;
  }
}

// OK
const alice = new User({id: 1, name: 'alice'});
```

[Playground](https://www.typescriptlang.org/play/#code/C4TwDgpgBAYglhANgEwM4B4AqA+KBeKAbygG0AFKOAO1gRQDkBDAWwgxwF0AuKTcjqAF8A3AChQkWkmRNW7XAUKiopCtSgBrCCAD2AM17de-KBAAewCFTRQAFADpHjAE4BzVD0ZUQJDgEp8XC8QKAB+KCoIADcIZygeMlFBEi1dA0wOMVEAY0RGVFQoAFVUWKJlSmQAQh4qAFdmACNYsRUqFggaqFRgZ2pXVqgwPqjGSyg6+tLkHkadHUQILyyVbJ0qHuc67OAdZ1thnTAPKRQMEtjsAKUVFQB5RoArCB37fNQ4VypbYAALOFQABohs4jqg-IMVH8AfZJnVpvgoHpGIhSoNBKIKq5nBAIMBbH4eJt+uVblAccA6s4aAADAAScGBAEkAOTMKAAEkI0NQ9narEE9hp6KSmIA9GKoHcANI5dY9KAouDZaAESIAd2KpX2hDgMygAEZgfyIDwWUqVSzBBCgA)

ところが、これでもまだ上手くいかないケースがあります。
それはデフォルト値を受け取るプロパティが存在する場合です。

```typescript
class User {
  id!: number;
  name!: string;
  createdAt: Date = new Date();

  constructor(props: Fields<User>) {
    Object.assign(this, props);
  }
}

// Error: 'createdAt' is missing
new User({id: 1, name: 'alice'});

// OK
new User({id: 1, name: 'alice', createdAt: new Date()});
```

`createdAt`にはデフォルト値があるにも関わらず、
`Fields<User>`を使うと全てのプロパティに値をセットしなければいけなくなります。
これではデフォルト値の意味がありません。
デフォルト値を持つプロパティは初期化を省略できるべきでしょう。

## 試み 4. 一部のプロパティを省略可能にする

メソッドを除外した時と同じように Conditional Types を使ってデフォルト値のあるプロパティを除外できれば良いのですが、
残念ながらデフォルト値の存在を型レベルで判定する方法は (自分の知る限りでは) ありません。

なのでこれに関しては、以下のように手動で省略可能にしたいプロパティ名を列挙するしかなさそうです。

```typescript
class User {
  id!: number;
  name!: string;
  createdAt: Date = new Date();

  constructor(props: SemiPartial<Fields<User>, 'createdAt'>) {
    Object.assign(this, props);
  }
}

// OK
new User({id: 1, name: 'alice'});

// OK
new User({id: 1, name: 'alice', createdAt: new Date()});
```

[Playground]: https://www.typescriptlang.org/play/#code/C4TwDgpgBAYglhANgEwM4B4AqA+KBeKAbygG0AFKOAO1gRQDkBDAWwgxwF0AuKTcjqAF8A3AChQkWkmRNW7XAUKiopCtSgBrCCAD2AM17de-KBAAewCFTRQAFADpHjAE4BzVD0ZUQJDgEp8XC8QKAB+KCoIADcIZygeMlFBEi1dA0wOMXFwaABlCGY4MhdgOEZELAAaKDJUUwsrGzJnHUhnUABpbQUoAHlC4HR4aXlq2twAMhqSsorhlHlsLNEAY0RGVDqAVVRYomVKZABCHioAV2YAI1ixFSoWCBOoVGBnaldbqBXnCEZLZAAgsAeAARP7QAiRADuUDBllsfiyKhWOioL2cZxWwB0zlsYBaYA8UHyhWK7VmQzoaHQO1i2GqAHJvr9-kCGdgAkoVCpepcAFYQLH2DaoOCuKi2YAACzgqGq+NaqERB0ESVEogA9Bq+h1RNCoLTcYQ4MgeABGar3Vg8BnlOArCAMwTKzXa3q6-WG2zG01QC0RB42u0OhnVZngwHAiIQGFwiAI51iIA

`SemiPartial`は指定されたキーのみを省略可能にしてくれる型で、
[ビルトイン][ts-built-ins]の`Partial`などを使って以下のように書けます:
([`Omit`][ts-3-5-omit]は TypeScript 3.5 で追加されました)

[ts-built-ins]: https://fettblog.eu/typescript-built-in-generics/  
[ts-3-5-omit]: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-5.html#the-omit-helper-type

```typescript
type SemiPartial<T, Ps extends PropertyKey> =
  Omit<Fields<T>, Ps> & Partial<Fields<T>>;
```

イメージとしては以下のように intersection を取る事で、指定されたプロパティのみ省略できるようにしています。

```
{
  id: number & (number | undefined),     -> number
  name: string & (string | undefined),   -> string
  createdAt: never & (Date | undefined), -> Date | undefined
}
```

手動でプロパティ名を指定する必要があるのは面倒だし忘れる可能性もありますが、
忘れたとしても実行時エラーが起きるような厄介なバグにつながるわけではないし、このあたりが妥協点ではないかと思います。
また`SemiPartial`の第2型引数を`keyof T`にすれば存在しないプロパティ名を指定できなくなるので、
プロパティ名を間違えたり、古いプロパティ名が誤って残り続けたりする事もありません。

しかし 1 点残念な事に、省略可能なプロパティに`undefined`を指定する事も出来てしまいます。

```typescript
// OK
new User({id: 1, name: 'alice', createdAt: undefined});
```

これをコンパイルエラーにする方法は見つけられませんでした。これを防ぐには結局実行時に引数をチェックするしかなさそうです。

また、逆に大半がデフォルト値を持つようなクラスなら、指定されたプロパティのみを必須にする`SemiRequired`を作る手もあります。
ただこちらの場合は必須のプロパティ名を指定し忘れると初期化されてるはずのプロパティが実はされてないという厄介なバグになるので、
注意が必要です。


## 試み5. もう少し頑張ってみる

一応こんな方法も考えました。これならデフォルト値が設定されているプロパティと、
引数で省略可能なプロパティがずれる心配はありません。

```typescript
class User {
  id!: number;
  name!: string;
  createdAt!: Date;

  static defaults = (): Partial<User> => ({
    createdAt: new Date(),
  })

  constructor(props: Initial<User, typeof User.defaults>) {
    Object.assign(this, User.defaults(), props);
  }
}

type Initial<T, F extends () => {}> = SemiPartial<Fields<T>, keyof ReturnType<F>>;
```

デフォルト値の指定をプロパティ定義時には行わず、デフォルト値を持つオブジェクトを返す関数を定義し、
この関数の戻り値型を使って除外すべきプロパティ名の一覧を取得しています。
しかしデフォルト値の設定に独自の方法を強いる事になるし、デフォルト値の設定がプロパティの宣言からは離れてしまうし、
メリットと釣り合う抽象化かどうかは微妙です。
けれどこの`defaults`を使うと、前述した`undefined`が明示的にセットされるケースにも対応しやすくなります。
プロパティが存在するかどうかに関わらず、値が`undefined`ならデフォルト値を適用すれば良いからです。

```typescript
constructor(props: Initial<User, typeof User.defaults>) {
  const defaults = User.defaults();
  Object.keys(defaults).forEach(key => {
    if (props[key] === undefined) {
      props[key] = defaults[key];
    }
  });
  Object.assign(this, props);
}
```

## 結論

- 省略可能なプロパティ名だけ列挙するくらいで手を打つのが良さそう。
- そもそもプロパティが多すぎるクラスは設計レベルで再考すべきかも。
    - でも RDB のレコードにマッピングされるクラスとかは仕方ないのでは。
- もっと良い方法があれば知りたい。

## 補足

最後に、そもそもクラスを使わないやり方の方が良いケースも多いかもしれません。
単に複数の値をまとめたいだけなら、`interface`や`type`でも良いでしょう。
しかし private なプロパティや constructor による初期化あたりはクラスならではなので、
この辺が欲しい時はやはり使いたくなります。
ただの`interface`や`type`型の値でも、
[Opaque Type][ts-opaque-type] のようなテクニックを使えば初期化のために関数呼び出し (実質的な constructor) を必須にする事はできそうですが、
プロパティが多い場合はその関数の定義に結局似たような不便さを抱える事になりそうです。

[ts-opaque-type]: https://codemix.com/opaque-types-in-javascript/

(あと地味なクラスの利点として、`console.log`などでプリントした時にクラス名が表示されるためわかりやすい、というのもあるかも...)


