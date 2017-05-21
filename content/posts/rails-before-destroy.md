+++
date = "2016-03-22T00:00:00+09:00"
description = "before_destroyの実行順序を考えないと、バリデーションが予期しない動作になる事がある。"
draft = false
tags = ["ruby", "rails"]
title = "Rails: before_destroyによる関連チェックに注意"
updated = "2016-03-22T00:00:00+09:00"

+++

既に色んな方が記事にされていますが、さっそく自分もハマったのでメモ。
(環境: Rails 4.1.1, Ruby 2.1.0)

### やりたかった事

あるモデルを削除する前にバリデーションを走らせ、バリデーションが通った時のみ削除するようにする。
そのために、[before_destroy]コールバックを登録する。

[before_destroy]: http://api.rubyonrails.org/classes/ActiveRecord/Callbacks.html

### ハマった事

`before_destroy`に登録したメソッドを単体で実行してみると正しく動くのに、
いざ`destroy`すると期待する挙動にならない。例えば以下のようなコードだと、
`before_destroy`のバリデーションが必ず通ってしまう。

```ruby
class User < ActiveRecord::Base
  has_many :events, dependents: :destroy

  before_destroy :check_all_events_finished

  def check_all_events_finished
    now = Time.zone.now
    if events.where('? < start_time', now).exists?
      errors[:base] << '未終了のイベントがあります'
    end
    errors.blank?
  end
end
```

### 原因

今回の場合は`before_destroy`の中で関連するモデルの存在をチェックしているのですが、
上記のコードではその**関連がbefore_destroyの前に削除されていた**事が原因でした。
そうなるとそもそも紐づく`event`がなくなるため、このバリデーションは常に`true`を返してしまいます。

問題は、例における`has_many`の関連が`dependents`オプションを持っている点にあります。というのも、

* `dependents`オプションの処理は`before_destroy`コールバックを使って実現される([Github][rails-add-before-destroy])
* `before_destroy`などのコールバックは定義した順番に実行される

[rails-add-before-destroy]: https://github.com/rails/rails/blob/d5c4b82b64f3cdd511eb79c4d43d6ab1548c0dee/activerecord/lib/active_record/associations/builder/association.rb#L88

からです。つまり上記の例でいうと、`has_many`を定義した段階で関連を削除するための`before_destroy`がこっそり登録されており、
バリデーション用に定義した`before_destroy`はその`dependent: :destroy`用コールバックの後で実行されるため、
バリデーション時には`events`が必ず空を返すようになっていたのでした。

### 解決策

下記いずれかの方法でバリデーション用の`before_destroy`を先に走らせるようにすれば、この問題は回避できます。

#### 1. `before_destroy`を`has_many`定義より前に宣言する

```ruby
class User < ActiveRecord::Base
  before_destroy :check_all_events_finished

  has_many :events, dependents: :destroy

  # ...
```

#### 2. `prepend`オプションを`true`にする

このオプションつきで登録されたコールバックは、コールバックキューの末尾でなく先頭に追加されます
([Ordering callbacks](http://api.rubyonrails.org/classes/ActiveRecord/Callbacks.html#module-ActiveRecord::Callbacks-label-Ordering+callbacks))。

```ruby
class User < ActiveRecord::Base
  has_many :events, dependents: :destroy

  before_destroy :check_all_events_finished, prepend: true

  # ...
```

原因がわかれば納得はできるものの、一見無関係に見えるクラスマクロの定義順序によって動作が変わってしまうというのはわかりづらいですね。
Githubを見てみると2011年から既にissueが上がっていました([rails/rails#3458])。

[rails/rails#3458]: https://github.com/rails/rails/issues/3458

### 参考

* https://github.com/rails/rails/issues/3458
* http://api.rubyonrails.org/classes/ActiveRecord/Callbacks.html
* http://gdgd-shinoyu.hatenablog.com/entry/2015/04/24/034238
* http://ameblo.jp/axio9da/entry-10810821007.html
