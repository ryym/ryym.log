+++
date = "2017-07-13T22:38:44+09:00"
description = "サブクラスからも見えないメソッドをモジュールに定義する"
draft = false
tags = ["ruby"]
title = "Ruby: Mixinモジュールに private メソッドを作る"
updated = "2017-07-13T22:38:44+09:00"

+++

## TL;DR

- Mixin用のモジュールで、それをインクルードするクラスからも見えないような private スコープが欲しい。
- Ruby2.4以降なら、自分の中で自分自身を refine する事でそれっぽい事はできる。
- 通常の private とは違うしイマイチな点もある。

## サブクラスからも見えない private スコープ

Mixin用にモジュールを作った時、「それをインクルードするクラスからも見えない private スコープ」
が欲しくなる事ってありませんか？

Rubyの private はそのクラス/モジュールを継承するクラスからも見えてしまうので、
Mixin用モジュールの中で private メソッドを定義しても、それをインクルードするクラスから呼べてしまいます。
特に、例えば Rails の Helper のように色んなところでインクルードされるモジュールでは、
インクルードするクラスから見えない private スコープが欲しくなる時が自分はあります。

```ruby
class Animal
    private

    def secret
        "secret of life"
    end
end

class Dog < Animal
    def bow
        "I know #{secret}"
    end
end

puts Dog.new.bow  # => "I know secret of life"
```

解決方法をちょっとググってみると、モジュール内に別のクラスを定義してその中にメソッドを置く方法がありました ([例][dont-mix-in-your-privates])。
実際に private になるわけではありませんが、ネームスペースの汚染を防ぐには有効そうです。
とはいえクラスが変わると`self`も変わってしまって何だかなーという気がするし、
いちいちクラスを定義するのも面倒です。

[dont-mix-in-your-privates]: https://thepugautomatic.com/2014/02/private-api/

## ActiveSupportが使っていた方法

そんな中、先日社内勉強会でActiveSupportのソースを読んでいた時に面白いコードを見つけました。
`Array#sum`を拡張する以下のようなコードです ([source][activesupport-array-sum])。

[activesupport-array-sum]: https://github.com/rails/rails/blob/d1b2b139a55aa0bda082a41946373374ff3e90a0/activesupport/lib/active_support/core_ext/enumerable.rb#L142-L147

```ruby
module Enumerable
    # ...

    # Using Refinements here in order not to expose our internal method
    using Module.new {
        refine Array do
            alias :orig_sum :sum
        end
    }

    class Array
        def sum(init = nil, &block) #:nodoc:
            if init.is_a?(Numeric) || first.is_a?(Numeric)
                init ||= 0
                orig_sum(init, &block)
            else
                super
            end
        end
    end
end
```

ここでは`sum`メソッドを再定義する前に、
元々ある`Array#sum`にもアクセスできるように`orig_sum`という alias をつくっているのですが、
興味深いのはその処理が`using`と`refine`によるブロックの中で行われている点です。

### `using`と`refine`

[Refinements][ruby-refinements]はRuby2.0から導入された機能で、
限定した範囲でのみオープンクラスによるクラス拡張を有効にする事ができます。

[ruby-refinements]: https://ruby-doc.org/core-2.1.1/doc/syntax/refinements_rdoc.html

```ruby
module StringExtensions
    refine String do
        def loudly(volume = 3)
            "#{self}#{'!' * volume}"
        end
    end
end

module Hello
    using StringExtensions

    def self.world
        "hello, world".loudly
    end
end

puts Hello.world # => "hello, world!!!"
puts "good bye".loudly # => NoMethodError
```

しかし先程ActiveSupportが使っていた Refinements は、このような例とは違いました。
その場で無名モジュールを作り、その中で`refine`を使って`Array`に alias を追加し、即座にそれを`using`しています。
このコードは`Enumerable`モジュール内に書かれているので、こうすると`Enumerable`内でのみ`orig_sum`メソッドが使えるようになります。
つまり、`orig_sum`という alias を外部に公開する事なく使えるわけです。面白いですね。

ところでこの方法って、通常のモジュールでも使えないでしょうか？
つまり、自分自身の中でのみ自身を`refine` & `using`してメソッドを定義すれば、
自分自身の中でのみ使用可能なメソッドを定義できるのではないでしょうか
(言うなれば self-refinements?)。

[ruby-2.4-news]: https://github.com/ruby/ruby/blob/v2_4_0/NEWS

## Self Refinements による擬似 private

実は、 2.3 までの Ruby ではモジュールでこの方法を使う事は不可能でした。
`refine`はクラスにしか使えず、モジュールを渡すとエラーになっていたからです。
しかし 2.4 からはモジュールにも使えるようになったため ([release note][ruby-2.4-news])、
それが可能になりました。実際にやってみます。

```ruby
module Greeter
    using Module.new {
        refine Greeter do
            def world
                "world"
            end
        end
    }

    def hello
        "hello, #{world}"
    end
end

class Stuff
    include Greeter

    def greet
        hello
    end

    def greet2
        world
    end
end

puts Stuff.new.greet   # => "hello, world"
puts Stuff.new.greet2  # => undefined local variable or method 'world'
```

動きました。
もう少し簡単に書けるようにするとこんな感じでしょうか。

```ruby
class Module
    # Add `privates` method to Module class.
    def privates(&block)
        target = self
        Module.new do
            refine target do
                class_eval &block
            end
        end
    end
end

module Greeter
    # Define private-like methods using Refinements feature.
    using privates {
        def world
            "world"
        end
    }

    def hello
        "hello, #{world}"
    end
end

class Stuff
    include Greeter

    def greet
        hello
    end

    def greet2
        world
    end
end

puts Stuff.new.greet   # => "hello, world"
puts Stuff.new.greet2  # => undefined local variable or method 'world'
```

`using`をメソッド内で使う事は禁止されているので、`refine`の部分だけメソッド化しています。
これでもう少し簡単に書けるようになりました。
この方法なら、サブクラスからも見えない private スコープを手軽に実現する事ができます。

### 問題いろいろ

が、この方法にはいくつかイマイチな点があります。

#### 定義ブロックに`do ... end`を使えない

以下のように`privates`に渡すブロックを`do end`で記述すると、
そのブロックは`using`への引数だと解釈されてしまうため、
代わりにブレース (`{}`) を使う必要があります。

```ruby
# ArgumentError: wrong number of arguments
using privates do
    def world
        "world"
    end
end
# => `using(privates) do; end`

# OK
using privates {}  # => using(privates {})

# OK
using (privates do; end)
```

複数行になるブロックには`do end`を使う方が一般的だと思うので、
ブレースで書かないといけないのはちょっと気持ち悪いです。

#### `using privates`の定義場所に気をつけないといけない

`using`文は、それによって拡張されるメソッドを使用している箇所よりも前に記述する必要があるようです。
例えば以下のように順番を変えて`hello`を実行すると、「`world`は未定義です」というエラーになってしまいます。

```ruby
module Greeter
    def hello
        "hello, #{world}"  # Error
    end

    using privates {
        def world
            "world"
        end
    }
end
```

#### 定義したメソッドを`send`で呼べない

これは当然ですが、
Refinements が効いていないモジュールの外からは`refine`ブロックで定義されたメソッドは全く見えないので、
通常の private とは違い`send`を使っても呼び出す事ができません。
逆に言えば、他の言語にあるような完全な private を実現できる、とも言えるかもしれませんが。

## まとめ

自身の Refinements を自身内でのみ使う事で、外部からは見えないメソッドを定義できました。
この「外部」にはサブクラスも含まれるので、
Mixin用モジュール内でのみ使える private-like なメソッドを定義できます。
また通常のクラスでも、サブクラスに公開したくないメソッドを定義するのに使えます。
ただし、記述方法は微妙と言わざるを得ません。

