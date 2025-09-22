+++
draft = false
date = "2025-09-20T21:32:19+09:00"
tags = ["fzf"]
title = "fzf に Vim-like なモード機能をつける"
updated = "2025-09-20T21:32:19+09:00"
+++

[fzf]: https://github.com/junegunn/fzf

CLIの fuzzy finder として [fzf] を長らく愛用しています。
単純なフィルターとして使うだけでも十分便利ですが、豊富なオプションを組み合わせることで思いの外自由な使い方ができます。
ここではその一例として、 fzf に Vim っぽい簡易的なモード切替機能をもたせ、キーバインディングの幅を広げる小技を紹介します。

## `unbind`/`rebind` による簡易モード切替

fzf では `--bind` で独自のキーバインディングを指定できます。

```bash
ls -l | fzf --bind='ctrl-k:kill-line' --bind='ctrl-/:toggle-preview'
```

しかし指定できるキーの種類には限りがあるし、 `ctrl-b,a` のような複合キーへのバインディングもできません。
また a-z, 1-9 の英数キーは通常の文字入力に使う以上、それらのキーバインディングは変更しづらいです。
こんな時、 Vim を使う人であれば「 Vim みたいにモード切替ができたら、英数キーにも自由にキーをバインディングできるのに」と思うかもしれません。

[vim]: https://www.vim.org/

Vim とは老舗のテキストエディターで、テキストを入力するには `i` などで入力モードに切り替える必要のある変わったエディターです。
その代わり通常のモードでは、 a-z など普通ならテキスト入力に使うあらゆるキーを自由に割り当てられるので、様々なアクションを簡潔なキー操作で実行できます。 `j,k,h,l` によるカーソル移動などは代表的な例ですね。

Vim ではいわば、1つのキーがモードに応じて複数のバインディングを持つわけです。
fzf でもそのようなモード切替ができたら便利じゃないでしょうか？
それを実現するには、実行中にキーのバインディングを切り替えられる必要があります。
fzf はそのような動的なバインディングをサポートしていないものの、 `unbind` / `rebind` アクションを使うことでそれに近い体験を実現できます。

- `unbind` ... 指定したキーのバインディングを無効にする。
- `rebind` ... 無効になっているキーバインディングを再度有効にする。

具体的には以下のようにします。
(途中にコメントを挟めるよう、 fzf へのオプションを配列にしてます)

```bash
fzf_options=(
    --prompt='Custom:'
    # j,k で上下移動。
    --bind='j:down'
    --bind='k:up'
    # i, esc でモード切替。 prompt もモードに応じて変える。
    --bind='i:unbind(j,k,i)+change-prompt(Filter:)'
    --bind='esc:rebind(j,k,i)+change-prompt(Custom:)'
)
ls -l | fzf "${fzf_options[@]}"
```

`j`,`k` を上下移動に割り当てています。すると当然 j,k という文字を入力できなくなりますが、 `i` で `unbind` することで `j`,`k` を通常の入力モードに戻せます。
`esc` または `ctrl-[` を押すと、 `rebind` により `j`, `k` は再び上下移動用のキーとなります。
まさに Vim っぽい挙動です。

このように `unbind` と `rebind` を使うことで、そのキーのデフォルトの挙動に加えてもう1つ独自のアクションを追加でき、擬似的にモード切替っぽい動きを実現できます。
大文字小文字に加えて数字・記号なども含めれば、実に 70 以上のキーに好きなアクションを設定できることになり、夢が広がりますね。

なお通常入力ではないモードの方、 `j`,`k` が上下移動になるモードを、以降では仮にカスタムモードと呼んでおきます。
上記例では `fzf` 起動時にはカスタムモードになりますが、以下のオプションを加えれば通常の入力モードで起動するようにもできます。

```bash
--bind='start:unbind(j,k,i)+change-prompt(Filter:)'
```

`i` と同じアクションを `start` イベントに割り当てることで、起動時にカスタムモードを解除しています。 fzf ではこのように、キー押下以外の各種イベントに対してもアクションを割り当てられます。
こうすると `esc` を押さない限りは通常の `fzf` なので、デフォルトでカスタムモードを使用可能にしておき、 複雑なことをしたい時だけカスタムモードを活用して色々なキーバインディングを定義する、といった使い方が可能です。

例えばカスタムモード時にはデフォルトで全てのキーを使用不可にしておき、

```bash
key_list=(a b c d e) # ... X Y Z)
keys=$(IFS=','; echo "${key_list[*]}")
fzf_options=(
    --prompt='Custom:'
    --bind='i:unbind($keys)+change-prompt(Filter:)'
    --bind='esc:rebind($keys)+change-prompt(Custom:)'
)
for key in "${key_list[@]}"; do
    fzf_options+=("--bind='$key:ignore'")
done
FZF_DEFAULT_OPTS="${fzf_options[*]}"
```

個別の用途で適宜カスタムモードのキーバインディングを上書きしたり。

```bash
git log --oneline -100 | fzf \
    --bind='y:become(echo {1} | pbcopy)' \
    --bind='R:execute(git rebase -i {1})+reload(git log --oneline)' \
    # ...
```

## Command-line-like mode

Vim にはコマンドラインモードというのもあり、キーバインディングの代わりにコマンド入力を通じて様々な操作を実行できます。
VSCode でいう `ctrl-shift-p` のようなものです。
このコマンドラインモードについても、頑張ればそれっぽいものを fzf で実現できます。

```bash
keys="j,k,i,:"
fzf_options=(
    ### Custom mode (さっきのやつ)
    --prompt='Custom:'
    --bind='j:down'
    --bind='k:up'
    --bind="i:unbind($keys)+change-prompt(Filter:)"
    --bind="esc:rebind($keys)+change-prompt(Custom:)"

    ### Command-line mode
    # ':' でコマンドラインモードに入る。
    --bind="::unbind($keys)+change-prompt(Command:)+disable-search"
    # Enter: コマンドラインモードならコマンドのハンドリング、それ以外ではデフォルトの動作。
    --bind='enter:transform{
        case "$FZF_PROMPT" in
            Command:)
                reset="change-prompt(Filter:)+clear-query+enable-search"
                case "$FZF_QUERY" in
                    last) # 最後のアイテムへ移動
                        echo "last+$reset" ;;
                    "sort:"*) # ソート変更 (e.g. "sort:size" -> sort by size)
                        value="${FZF_QUERY#*:}"
                        echo "reload(ls -l --sort=$value)+$reset" ;;
                    *)
                        echo "$reset" ;;
                esac ;;
            *)
                echo "accept" ;;
        esac
    }'
)
ls -l | fzf "${fzf_options[@]}"
```

これで `:` でコマンドラインモードに入り、 任意のコマンドを入力して `Enter` でアクションを実行、という動きになります。
以下のアクションを活用しています。

- `disable-search` (`:`) ... クエリを入力しても一覧が絞り込まれなくなる。これを利用して、一時的にクエリ入力欄をコマンド入力欄として使う。
- `transform` (`Enter`) ... 実行するアクションを動的に変える。上記例では prompt を見てコマンドラインモードかどうかを判定し、コマンドラインモードであればクエリの内容を確認して、それに応じたアクションを実行している。

`transform` に限らず、シェルスクリプトを実行できるアクションでは `FZF_PROMPT` や `FZF_QUERY` など fzf の状態を示すいくつかの環境変数にアクセスできます。

## 複合キーへのバインディング

コマンドラインっぽいモードを更に応用すれば、本来できない複合キーへのバインディングも夢ではありません。
同じように `disable-search` して入力を行い、特定の値になったらその瞬間にアクションを実行すればいいだけです。

```bash
# s,g を unbind/rebind の対象に追加
keys="j,k,i,s,g,:"

fzf_options=(
    ### Custom mode
    --prompt='Custom:'
    # ...

    ### Multi-key mode
    # s, g が押されたら複合キーモードへ。
    --bind="g:change-query(g)+unbind($keys)+change-prompt(Keys:)+disable-search"
    --bind="s:change-query(s)+unbind($keys)+change-prompt(Keys:)+disable-search"
    # 複合キーモードの間はクエリの変更を監視し、キー入力に応じてアクションを実行。
    --bind="change:transform{
        if [ \$FZF_PROMPT = 'Keys:' ]; then
            reset='change-prompt(Custom:)+clear-query+enable-search+rebind($keys)'
            case \"\$FZF_QUERY\" in
                gg) echo first+\$reset ;;
                ss) echo \"reload(ls -l --sort=size)+\$reset\" ;;
                st) echo \"reload(ls -l --sort=time)+\$reset\" ;;
                sn) echo \"reload(ls -l --sort=name)+\$reset\" ;;
            esac
        fi
    }"
)
ls -l | fzf "${fzf_options[@]}"
```

これで `gg` により先頭のアイテムへ移動、 `st` で結果を更新日時順にソート、 `ss` ならサイズ順、など複合キーによるアクションを実現できます。

肝は `change` イベントへのバインディングです。これによりクエリの入力を監視し、クエリが変わるたびにアクションを実行しています。
コマンドラインモードとは違って `Enter` を押さずに即アクションを実行することで、複合キーバインディングっぽくしています。

ここまでくれば、 fzf 内で実現できるキーバインディングのパターンはほとんど無限といってよいでしょう。

## まとめ

以上、 fzf で Vim っぽいモード切替を実現する設定を紹介しました。

{{< details summary="各種モード切替のサンプルコードまとめ" >}}

```bash
keys="j,k,i,s,g,:"
fzf_options=(
    ### Custom mode
    --prompt='Custom:'
    --bind='j:down'
    --bind='k:up'
    --bind="i:unbind($keys)+change-prompt(Filter:)"
    --bind="esc:rebind($keys)+change-prompt(Custom:)"

    ### Command-line mode
    --bind="::unbind($keys)+change-prompt(Command:)+disable-search"
    --bind='enter:transform{
        case "$FZF_PROMPT" in
            Command:)
                reset="change-prompt(Filter:)+clear-query+enable-search"
                case "$FZF_QUERY" in
                    last)
                        echo "last+$reset" ;;
                    "sort:"*)
                        value="${FZF_QUERY#*:}"
                        echo "reload(ls -l --sort=$value)+$reset" ;;
                    *)
                        echo "$reset" ;;
                esac ;;
            *)
                echo "accept" ;;
        esac
    }'

    ### Multi-keys mode
    --bind="g:unbind($keys)+change-prompt(Keys:)+disable-search+change-query(g)"
    --bind="s:unbind($keys)+change-prompt(Keys:)+disable-search+change-query(s)"
    --bind="change:transform{
        if [ \$FZF_PROMPT = 'Keys:' ]; then
            reset='change-prompt(Custom:)+clear-query+enable-search+rebind($keys)'
            case \"\$FZF_QUERY\" in
                gg) echo first+\$reset ;;
                ss) echo \"reload(ls -l --sort=size)+\$reset\" ;;
                st) echo \"reload(ls -l --sort=time)+\$reset\" ;;
                sn) echo \"reload(ls -l --sort=name)+\$reset\" ;;
            esac
        fi
    }"
)
ls -l | fzf "${fzf_options[@]}"
```

{{< /details >}}

まあ正直、モード切替が必要になるほど複雑なことを fzf でしたい場面は少ないと思いますが、やろうと思えばこういうこともできるのが面白いところです。 fzf も Vim も好きなので、有効活用できる場面を積極的に探していこうと思います。
