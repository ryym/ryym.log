+++
date = "2019-08-05T23:35:02+09:00"
description = ""
draft = false
tags = ["ruby", "rails"]
title = "Rails 開発で意識していること 1"
updated = "2019-10-04T22:49:00+09:00"
+++

Rails での開発を始めて結構経つので、
ふと思いたち普段の開発で考えている諸々を文章化してみる事にしました。
あくまで自分の開発経験に基づくものであり、以下を前提とします:

- 中規模なアプリ (テーブル数 100以下)
- 小規模なチーム (開発者が5名以下)
- なぜ Rails なのかは考えない (そこは動かせないものとする)

また若干長いので分割しました:

- Rails 開発で意識していること 1 (この記事)
- [Rails 開発で意識していること 2][my-rails-practice-2]

[my-rails-practice-2]: /posts/my-rails-practice2/

## はじめに

[clean-architecture-book]: https://www.amazon.co.jp/dp/B07FSBHS2V

まず初めに、これから書く事のおおもとの前提を確認しておこうと思います。
例えば適切な命名なり責務の分割なり、 Rails に限らずコードを書く際に考えるべき事は山程あるわけですが、そもそもなぜそういった種々のプラクティスが必要とされ、たかがコードの記述に試行錯誤するのかといえば、それはきっと変化に強いコードを書くためだと要約できるように思います。  
これに関しては [Clean Architecture][clean-architecture-book] の第一部がとても明確に示してくれています。
プロジェクトのフェーズや性質にもよるでしょうが、一般にソフトウェアは要件や状況に応じて柔軟に変わり続ける (べき) ものであり、それが可能である事こそソフトウェアの「ソフト」ウェアたる意義であるはずです。
特に現実のビジネスや日々の生活に関わるプログラムにおいては、それらの変化に合わせてプログラム自体も絶えず変わっていく必要があり、そこに明確な完成像は存在しません。
つまり乱暴に言い切ってしまえば、ソフトウェアにとっては修正や拡張のしやすさこそ正義だ、という事になります。  

ではどんなコードのソフトウェアなら修正しやすいかといえば、

-  読みやすい。
    - 既存の挙動を把握しやすい。
    - 修正すべき範囲を特定しやすい。
- 動作検証すなわちテストを自動化しやすい。
    - 何をテストすべきか、どこをテストすべきかが明確である。
    - テストに必要なセットアップが少ない。

といった性質が基盤になってくるのではないでしょうか。
コードに手を加えるにはまず読む必要があるし、自動テストがあれば頻繁に修正しても繰り返し動作検証できます。
読みやすさというのは曖昧ですが、これは別に「自然言語のように読める」といった話ではなく、むしろミクロ的にはいかに意図の曖昧さが排除され誤読しづらくなっているか、マクロ的にはコードが構造化され全体の流れが認識しやすくなっているか、といった点に関わってくると思います。
でこれら基盤のための大雑把なプラクティスとして、

- 可変な状態は極力減らして局所化する。
- 大きくて複雑な責務は小さくて比較的単純な責務 (モジュール) に分割する。
- 個々のモジュールはなるべく独立させてテストしやすくする。

あたりが大事になってくるのだと思っています。

またこれだけではなく、使用する言語やフレームワークの慣習・規約に則る事も大切です。
無意味にオリジナリティを加えるのは読み手に余計な負荷を与えるだけだからです。
しかし時にはその手の慣習や規約が、上記のような方針と相容れない事もあります。
そういう場合、私は基本的に前述した方針を優先します。

では以上のような方針を Rails 開発においてはどう適用しているか、ざっくばらんに書いていきます。

## DSLとロジックを分離する

まずはある種の心構え的な話になりますが、 DSL とロジックは分離する意識を持つ方が良いと思っています。
これは一言で言えば、フレームワークが担う責務と、フレームワークを使う側が担う責務を分けて考えよう、という話です。
この分離をコードにも反映できれば、フレームワークを使う側である自分たちがメンテ・テストしていくべき範囲が明確になるはずです。

Rails ではコントローラによるリクエストのハンドリングや、モデルのバリデーション・関連定義といった様々な機能を簡潔に書ける仕組みが用意されています。そしてこの手の DSL は必要な機能を宣言的に記述できるという意味で、極端に言えば YAML などの設定ファイルを書くのに近い側面があります。

```ruby
# ユーザモデルの定義。
class User < ApplicationRecord
  # User は 0 個以上の Post を持つ。
  has_many :posts, dependent: :destroy

  # name は必須項目であり、最大文字数は30。
  # (実際のバリデーション処理を書く必要はない)
  validates :name, presence: true, length: { maximum: 30 }
end

# ほとんどこんな感じ
# User:
#   associations:
#     posts:
#       type: has_many
#       dependent: destroy
#   validations:
#     name:
#       presence: true
#       length:
#         maximum: 30
```

そしてこういった DSL が正しく機能する事に関しては Rails が責任を持ちます。
そういう意味では、例えば「`name`が空な`User`はバリデーションに失敗するか」だけを確認するようなテストは不要だと思っています (後述)。
DSL を期待通りに動かすのはフレームワークの仕事であり、自分達は DSL を正しく記述しさえすれば良いからです。

一方で、Rails の DSL を書くだけで完成するアプリケーションは少ないでしょう。
普通はそのアプリケーションに固有のビジネスロジックが存在するはずです。
そしてこちらに責任を持つのは当然、フレームワークではなくアプリを開発する私達自身となります。
となればそれらのロジックはメンテナブルであるべきだし、ユースケースに沿ったテストも必要です。
これが DSL とビジネスロジックを分離したい理由で、
モデルでもコントローラでも、ロジックが DSL 内に散りばめられていると、それを単体でテストするのが面倒になりがちです。
特に副作用満載のコントローラがロジックを持ってしまうと非常にテストしにくいです。
Rails のコントローラは副作用を差し替えてテスト出来るようにはなっていないので、単体テストには不向きです。
Fat Controller がアンチパターンと呼ばれる所以の 1 つですね。  

よってビジネスロジックはなるべくフレームワークとは独立して実装し、 DSL 内ではそれを使うだけ、という形にするのが良さそうです。

## ミックスインは極力使わない

次に個人的に徹底したいのはこれです。
Ruby に限らず、私は動的型付け言語でのミックスイン機構があまり好きではありません。
以下が主な理由で、コードの共有だけを目的としたクラスの継承と同じようなデメリットが多いと感じます。

- そのミックスインによってどんな依存が増えるのかが明瞭でない。
- ミックスインとミックスインを使う側が密結合 (特に相互依存) になりやすい。
- ミックスイン部分をモックに差し替えてテストしづらい。

Rails にはミックスインをより簡単に書くための Concern という機構もありますが、同じ理由であまり積極的には使いません。

ミックスインというのは基本的に、ライブラリやフレームワークが DSL を提供するための機能だと思っています。
そのためミックスインはそのような用途に限定して使い、ビジネスロジックのモジュール化には別の方法を使いたいです。  
以下で上述したデメリットについてもう少し詳しく書きます。
(以後ミックスインを使う側を便宜的に「インクルーダ」と呼びます)

### どんな依存が増えるのかが明瞭でない

これは特に複数のモジュールを include すると顕著で、動的型付け言語だとどのメソッドがどのミックスインに属しているのかを形式的に判断する方法がなく、せいぜいモジュール名とメソッド名から類推するしかありません。
これはコードを追う上でとても不便だと感じます。

```ruby
class Invoice < ApplicationRecord
  include NumberUtils
  include InvoiceUtils

  def price_with_tax
    # with_tax はどこからきた!?
    with_tax(price, created_at)
  end

  # ...
end
```

こういう例を見ると、冒頭に述べたコードの「読みやすさ」という性質は、必ずしもコードが短いとか、自然な文章っぽく読めるという事ではないと気づきます。
それよりもモジュール同士の境界が明確で、どこに何が書いてあるかを把握しやすい事の方がずっと大事です。
しかしミックスインを使うとそれが難しくなりがちです。


### ミックスインとインクルーダが密結合になりやすい

依存は厳密に一方向であって欲しいのに、仕組み上ミックスインがインクルーダに依存する事も容易に出来てしまいます。
これはミックスインとインクルーダの相互依存を招きやすいし、意図せずネームスペースの衝突を起こすケースもありえます。
例えば以下の例です。これは Rails の Concern を使い、ActiveRecord を継承するクラスに include される事を意図したミックスインです。

```ruby
module AdminNotifiable
  extend ActiveSupport::Concern

  included do
    after_create :notify
  end

  def notify
    details = { id: id, name: name }
    # Send an email to administrators...
  end
end
```

`after_create`というクラスメソッドや`id`、`name`という属性の存在を暗に前提としています。
モデルに include される想定とはいえ、`name`という特に一般的ではないカラムにまで依存しています。
問題なのは、ミックスインだとこの手の依存がコード内に暗黙的に散らばりやすい事です。
にもかかわらず、このミックスインを使う側はそれらの依存を知っている必要があるし、カラム名を変える際はこのミックスインにも影響がある事を覚えておかなければいけません。
更にこの例では`notify`という抽象的なメソッド名を使ってしまっているので、インクルーダが同名のメソッドを持っていると、知らないうちに`after_create`の処理を上書きしてしまう事になります。
どれも実に不便です。

そんなの設計が悪いといえばそれまでですが、ミックスインとインクルーダが互いにそれぞれの中身にアクセス出来てしまう構造はどうしてもこの手の設計を引き寄せやすいと思います。
Ruby ではそもそもミックスイン内の`self`はインクルーダになるわけで、
そう考えると include した時点でほとんど相互依存しているようなものだという気もします。

### テストがしづらい

更にはテスタビリティの問題もあります。
ミックスインという形でロジックを切り出すと、ミックスイン側のテストもインクルーダ側のテストもしづらくなると感じます。
まずミックスイン側はクラスでいうコンストラクタのような初期化のタイミングがないので、依存したいモジュールがあってもそれを差し替え可能な形で宣言しにくいです。
一方のインクルーダ側をテストするにしても、ミックスインをモック化するのが面倒です。
ミックスインの依存については別途依存をセットするメソッドを用意する、
インクルーダのテストではミックスインのメソッドを動的に上書きしてモック化する、など対応方法はありますが、そこまでしてミックスインを使う意義をあまり感じません。

よって、 Ruby でコードをモジュール化するなら基本はプレーンなクラスを使うのが簡単で良いと思っています。

## プレーンなクラスを基本単位とする

クラスは継承に頼ると途端にわかりづらくなるものの、継承なしで運用する分にはミックスインのような危なっかしさが少なく、使いやすいと感じます。

### 依存を分離する

何らかのビジネスロジックあるいはユースケースを実装する際、他のモジュールや外部のライブラリを必要とする場面は多くあります。
この時、これらを直接コード内で使ってしまうと、そのユースケースのテストが難しくなる事があります。
例えば外部の API を使うユースケースの場合、テストの度に実際の API コールが走ってしまうのは困るでしょう。
仮に他のモジュールに依存しているだけの場合でも、そのモジュールがどんな依存を持っているかはわかりません。
そのモジュールが依存するモジュールが依存するモジュールが大きな副作用を持っているかもしれません。
こういう場合、頑張って全ての依存を含めてテストしようとしているとキリがないので、普通はインターフェイスにのみ依存する事に務めるわけです。
そうすれば、テスト時にはそのインターフェイスを満たすモックに依存を差し替える事で、そのモジュールが期待通りに動いてくれるか、副作用の心配なくテスト出来るからです。    
言語の機能でインターフェイスを定義できるなら DIP (依存性逆転の法則) のように「全ては抽象に依存すべし」となるでしょうが、動的型付けの Ruby では無理なので、依存するオブジェクトの型がそのまま依存するインターフェイスとなり、テストでは同じシグネチャのメソッドを持つオブジェクトをモックとして使う感じになるでしょう (ダックタイピング)。

というわけで、モジュールには実行時に依存を差し替えられるような仕組みが必要です。
そしてそれを Ruby でやるなら、クラスのコンストラクタを使うのが簡単かつ自然ではないでしょうか。
クラスであれば、

- コンストラクタで依存を明示的に受け取る。
- モジュールを使う側は、そのクラスが必要とする依存を知らずに済むようにする。

という仕組みを作りやすいです。コードにするとこんな感じのイメージになります:

```ruby
class BookmarkImporter
  def self.create
    new(
      api: BookmarkAPI.create,
      logger: DefaultLogger.create,
    )
  end

  def initialize(api:, logger:)
    @api = api
    @logger = logger
  end

  def import(access_token)
    # Use @api and @logger
    # ...
  end
end

# モジュールを使う側
importer = BookmarkImporter.create
importer.import(access_token)
```

これなら、モジュールを使う側は`create`を呼ぶだけでインスタンスを得られるし、テスト時には直接コンストラクタを呼ぶ事でモックに差し替えて実行できます。

```ruby
it "imports bookmarks using API" do
  importer = BookmarkImporter.new(
    api: mock_api,
    logger: noop_logger,
  )

  result = importer.import(dummy_token)
  expect(result.ok?).to be(true)
  # ...
end
```

依存や状態を持たないモジュールであれば、いちいちクラスのインスタンスを作るのは無駄に感じるかもしれませんが、
この形式に統一しておけば将来依存が増えてもインターフェイスは変えずに済みます。
まただいたいのモジュールはステートレスになるでしょうから、インスタンスをキャッシュしてシングルトンで運用しても良いかもしれません。
直接`new`を使わず専用のメソッドを使うルールにしておくと、この辺の切り替えもインターフェイスを変えずにやりやすいです。

### Rails ではどうなる？

とはいえ、素の Rails にはプレーンなクラスなんてほとんど登場しません。
Rails の規約に沿って作るクラスはコントローラなりモデルなり、フレームワークのクラスを継承して始めから何かしらの責務を担っています。
しかし前述のようにビジネスロジックをフレームワークから分離できれば、そこではこの方針を適用する事ができるはずです。

---

続き - [Rails 開発で意識していること 2][my-rails-practice-2]
