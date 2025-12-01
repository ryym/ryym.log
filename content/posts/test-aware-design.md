+++
draft = true
date = "2025-11-30T08:57:00+09:00"
tags = []
title = "Test-aware Design?"
updated = "2025-11-30T08:57:00+09:00"
+++

## 免責事項

- これはアイディアのメモであり、実用性を試してはいない。
- サンプルコードは Ruby だが、特に Ruby に閉じた話ではない。
- テスト時に使う代替オブジェクトには mock, stub など色々あるが、ここでは簡潔さのためざっくり「実際の処理内容をテスト用に差し替えたもの」一般をまとめてモックと呼ぶ。

## 課題: 間接的な依存をモックする手間

Webサービスなどの開発において、テスト時にモックをどのように・どのくらい使うべきかには流派があるが、モックの使用を完全に避けることは難しい。
典型的には外部サービスの API コールやメール送信など、自動テストを実行するたびに走ると困る処理は少なくない。
そのような処理はテスト時にはモックに置き換えた上で、アプリケーションの他の部分が期待通りに動くかを検証することになる。

これはそのような処理に直接依存するコードをテストする際には、大して難しくない。依存を差し替えれば済む。

```ruby
# 雑な例: 何らかの注文の支払いを実行するクラス
class OrderPaymentService
  def initialize(payment_api: PaymentApi.new, notification_service: NotificationService.new)
    # ...
  end

  # payment_api による支払いの実行、 notification_service による確認メールの送信を含む。
  def process_payment(order:)
    # ...
  end
end
```

```ruby
test "mark Order status declined if payment is declined" do
  service = OrderPaymentService.new(
    payment_api: PaymentApiMock.new(mode: :charge_declined),
    notification_service: NotificationServiceMock.new,
  )
  # ...
end
```

面倒なのは、そのような差し替えたい依存がテスト対象の奥深くに存在する場合だ。例:

- 内部的に `OrderPaymentService` を使う別のクラスのテスト
- 内部的に `OrderPaymentService` を使う Web API のテスト

外部サービスへの API コールやメール送信は `OrderPaymentService` に依存するあらゆる処理で発生することになる。
そのため `OrderPaymentService` に直接的・間接的に依存するあらゆる処理のテストには、以下が求められる。

- `OrderPaymentService` が外部に依存する処理を行うと、まず知っていること。
- 外部に依存する処理を、必要に応じて適切にモックに差し替えられること。

特に2点目は `OrderPaymentService` の内部実装に関する知識を暗に要求し、コンポーネント間の依存関係を必要以上に強めてしまう。
自分が知っている範囲では、以下のような対処法がある。

1. どんなクラスのテストであれ、直接の依存関係を常にモックに差し替えてテストする。
2. 何らかの方法で、差し替えたい依存を外から指定する。
3. 外部依存へのクライアントをモックにするのではなく、外部サービス自体のモックを使う。

方法1はいわゆるロンドン学派に相当する考え方で、上記例でいえば `OrderPaymentService` 自体をモックに差し替えてテストする形になる。これなら、実際の処理では内部的に API コールやメール送信が走ることを気にせずに済む。

この方針でテストコードを書く方針に開発チームが合意するなら、それでこの課題は解消する。
一方でなんでもかんでもモックにすると、コンポーネント間の結合時に発生する問題を見落としやすくなったり、各テストが対象の内部実装の詳細に依存しやすくなったり、デメリットもある。もしモックにせざるを得ない部分以外は実際の処理を使ってテストできるなら、その方がテストの信頼性は高いと言える。

方法2は言語やフレームワークによって色々やり方があるだろうが、一番単純なのは差し替えが必要な依存を引数でリレーしていくことだろう。

```ruby
# OrderPaymentService に依存する別のクラス
class SubscriptionBillingService
  def initialize(payment_api: PaymentApi.new, notification_service: NotificationService.new)
    @order_payment_service = OrderPaymentService.new(payment_api:, notification_service:)
  end
end

test "process monthly recurring billing" do
  service = SubscriptionBillingService.new(
    payment_api: PaymentApiMock.new(mode: :success),
    notification_service: NotificationServiceMock.new,
  )
  # ...
end
```

強力なテストライブラリや DI コンテナなど、直接引数を渡さずに差し替えを実現する手もありえる。

```ruby
# RSpec
allow(PaymentApi).to receive(:new).and_return(payment_api_mock)

# Dependency injection (imaginary)
DI.register(PaymentApi, payment_api_mock)
```

しかしどんな方法であれ、 `OrderPaymentService` が他のどんなクラスに依存しているかは知らないといけない。
特に各種失敗やエラーケースをテストしたい場合、 `PaymentApi` のどのメソッドをどのように失敗させる必要があるか、といった知識も必要になる。

方法3は実際の外部サービスと同等に振る舞うダミーのサービスを起動し、 `PaymentApi` 等のクライアントをダミーのサービスにアクセスさせてテストする方法だ。これならアプリケーション内のテストではモックを一切使う必要がなく便利だが、やはりエラーやエッジケースといったバリエーションをテストしたい場合、何らかの方法でダミーのサービスがそういうレスポンスを返すよう設定しなければならない。
つまり `OrderPaymentService` と外部サービスがどのように通信するかの知識は漏れることになる。
また特定の条件でのみ API コールを失敗させる、といった柔軟な対応もしづらいことが多い。
方法3のバリエーションとして、ダミーのサービスを起動するのではなく API リクエストを何らかの方法でキャッチしてダミーのレスポンスを返す、といった仕組みを使う手もあるが、やはり API の詳細を知らなければならない。

sample code

- 常に直接の依存関係をモックする。
- 外からモックをリレーする。 (リレーじゃなくても、間接的な依存関係を指定する)
- 外部のダミーサービスを使う。

1の方針で統一するなら、話はそれで終わり。コンポーネント間の統合は別途テスト
1以外の方法では、どうしても内部の実装詳細に依存する。

---

アイディアのメモであり、実際に試してはいない。

- 多くのサービス開発では、テスト時にモックが必要。
- モックをどの程度使うべきかには流派があるが、使用を完全に0にすることは難しい。
- モックを使うテストはしばしば煩わしい。特に間接的な依存をモックしたい場合、テスト対象のコードの内部詳細に大きく依存する。
- 内部実装に強く依存せず、テスト時に実行できない副作用を迂回するには (例: 外部APIの使用)
  - 直接の依存関係を常にモックする (ロンドン学派)。
    - 確かにモックだらけにはなるが、実際どのくらいデメリットがあるんだろう。単体テストとその考え方に何かあったような。
  - クライアントをモックするのではなく、ダミーのサーバーを作ってそこにリクエストする。
    - 成功 (デフォルト) 以外のパターンを試したい場合、結局何らかの方法で結果を指示する方法が必要。
  - 差し替えが必要になりえるクラス (api client etc) は内部に隠さず、引数でリレーする。
    - 結局そのクライアントのどのメソッドがどういう風に失敗してほしいか、を外から指定しないといけないのは内部実装依存。

本来アプリケーションがテストを意識した作りになるのは避けるべきだが、いうて DI とかインターフェイスとかテストのために必要になることは多いし、実際有用。ならもう一歩踏み込んで、テストのシナリオまでここのサービスが管理できる方が、より凝集的でいいのでは。

内部実装に依存せずいい感じにやる第三の案: アプリケーションコード側が、テストのユースケースにも対応する。

簡潔にやりたければロンドン学派方式でいいと思うが、一応下記のメリットはある。

- モックを最小限にできる。
- 単体テスト以外でも、例えばリクエストヘッダー等で simulate を指定してモックを使える。
  - サービス間の統合テストにはならないが、アプリ内での統合はテストできるし、エッジケース等を自由に再現できたら便利では。
  - それに依存先のサービスもコントロール下にあるなら、そちらでも同様の test-aware な作りにして引数で挙動を変えられるようにすれば、サービス間の統合もテストしつつ各種パターンをシミュレートできる。

ただし実装は面倒になる。

---

- アイディアのメモのみで試してない
- 外部APIの呼び出し、滅多に起きない失敗のケースなど、モックが必要な場面は0にはならない
- テスト時のモックの流派はあるが、どちらもメリデメあり
  - 全部をモックするのはセットアップが面倒になりがちだし、結合時の問題を見落としやすくなる
  - なるべくモックしない方が信頼性も高まるが、間接的な依存をモックせざるを得ない時に不便
- いっそアプリケーション本体にテストのための機能を追加してしまうのはどうか
  - `simulate: <pattern( | patterns)?>`
  - 各サービス (?) は、本来の処理に加え、その中で置きうるパターンのシュミレーションを合わせて提供する責務を持つ
  - 外からはシュミレートしたいパターンを指定するだけ。内部の構造を知る必要がない
  - サービスは自身の依存関係に対して、指定されたシミュレーションを実行するために必要なパターンを再帰的に指定していく
  - ほとんどのサービスはただシミュレーションのパターンをリレー・変換するだけ
  - 末端の、実際にテストでは実行できない処理を行うクラスのみ、処理内容を simulate に応じて置き換える
- この方針のいいところ
  - テストフレームワーク非依存
  - 単体テスト以外でも使える。
    - Webアプリなら URL やリクエストヘッダーでパターンを指定可能にすれば、E2Eテスト等でもシミュレートできる
    - 依存するマイクロサービスがあれば、実際に通信はしつつパターン指定によりわざと失敗させる、とかもできそう
- この方針のしんどそうなところ
  - 試したいパターンが多い処理では、指定方法が複雑になりうる
  - そもそもテストのためのコードを混ぜ込むことをどの程度許容できるか
- 値の取得ではなく更新、すなわち実際の結果がアプリケーションの外部を確認しないとわからないような副作用は、
  別途それ用の仕組みを作れると良さそう。シミュレーション時には特定のテーブルに結果を書き込むとか。

---

主に Rails におけるテスタビリティの確保について、ふと思いついた案。試してない

テストにおいてモックをどの程度使うかには流派があると思う (積極的に使ってテストの責務を分離 or 最小限にして実動作で担保)

モックを最小限にして、必要な箇所以外では実際のコードを使う
(テストの重複を多少は許容する)
(でもこの辺の話なくても別に良いか)

モックの必要性が 0 になることはない
間接的な依存関係にあるやつをモックに差し替えるのがしんどい
単に常に特定のパターンになれば良いならデフォルトでモックすれば良いが、失敗ケースをテストしたいとか、別の挙動を指定したい時は特に
無理にアプリケーションコードをテストと分離せず、積極的にテストを意識してしまって良いかも

`A->B->C` という3段階の依存がある自然な例がほしい。 C が外部との通信 (とか面倒な副作用) を持つ。
でも全部サービスじゃなくても、 A はコントローラでも良いのか。

```ruby
# app/features/billing/build_summary.rb
class Billing::BuildSummary
  def initialize(payment_api: PaymentApi.new, collect_service_usage: Stat::CollectServiceUsage.new)
    @payment_api = payment_api
    @collect_service_usage = collect_service_usage
  end

  if Env.test?
    def self.new(mode: :default)
      case mode
      in :default
        super()
      in :payment_history_error
        super(
          payment_api: PaymentApi.new(failures: {
            get_payment_history: PaymentApi::NetworkError.new('fake error'),
          })
        )
      in :unexpected_error
        super(collect_service_usage: Mock::Feature.will_raise(NoMethodError))
      end
    end
  end

  def run(account:)
    payment_method = @payment_api.get_payment_method(account:)

    payment_history = begin
      if payment_method
        @payment_api.get_payment_history(account:, max: 5, order: :newest)
      else
        PaymentHistory.unavailable(account)
      end
    rescue PaymentApi::Error => err
      Rails.logger.warn("failed to get payment history: ", err)
      PaymentHistory.unavailable(account, err:)
    end

    usage = @collect_service_usage.run(account)
    BillingSummary.new(account:, payment_method:, payment_history:, usage:)
  end
end

# app/lib/payment_api.rb
class PaymentApi
  def initialize
    @client = SomeExternalPaymentApiClient.new
  end

  if Env.test?
    def self.new(failures: {})
      Mock::PaymentApi.new(failures:)
    end
  end

  # ...

  # def get_payment_history(account:, last:)
  #   @client.history(customer_id: account.public_id, history_count: last, order: :latest)
  # end
end

class Api::BillingController < ApplicationController
  def setup(build_billing_summary: Billing::BuildSummary.new)
    @build_billing_summary = build_billing_summary
  end

  if Env.test?
    def setup
      case test_context
      in 'default'
        super()
      in 'payment_history_error'
        super(build_billing_summary: Billing::BuildSummary.new(mode: :payment_history_error))
      end
    end
  end

  def summary
    summary = @build_billing_summary.run(account: signin_account)
    render json: serialize(summary)
  end
end

class ApplicationController
  before_action :setup
  def setup; end

  if Env.test?
    def test_context
      request.headers['App-Test-Context'] || 'default'
    end
  end
end
```

深い階層にいるやつの挙動をあとからカスタマイズしたくなったら、そこにいたるまでのツリーを全部いじらないといけないのが大変。
逆にもうカスタマイズが不要になって実は使われてないのに残り続けるとかもありえる。
なるべくこういうことしないで済むならそうしたいが、どうすれば。結局依存関係を飛び越えてテスト中に邪魔なやつを全部モックする方が簡単？　うーん。

ここで考えてるような工夫は、そもそもテスト時に依存は全部モックにして対象のクラスを完全に単体でテストする方針なら問題にはならないんだよな。
直接依存するクラスだけを気にすれば済むから。
その代わり各クラスが実際に協調して正しく動くことの担保は別途欲しくなるし、 Ruby だとモックと実際のクラスにずれがあっても見逃したりしうる。
また E2E テストの時にも困る。

一方で上記のようなことをするなら、テストモードのテストも欲しくなる。

もしくはアプリケーションコード内でのカスタマイズはなし？

- テスト時にはデフォルトでモックされるが、成功ケースのみ提供
- 失敗やイレギュラーケースをテストしたい時には、間接的な依存先についても知った上でそれを外からカスタマイズする

もしくはテスト時に引き起こせるパターンをグローバルに一覧化して、それを指定する？　そうするとクラス階層感で内部を隠蔽しつつ情報を渡してくような手間がなくなる。
でもこれで全部網羅するのは現実的じゃない気がする...

```ruby
class PaymentApi
  if Env.test?
    def self.new
      if Env.test_context.include?(:fail_to_payment_history_by_network_error)
        # ...
      else
        # ...
      end
    end
  end
end
```

うーん、でも例えば context に含める期待挙動を「ユーザー目線のストーリーで書く」ルールにして内部実装とは切り離した上で、その期待されるパターンを実現するために必要なクラスが自身の挙動を調整する、とかなら何とかなる？
