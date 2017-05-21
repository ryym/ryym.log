+++
date = "2016-06-30T00:00:00+09:00"
description = "JSに頼らずCSSをコンポーネント化する方法を考えたけど無理だった。"
draft = false
tags = ["css"]
title = "クラス名が長くならないCSS設計を求めて"
updated = "2016-06-30T00:00:00+09:00"

+++

ちょっと前からReactを使うようになり、CSSの良いコンポーネント化方法についてぼんやり考えていたところ、たまたま以下の記事を見つけた。

<http://terkel.jp/archives/2014/12/css-components/>

これはプリプロセッサの存在を前提にしつつ、シンプルにCSSをコンポーネント化する方法についての考察だった。すでに1年以上前の記事だけどちょっと面白かったので、似たような事を実現できないか考えてみた。しかし残念ながらきれいに形にするのは難しそうだったので、せめて考えた事だけでもまとめておく。

## やりたかった事

ここで解決したいCSSの問題は、CSSのルールが全てグローバルスコープで定義されるため、気をつけないとすぐにルール同士の意図しない衝突が起こる事。特にReactなんかを使ってると、CSSもReactのようにコンポーネント単位でまとめたくなる。そのために[CSS Modules][css-modules]というReact前提(JSフレームワーク前提)のツールや、[BEM][bem]や[SMACSS][smacss]などの方法論が存在する。BEMならHTMLはこんな雰囲気になる。

[css-modules]: https://github.com/css-modules/css-modules
[bem]: http://getbem.com/
[smacss]: https://smacss.com/

```html
<div class="panel">
  <div class="panel__header">
    <h1 class="panel__title">The panel title</h1>
  </div>
  <div class="panel__content">
    <div class="card">
      <div class="card__content">
        <h2 class="card__title">The card title</h2>
      </div>
    </div>
    <!-- ... -->
  </div>
</div>
```

この例では`panel`や`card`が独立したコンポーネントであり、その内部のルールは必ずクラス名に`panel`などコンポーネント名のプレフィックスをつける事で、メンテナビリティを確保しつつルールの衝突を避けている。しかし`panel__header`とか冗長だし面倒くさい。もっと単純に以下のように書きたい。

```html
<div class="panel">
  <div class="header">
    <h1 class="title">The panel title</h1>
  </div>
  <div class="content">
    <div class="card">
      <div class="content">
        <h2 class="title">The card title</h2>
      </div>
    </div>
    <!-- ... -->
  </div>
</div>
```

CSS ModulesのようにJSの力を借りて解決するのもありだけど、CSS(とプリプロセッサ)だけでこれを解決する方法があればそれに越した事はない。それに上記のような方法論では嫌われているカスケーディングや詳細度といった、CSS本来の特性もいっそ上手に使えたら面白い。

## 問題

上記の素朴なクラス設計を実際に行った時問題になるのは、`panel`も`card`も共に`.title`という内部クラスを持つ事。だから両者が入れ子(親子関係)になると、親コンポーネントの`.title`クラスへのルールが子コンポーネントの`.title`にも影響してしまう。例えば`panel`のタイトルは太字、`card`のタイトルは下線付きとなる事を意図して以下のようにルールを定義した場合(そして両者ともタイトルがコンポーネントのルート要素の直下にあるとは限らない場合)、`panel`の中にいる`card`のタイトルは下線付きでかつ太字になってしまう。

```scss
.panel .title {
  font-weight: bold;
}

.card .title {
  text-decoration: underline;
}
```

要はこのコンポーネント同士のルールの侵食を防げればいい。

## 方策1

上記のブログで示されていた方法は、「セレクタを工夫して子コンポーネント以下には親コンポーネントのルールが適用されないようにする」というもの。まずコンポーネントとなる要素には必ず`data-component`属性を付与するようにする。

```html
<div class="panel" data-component>
  <div class="title">Panel title</div>
  <div class="card" data-component>
    <div class="content">
      <div class="title">Card title</div>
    </div>
  </div>
</div>
```

そしてコンポーネント内のルールは以下のように書くようにする。

```css
/* panel title rule */
.panel[data-component] > .title,
.panel[data-component] :not([data-component]) .title {
  /* ... */
}
```

このスタイル定義は、「コンポーネント直下の`.title`」と「子コンポーネントの下ではない`.title`」にのみスタイルを当てる事を意図している。これは一見上手くいきそうに思えるものの、実際には上手くいかず、上記HTMLの場合`.card`内の`.title`にも`.panel`の`.title`ルールは適用されてしまう。なぜなら2番目の`.panel[data-component] :not([data-component]) .title`は、`.panel[data-component]`と`.title`の間に1つでも`[data-component]`でない要素があればマッチしてしまうため。この例では`content`クラスを持つ要素がそれに当たる。つまりこのセレクタでは、「子コンポーネント以下の要素には適用しない」という意図したルールは実現できていない。

現状`:not`は単純な否定しか表現できないので、子コンポーネント以下をルールの適用対象から上手く除外しようというアプローチは難しそうに思える。

## 方策2

マッチさせたくないものを除外するのが無理なら、必要に応じてルールを上書きする方法はどうだろう。子コンポーネント以下にはマッチさせないのではなく、親コンポーネントが内部の要素に設定したルールを、子コンポーネント以下の要素に対してはリセットするような方式はどうか。

そこで同じように各コンポーネントに`data-component`属性を付与した上で、以下のようなスタイル定義を考えてみる。

```css
/***** card *****/

/* card title */
.card[data-component] .title,
[data-component] .card[data-component] .title {
  text-decoration: underline;
}

/* reset card title */
.card [data-component] .title {
  text-decoration: none;
}

/***** panel *****/

/* panel title */
.panel[data-component] .title,
[data-component] .panel[data-component] .title {
  font-weight: bold;
}

/* reset panel title */
.panel [data-component] .title {
  font-weight: normal;
}
```

これは長い上に汎用的でもないが、これなら`panel`と`card`がどちらを親とする入れ子になっても、親の`.title`へのルールは子には適用されずに済む。`.title`要素が位置する深さも関係ないし、スタイルシート上で`card`と`panel`のスタイルのどちらが先に定義されても問題ない。

### 仕組み

上のCSSで行っているのは以下の2つ。

1. コンポーネント内の各ルールに、自身が別のコンポーネント内にいる場合のルールを合わせて記述する。
2. コンポーネント内の各ルールに、自身の内側にいる別のコンポーネントに対して、自身のルールをリセットするようなルールを追記する。

#### 1. 自身が別のコンポーネント内にいる場合のルールを合わせて記述する

まず先ほどのCSSにおいて、`card`コンポーネントの`.title`要素に適用されるルールを考えてみる。`card`が他のコンポーネントの内側にいない場合、 その要素には`.card[data-component] .title`のセレクタがマッチし、`text-decoration: underline;`が適用される。もし`card`が別のコンポーネント(`[data-component]`)の内側にいる場合には、それに加えて`[data-component] .card[data-component] .title`というより詳細度の高いセレクタがマッチする。適用されるルールは同じだが、適用ルールのセレクタは通常の`.card[data-component] .title`よりも詳細度が高くなっている点が重要。

#### 2. 自身の内側にいる別のコンポーネントに対して自身のルールをリセットするようなルールを追記する

これは、入れ子になった子コンポーネントに対して親コンポーネントのルールが適用されるのを防ぐために行う。上記の例では、`.panel[data-component] .title`のルール(`font-weight: bold;`)は、すぐ下の`.panel [data-component] .title`のルールによってリセットされる(`font-weight: normal;`)。この2つのセレクタは良く似ているが、リセットルールの方は`panel`と`[data-component]`の間にスペース(子孫セレクタ)があるので、こちらは「`panel`の下にある`[data-component]`の下にある`.title`」を選択する。このリセットルールで重要なのは以下の2点。

* リセット対象のルールと同じ詳細度にする。
* リセット対象のルールよりも後に記述する。

詳細度が同じで定義順序が後なので、`.title`のルールとそのリセットルールの両方がマッチする要素があれば、その要素にはリセットルールが優先して適用される。上記例のHTMLの場合、`card`下の`.title`には`panel`の`.title`のルールも適用されてしまうが、同じくそのリセットルールも適用されるので、`font-weight`は`normal`(初期値)となる。このようにして、入れ子になった子コンポーネントに対しては自身のルールをリセットするようにする。

#### リセットした上にルールを適用する

以上の事から、`panel`コンポーネントの中に`card`コンポーネントがいた場合、`card`内の`.title`には以下4つのルールが以下の優先順位で適用される事がわかる。

```html
<div class="panel" data-component>
  <div class="title">Panel title</div>
  <div class="card" data-component>
    <div class="content">
      <div class="title">!! Card title !!</div>
    </div>
  </div>
</div>
```

```scss
// 1
[data-component] .card[data-component] .title {
  text-decoration: underline;
}

// 2
.panel [data-component] .title {
  font-weight: normal;
}

// 3
.panel[data-component] .title {
  font-weight: bold;
}

// 4
.card[data-component] .title {
  text-decoration: underline;
}
```

本来なら4番目の`card`内の`.title`へのルールが適用されてくれれば簡単だが、スタイルシートでは`panel`コンポーネントのルールの方が後に記述されているため、詳細度が同じ3番目のルールの方が優先されてしまう。しかし、各コンポーネントは自身のルールをリセットするルールを定義している。`card`内`.title`は`panel`内`.title`をリセットするルール(2番目のルール)のセレクタにマッチするので、`panel`内`.title`のルールはここではリセットされる。そして更に各コンポーネントは自身が別コンポーネント内にいる場合のルールも定義しており、それは通常のルール(とそのリセットルール)よりも詳細度が高くなっているため、それが最も優先して適用される(1番目のルール)。これで仮に`panel`内`.title`が`text-decoration`を変更しており、そのリセットルールに`text-decoration: none;`があっても、詳細度の高い1番目のルールが優先されるため問題ない。

このように、詳細度と定義順序による優先順位を利用する事で、入れ子になった子コンポーネントに親コンポーネントのルールが影響する事を防ぐ事が(一応)できた。ただし上記のやり方では、コンポーネントのネストが3段階以上になる場合に対応できない。とはいえ、いずれにせよ上記のようなルールを手で書くなどどう考えてもやってられないので、例えば以下のような記述からプリプロセッサに自動生成させる事ができればいい。となれば、その際に最大いくつのネストまで考慮するかを指定できるようにし、指定したネスト数の分だけネスト用セレクタとリセットルールを生成するようにすればまあ何とかなる。それとプロパティごとにどんな値にリセットするかを指定できる必要もありそう。

```scss
@component .panel {
  .title {
    font-weight: bold;
  }
}

@component .card {
  .title {
    text-decoration: underline;
  }
}
```

### デモ

実際にこの方法を使って、内部に`.title`を持つ数種類のコンポーネントを色々なパターンで入れ子にしてみた。

http://ryym.github.io/wip-css-layers

赤文字のタイトル、下線を持つタイトルなどいくつかの種類があるが、コンポーネント同士がどう入れ子になっても、親コンポーネントの`.title`のルールは子コンポーネントに影響を及ぼしていない事がわかる。
かなり無理やり感のある方法だが、とりあえず当初考えていた単純なようなスタイル指定はできるようになった、ように見える。がしかし、もう少し考えるとこの方法では対応できないケースが普通にある事に気づいた。

### 上手くいかないケース

上記方法の問題点は、親子コンポーネントそれぞれのルールの内、(詳細度が)異なるセレクタのルールが同じ要素に適用される場合を考慮していない点にある。例えば以下のようなケース。

```css
/* rule A */
.foo[data-component] .head .title .strong {
  font-weight: bold;
}

/* reset rule A */
.foo [data-component] .head .title .strong {
  font-weight: normal;
}

/* another component's rule B */
[data-component] .bar[data-component] .strong {
  font-size: 2rem;
  font-weight: bold;
}
```

```html
<div class="foo" data-component>
  <div class="head">
    <div class="title">
      <span class="strong">This</span> is a title!
    </div>

    <div class="bar" data-component>
      <div class="head">
        <div class="title">
          <p class="strong">bar's paragraph.</p>
        </div>
      </div>
    </div>

  </div>
</div>
```

ルールA(及びそのリセットルール)は`bar`内の`.strong`にもマッチする。同様にルールBも同じ要素にマッチするものの、ルールAよりもセレクタの詳細度が低いため、この場合はルールAのリセットルールの方が優先されてしまい、`font-weight`が`bold`にならないという事態が起こる。

方策2はつまるところ、コンポーネントが別のコンポーネントの内側にいる場合は意図的に詳細度を高めたセレクタのルールを適用し、親となったコンポーネントのルールよりそれが優先されるようにしよう、というアイデアだった。しかしCSSのセレクタとHTMLの構造は1:1に紐付くわけではないため、親と子で全然違うセレクタのルールが同じ要素にマッチする事も当然ある。そういう場合、子のセレクタの詳細度が常に親より高くなる事を保証するような方法は(たぶん)存在しない。

つまりは以下のように、子孫セレクタと2つ以上の要素指定を含むセレクタのルールは全て破綻しうるという事になる。残念。

```scss
@component .foo {
  // OK
  .title {}
  > .head > .title {}

  // BAD
  > .head .title {}
  .head > .title {}
  .head .title {}
}
```

ちなみに、子セレクタ(`>`)のみによる内部要素の指定であれば問題ない。何段階下の要素へのルールだろうと、全てのセレクタに`:not([data-component])`を追加すればいいだけので、リセットルールさえ必要ない。

```scss
/* input */
@component .foo {
  > div > div > label {
    // ...
  }
}

/* output */
.foo[data-component] > div:not([data-component]) > div:not([data-component]) > label:not([data-component]) {
  // ...
}
```

## ひとまず結論: コンポーネントCSSはできなくもないが制約あり

というわけで、「子孫セレクタを使って2段階以上下の要素に対するルールは定義できない」という制約の上でなら、素朴なクラス名によるルール定義という当初やりたかった事は実現できそうに思える。こうなると、内部で更にネストしたい場合は子セレクタしか使えない。

```scss
/*
 * 他のコンポーネントが`.title`など同じ内部ルールを持っていても、
 * それらと衝突せずに済むCSSコンポーネント
 */
@component .panel {
  padding: 5px 12px;
  // ...

  .title {
    // ...

    &:hover {
      // ...
    }
  }

  > .content {
    // ...

    > .row {
      // ...
    }
  }
}
```

しかし、これはちょっと実用性に乏しい。それに他にもまだ問題はある。

* プリプロセッサで生成されるCSSの記述量が大きくなる
* 各要素に何段階ものルールを重ねて適用していく方法はパフォーマンスに影響しないか
* 無駄に複雑

JSの力を使わないなら、やはりおとなしく多少冗長でも安全でシンプルなCSSを書いていくのが良さそう。
