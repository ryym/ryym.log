+++
date = "2017-06-29T21:14:30+09:00"
description = "ruby -cwe 'http://example.com'"
draft = false
tags = ["ruby"]
title = "Rubyで http://example.com を実行する"
updated = "2017-06-29T21:14:30+09:00"

+++

## TL;DR

```shell
$ ruby -cwe "https://example.com"
Syntax OK
```

## これを知った経緯

今日読んだ[とあるブログ][blog_dont_comment]に書いてあった。  
そのブログではソースコード内のコメントのうち、
Rubyコードとして解釈可能なもの (コメントアウトされたコード) を検出するツールについて語っていて、
その中に以下のような一文が出てくる。

[blog_dont_comment]: http://pocke.hatenablog.com/entry/2017/04/15/130331

> `http://example.com`は Ruby コードとして解釈することが可能であるが、人間からしたら URL に見えるだろう。


......**え、そうなの?**

`http://example.com`がRubyコードとして解釈可能?  
このURLの一体どこがRubyコードなんだろう?  

しかしそのブログでは、この点についてはこれ以上言及されていない。
半信半疑で試してみると、TL;DR にあるように本当にRubyコードとして解釈できた。

でもなぜなのだろう? 
ずいぶんサラっと書かれていたけど、Rubyに詳しい人にとっては驚く事でもないのだろうか?  
自分には不思議だったので調べてみた。

```shell
$ # Environment: macOS Sierra
$ ruby --version
ruby 2.4.1p111 (2017-03-22 revision 58053) [x86_64-darwin16]
```

## Rubyは`http://example.com`をどう解釈するか

まずは構文をチェックするだけでなく、
`http://example.com`を`a.rb`というファイルに保存して実際に実行してみる。

```ruby
# a.rb
http://example.com
```

すると以下のようなエラーが出た。

```shell
$ ruby a.rb
a.rb:2:in `<main>': undefined local variable or method `example' for main:Object (NameError)
```

`example`というオブジェクトは存在しないというエラーだ。
どうやらあの文字列をRubyコードとして実行した場合、`example`という部分が最初に評価されるらしい。

`example.com`の部分は唯一、自分でもRubyとして解釈可能だとわかる箇所だ。
`example`をレシーバとした単なるメソッド呼び出しだろう。
そこで`com`というメソッドを持つ`example`というオブジェクトを定義して、再度実行してみる。

```ruby
example = Struct.new(:com).new
http://example.com
```

```shell
$ ruby a.rb
a.rb:2:in `<main>': undefined method `/' for :/:Symbol (NoMethodError)
```

今度は`:/`というシンボルに`/`というメソッドがないと言われた。
なるほど`:/`はシンボルであり、続く`/`はそのメソッドとして解釈されていたのか!

ここまで来れば`http://example.com`がRubyコードとして解釈可能な理由がわかる。
つまりはこういう事だ。

```ruby
# 括弧をつけて糖衣構文をなくすとこうなる
http(:/./(example.com))

# もっと冗長に書くと:

# 1. `example`オブジェクトの`com`メソッドを呼び出して
v1 = example.com

# 2. その戻り値をシンボル`:/`の`/`メソッドに引数として渡し
v2 = :/ / v1

# 3. 更にその戻り値を`http`というメソッドに渡している
http(v2)
```

な、なるほど...。`http:/`や`://`なんて書き方ありだったのか。

### 鍵はメソッド呼び出しの糖衣構文

肝となるのは以下の2点だと思う。

- `/`などの演算子メソッドを呼び出す場合、`.`と引数を囲う括弧orスペースを省略できる

    ```ruby
    1.+(1)
    1+1
    4/4
    ```

- メソッドの第一引数がシンボルや文字列の場合、括弧のみならずスペースも省略できる

    ```ruby
    puts(:bar)
    puts :bar
    puts:bar # valid
    puts"bar" # valid
    ```

前者はまあそうだろうという感じだけど、後者は知らなかった。  
でも後者のような書き方を許容するメリットってあるのかな...? 😅 

## 実際に実行してみる

というわけでようやく構文が理解できたので、実際に`http://example.com`を実行できるようにしてみよう。

```ruby
# Add `/` method to Symbol class.
class Symbol
  def /(value)
    "#{self}/#{value}"
  end
end

# Define `http` method.
def http(path)
  "http:#{path}"
end

# Create `example` object.
example = Struct.new(:com).new("example.com")

# Run.
result = http://example.com
puts result
```

```shell
$ ruby a.rb
http://example.com
```

できた!

まさかただのURLが有効な式になりうるとは。Rubyの構文の奥深さを垣間見た日だった。

### 場所によっては解釈できない

ちなみに上記のスクリプトの最後の部分で、
`http://example.com`の結果を直接`puts`に渡そうとすると構文エラーになる。

```ruby
# ...

puts(http://example.com)
```

```shell
$ ruby a.rb
a.rb:13: unknown regexp options - apl
```

一瞬あれ!? となるけど、理由はエラーメッセージに思いっきり書いてある。

> unknown **regexp** options - apl

なんと今度は`//`が正規表現リテラルと解釈されたようだ。
もうおわかりだと思うが、
この書き方だと`http:`の部分がハッシュのキーとして解釈されている。
そしてその値に`//`という空の正規表現が指定されており、
更にそのオプション・修飾子として`example`という文字列が指定されている形になる。
しかし`a`, `p`, `l`は無効なオプションなので、
そんなオプションはないというエラーになっていた ([doc][rubydoc-regexp])。  
この場合`.com`は正規表現のインスタンスメソッドになるので、
それさえ実装すれば以下のようなURLは実行可能になる。

[rubydoc-regexp]: https://ruby-doc.org/core-2.4.1/Regexp.html#class-Regexp-label-Options

```ruby
# valid options/encodings: i,m,x,o, u,e,s,n
puts(http://mixin.com)
```

面白い。  
探せば他にも面白いケースがあるかもしれない。

## 感想

こういうケースから察するに、Rubyの構文解析って実は相当複雑な仕事なんじゃないだろうか。
すごいなぁ。
