+++
date = "2017-12-31T23:53:30+09:00"
description = ""
draft = false
tags = ["javascript"]
title = "JS: ちょっと変わった状態管理ライブラリを作った"
updated = "2017-12-31T23:53:30+09:00"

+++

[Flux][flux] 的なフロントエンドの状態管理について、[Redux][redux] よりもう少しお手軽な設計はないかなと色々考えていた時、ふと浮かんだ案があったので試しに実装してみました。

https://github.com/ryym/thisy

元々のモチベーション:

- Action, ActionCreator 無くても良い気がする
- Redux のような View のテスタビリティは確保したい
- ある状態に関する処理は、read も write もなるべく一箇所に書きたい
  (コードをその種類ではなく扱うドメインごとに集約させたい)

以下のアイデアはハッキリ言って綺麗なものではないし、もはや Flux でもありませんが、アイデアの1つとしては面白いと思ったのでまとめておく事にしました。

[flux]: https://facebook.github.io/flux/docs/overview.html
[redux]: https://github.com/reactjs/redux

## Thisy の概要

Thisyと名付けたこのライブラリでは、状態とその操作を1つまたは複数のクラスで表現します。

```javascript
class CounterState {
  constructor(count = 0) {
    this.count = count
  }

  getCount() {
    return this.count
  }

  $increment(n = 1) {
    this.count += n
  }
}
```

Storeはそれら状態クラスのインスタンスを保持し、外部から何らかの更新依頼を受け取ると、適宜インスタンスのメソッドを呼び出して状態を更新し、購読者に通知します。

ポイントは更新依頼の発行方法で、通常の Flux なら Action を別途定義しますが、Thisy では状態クラスからメソッド情報を事前に抽出しておき、それらを Action として使用します。

```javascript
import { methodsOf, makeStore } from 'thisy'

const Act = methodsOf(CounterState)
console.log(typeof Act.$increment)
//=> 'function'

const store = makeStore(() => [new CounterState()])
store.subscribe(act => console.log('Updated by', act.name))

store.send(Act.$increment, 10)
//=> 'Updated by $increment'
```

更新依頼を出す側はこの Action (メソッド) だけを知っていればいいので、状態クラスのインスタンスや実装の詳細については立ち入る事なく外部からメソッドを呼び出せます。
このように状態更新のメソッドがそのまま Action となるので、Thisy では良くも悪くも Action とそのハンドラが常に 1:1 に紐づく事になります。状態操作とは独立した Action (Creator) がなくなる分、Store (State) と View の結合度が少し上がりますが、実用的には問題ないケースも多いのではと考えています。ただ当然、 Redux のように状態 (の更新) を Reducer という形で分割する事はできなくなります。

更なるポイントとして、Store の`send`には状態更新のメソッドに限らず、任意のメソッドを渡せます。そのため、状態の変更だけでなく取得にも使う事ができます (`send`は常に受け取ったメソッドの実行結果を返します)。

```javascript
store.send(Act.$increment)
store.send(Act.$increment)
const finalCount = store.send(Act.getCount)
```

つまり Thisy では、状態の read/write 処理を`send`1つで行う事ができます。

なお、上記例の`$increment`メソッドが`$`始まりなのは、これが状態を更新するメソッドである事を Store に伝えるためです。「どんなメソッドが状態更新メソッドをするのか」の判別方法はカスタマイズ可能ですが、[ES decorator][es-decorator] が使える環境なら`@updater`という decorator を使う事もできます。

[es-decorator]: https://github.com/tc39/proposal-decorators

```javascript
@updater
increment(n = 1) {
  this.count = n
}
```

つまり Thisy では状態クラスが核となり、その定義から Action を、そのインスタンスから Store を作る形になります。Action や ActionCreator を別途定義する必要がなくなるため、View と Store のコミュニケーションは簡略化されます。その分、単純な状態更新からAPIコールといった各種非同期処理まで、諸々のロジックは全て状態クラスに詰め込まれる形になりますが、内部の構成は使う側が自由に設計できます。

ちなみに React で使う場合は以下のような感じになります。インターフェイスはほぼ Redux を真似しました。

```javascript
import { Provider, connect } from 'thisy-react'
import { Act } from '../store/counter'

// Define a component.
const Counter = ({ send, title, count }) => (
  <div>
    <h1>{title}</h1>
    <p>Count: {count}</P>
    <button onClick={() => send(Act.$increment)}>
      Increment
    </button>
  </div>
)

// Connect the component with your store.
const ConnectedCounter = connect(Conunter, {
  mapProps: send => ({ title }) => ({
    send,
    title,
    count: send(Act.getCount),
  })
})

ReactDOM.render(
  <Provider store={store}>
    <ConnectedCounter title="Counter example" />
  </Provider>,
  document.getElementByID('counter')
)
```

## Thisy の仕組み

Thisy のポイントはやはり、クラスからそのインスタンスに関与する事なくメソッド情報を抽出する点です。しかしこれは、特に難しい作業ではありません。ES2015 以前から JS を使っていた人なら想像がつきやすいと思いますが、Thisy では以下のような JS の特徴を利用しています。

- クラス・インスタンス関係の実体は [prototype chain] である点
- 関数内の`this`がコントローラブルなものである点

[prototype chain]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain

これら JS の仕組みについては説明を割愛しますが、ざっくり内部の仕組みを説明すると以下になります。

- `methodsOf` - 受け取ったクラスの`prototype`に生えているメソッド一覧を取得する (実際にはいくつかのメタデータも付与する) 。
- `send` - 受け取ったメソッドを、Store が保持するインスタンスを`this`に設定して実行 ([`apply`][js-apply]) する。

[js-apply]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply

```javascript
// 雰囲気はこんな感じ

const methodsOf = clazz => copyAndSetMetaData(clazz.prototype)

class Store {
  send(method, ...args) {
    const instance = this.getInstanceFor(method)
    method.apply(instance, args)
  }
}
```

これだけです。Action (メソッド) 内の`this`で参照されるインスタンスは実行時に動的に決まるため、事前に抽出しておいたメソッドを Action として用い、インスタンスには直接触れる事なく外からその処理を dispatch できるというわけです。
このように、Thisy は昔ながらのJSの特徴を使っています。手法として筋が良いかどうかはあまり考えないようにします。

## 状態クラスの作り方

Thisy の状態クラスを作る場合、基本は普通のクラスで良いものの、少し変わった書き方が必要になります。

- 状態を更新するメソッドには目印をつける (前述)
- クラス内で状態を更新する場合も`send`を使う

1つ目は前述の通りで、`$`プレフィックスなどの目印をつけてどのメソッドが状態を更新するのかを示さないと、Store が購読者に通知できません。

2つ目もそれに関連しています。Thisy は単純に`send`による実行しか監視していないため、クラス内で状態更新メソッドを直接呼んでしまうと、Storeが状態更新を検知できません。そのため、クラス内でも状態更新メソッドを呼ぶ場合は`send`を使う必要があります。`send`はStore生成時に状態クラスのインスタンスに渡す事ができます。

```javascript
class CounterState {
  constructor(send) {
    this.send = send
    this.count = 0
  }

  $increment() {
    this.count += 1
  }

  // BAD: これだと状態更新は外部に通知されない
  $incrementAsync() {
    setTimeout(() => this.$increment(), 100)
  }

  // GOOD
  $incrementAsync2() {
    setTimeout(() => this.send(this.$increment), 100)
  }
}

// send をインスタンスに渡す
const store = makeStore(send => [
  new CounterState(send),
])
```

`send`が必須になるのは中々気持ち悪いですが、良い面もあります。それは、状態クラスのテストがしやすくなる事です。内部で他のメソッドを呼んでいるメソッドを単体テストする場合、内部で呼ばれているメソッドをモックしたくなるケースは少なくないと思います。しかしその呼び出しを`send`で抽象化していれば、クラス生成時に渡す`send`をモックするだけで良くなります。

この`send`による抽象化は、状態クラスを複数定義する場合にも有用です。例えば複数の独立した状態クラスがあり、それらを横断した処理が必要になる場合、単純に考えるとそれらクラスのインスタンスを持つ上位クラスを作る感じになると思います。

```javascript
class AppState {
  constructor(entities, users) {
    this.entities = entities
    this.users = users
  }

  loadUsers(params) {
    return this.users.getVisibleIDs().map(this.entities.getUser)
  }
}
```

しかし Thisy の場合、必要なのは`send`と Action だけです。そのため、いくつかの状態クラスにアクセスする親状態クラスのようなものを作る場合でも、その親状態クラスは`send`と各クラスのインターフェイスにのみ依存する事になります。

```javascript
import { Act as Entities } from './entities-state'
import { Act as Paginations } from './paginations-state'

class AppState {
  constructor(send) {
    this.send = send
  }

  loadUsers(params) {
    this.send(Users.getVisibleIDs).map(this.send.to(Entities.getUser))
    // `send.to(getUser)` is same as `id => send(getUser, id)`
  }
}
```

この場合もテストでは`send`だけをモックし、渡された Action (メソッド) によって条件分岐するだけで済むはずなので、見通しの良い単体テストを書きやすいのではと思います。

このように`send`でメソッド呼び出しを抽象化する事で、一般的な DI とは違った形で依存を代替可能なものにできます。ただ Action さえあれば`send`で何でも呼べてしまうというのは諸刃の剣なので、そもそもクラス同士をなるべく疎結合にする必要性は変わりません。

## Thisy の問題点

最後に現時点でわかっている問題点をまとめておきます。

### メソッドは必ず prototype に生やさないといけない

上記のような仕組みであるがゆえに、状態クラスの定義には少し制約があります。事前にメソッド一覧を抽出するにはそれらを prototype に生やさないといけないため、例えばクラスプロパティの記法を使ってメソッドを定義すると上手く動作しません。クラスプロパティでメソッドを定義する場合、そのメソッドは prototype ではなく生成されるインスタンスに直接生える事になり、`methodsOf`では抽出できないからです。

```javascript
class CounterState {
  // BAD: インスタンスに毎回関数を定義する形になり、 prototype からは見えない。
  getCount = () => this.count
}
```

### メソッド一覧の型チェックが完璧じゃない

`methodsOf`により抽出した Actions は、[TypeScript][typescript] や [Flow][flowtype] を使うと基本的にタイプセーフにアクセスできます。しかし残念ながらこれは完璧ではありません。現状だと TypeScript でも Flow でも prototype はインスタンスと同等の型として扱われるため、インスタンスにしか存在しないプロパティやメソッドにアクセスしても型チェックが通ってしまいます。

[typescript]: https://www.typescriptlang.org/
[flowtype]: https://flow.org/

```javascript
class State {
  name: string
  getName: () => this.name
}

const Act = methodsOf(State)

// prototype には getName や name が存在しないので実行時にエラーになるが、型チェックは通ってしまう。。
console.log(Act.getName.length)
console.log(Act.name[0])
```

そのためメソッドは必ず prototype に生やす必要があります。また状態プロパティに関しては、TypeScript なら[private 修飾子][ts-classes]を、 flow なら [munge_underscores][flow-mungeunderscores] オプションを使って`private`化すれば外部からのアクセスを禁止できるため、Action に対するプロパティアクセスをエラーにする事ができます。

[ts-classes]: https://www.typescriptlang.org/docs/handbook/classes.html
[flow-mungeunderscores]: https://flow.org/en/docs/config/options/#mungeunderscores-boolean-a-classtoc-idtoc-munge-underscores-boolean-hreftoc-munge-underscores-booleana

### `send`に渡される関数自体の正当性は型チェックできない 

`send`にはどんな関数でも渡せてしまいます。その際、渡された関数とその引数の型整合性はチェックできるのですが、例えば「`send`に渡されるメソッドが、その`send`を持つ Store が管理している状態クラスのメソッドであるかどうか」、というチェックは静的にできません。

```javascript
class A {
  hello(name: string) {}
}
class B {
  goodbye(n: integer) {}
}

const ActA = methodsOf(A)
const ActB = methodsOf(B)
const store = makeStore(send => [new A()])

store.send(ActA.hello, 'john') // OK
store.send(ActA.hello, 1) // 引数の型が違うので正しくエラーになる。

// Store は B のインスタンスを持っていないが、
// 型チェックは通るため間違いに気づきにくい。
store.send(ActB.goodbye, 1)
```

他の Flux 実装であれば渡される Action が正当なものかどうかを型チェックできるものもあるので、これは結構残念な点です。

## 雑感

Thisy ではシングルトンなクラスに状態管理をまかせる以上、その状態更新はどうしても mutable になりがちです。また、状態の読み書きに関しても本当は関数ベースで定義していく方がきれいだとは思うのですが、JS におけるビルトインの配列やオブジェクトのメソッドは破壊的なものが多いし、immutability や 参照透明性を必要に応じて捨てられる単純さがメリットになるケースもそれなりにある気がしています。
だったらいっそその手の単純さと親和性が高いライブラリがあっても良いかな、という気持ちで作りました。

## まとめ

まとめると Thisy はこんなライブラリです。

- 状態とその操作をトラディショナルなクラスとして表現できる
- 状態クラスのメソッド単体を Action として使う事で、外部からは状態の実体に触れる事なくその読み書きができる
- View での状態操作は`send`の呼び出しに集約されるため、`send`をモックすればテストできる
- ある程度はタイプセーフに書ける
- 状態クラスの設計はユーザが頑張って考える



