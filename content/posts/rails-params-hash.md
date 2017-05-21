+++
date = "2016-04-22T00:00:00+09:00"
description = ""
draft = true
tags = ["ruby", "rails"]
title = "Rails: paramsとHashの微妙な違い"
updated = "2016-04-22T00:00:00+09:00"
+++

## 一言でいうと

`params`がSymbolとStringのキーを区別しないの知らなかった!
([普通にドキュメント化されてた](http://api.rubyonrails.org/classes/ActionController/Parameters.html#method-i-5B-5D))

## 問題

今回遭遇した状況を簡単に再現すると以下のような感じ。

```ruby
# UsersController#index
def index
  @users = User.search params['form']
  # ...
end

# User.search
def search(form = {})
  form.reverse_merge!({
    type: 0,
    name: ''
  })

  return User.all if form['type'] == 0

  # ...
end
```

これはあるWebサイトのコードで、そのサイトはトップページが検索画面になっており、最初に画面を開いたタイミングで検索が実行される。
画面にはフォームと検索ボタンがあり、その検索ボタンをクリックする事でも検索できる(フォームの内容を付加して
`#index`にリクエストする)。
初回アクセス時、およびフォームに何も入力せずに検索実行した時は、同じ検索結果になる想定だった。
ところが実際には、どちらもリクエストパラメータは空なのに、両者で検索結果が異なっていた。
コードを書いた人が期待していたのは、フォームが空の場合は`type: 0`というデフォルト値により`search`メソッドが`User.all`の結果を返すこと。
しかしそうなるのは検索ボタンから検索実行した場合だけで、初回検索時にはこの`type: 0`が上手く機能していないようだった。

## 原因

### `search`メソッドのバグ

RubyにはHashを生成するための構文が2種類あり、上記の`reverse_merge`の引数で使われている省略記法は、
全てのキーをSymbol型で登録する。ところがその下の`if`文では`form['type']`というように文字列をキーにして
値を取得しようとしている。Hashはキーの型を区別するため、`:type`(Symbol型)をキーにしてセットされた値を`'type'`(String型)
で取り出すことはできない。だから上記の`reverse_merge`によるデフォルト値のセットは全く意味がない事になる。
正しくは後続の処理でもSymbolのキーを使うか、以下のようにしてデフォルト値を文字列のキーでセットすれば良かった。

```ruby
form.reverse_merge!({
  'type' => 0,
  'name' => ''
})
```

しかしそれなら逆に、なぜ検索ボタンからの検索時には`form['type']`という記述でちゃんとデフォルト値が
取得できていたのか？

### paramsはHashWithIndifferentAccess

Railsは[ActiveSupport::HashWithIndifferentAccess][hash-with-indifferent-access]というHashのサブクラスを提供している。このクラスではSymbol型のキーとString型のキーは区別されないため、どちらのキーでセットされた値も、どちらのキーでも取得できる。
そして**`params`すなわち`ActiveSupport::Parameters`はこの`HashWithIndifferentAccess`を継承している**([Github][github-parameters])。つまり`params`に格納された値は、SymbolとStringのキーどちらでも取り出せるようになっている。

[hash-with-indifferent-access]: http://api.rubyonrails.org/classes/ActiveSupport/HashWithIndifferentAccess.html
[github-parameters]: https://github.com/rails/rails/blob/9ab2d030209d9608a6c866d83210f5b3b7d2319e/actionpack/lib/action_controller/metal/strong_parameters.rb#L108

### つまり起きていたのは..

#### 初回検索時

1. `params['form']`は`nil`
1. 引数の`form`にデフォルト値`{}`(通常のHash)が入る
1. `form[:type] = 0`
1. `form['type']` => `nil` (キーの型が違うから)

#### 検索ボタンからの検索時

1. `params['form']`は空の`ActiveSupport::Parameters`
1. 引数の`form`に`params['form']`が入る
1. `form[:type] = 0`
1. `form['type']` => `0` (`form`はHashWithIndifferentAccessだから)

## 教訓

* `params`とHashを一緒くたにして扱う時は気をつける。
* Hashのキーの型は統一する。下手にHashWithIndifferentAccessを使うより混乱しない。
* てか引数のHashを直接書き換える必要はない。
