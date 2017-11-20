+++
date = "2017-11-19T11:01:14+09:00"
description = ""
draft = true
tags = ["git", "github"]
title = "GitHub で Squash merge されたブランチを削除する"
updated = "2017-11-19T11:01:14+09:00"
+++

## TL;DR

<https://github.com/not-an-aardvark/git-delete-squashed>

## `git branch -d`できない問題

GitHub には[`Squash and merge`][squash-and-merge]という機能がある。
これを使うと Pull Request のコミットを1つにまとめてマージする事ができるため、
1コミットだけのわずかな変更でマージコミットを作りたくない時などには便利だ。

しかし squash して新たにコミットを作る以上、
Git のコミット履歴としてはブランチのマージが残らないため、
ローカルに残っている`Squash and merge`済みのブランチを自動で削除しづらいという問題がある。
例えば普通にマージされたブランチなら、以下のようなコマンドで簡単に一括削除できる。

[squash-and-merge]: https://help.github.com/articles/about-pull-request-merges/#squash-and-merge-your-pull-request-commits

```sh
git branch -d `git branch --merged | grep -v '*'`
```

だが`Squash and merge`された場合は`--merged`では検出できない。

これの解決方法を自分で考えてもあまりちゃんとしたアイデアが浮かばなかったので、
既存ツールを探してみたところ [not-an-aardvark/git-delete-squashed][git-delete-squashed] が見つかった。
普段は使わないような git のサブコマンドが活用されており、自分ではまず思いつかない方法が使われていたのでメモしておく。

[git-delete-squashed]: https://github.com/not-an-aardvark/git-delete-squashed

## `Squash and merge`されたブランチの検出方法

自分が見つけた時点でのコードはこちら: [f36f2c08:bin/git-delete-squashed.js][git-delete-squashed-bin]

[git-delete-squashed-bin]: https://github.com/not-an-aardvark/git-delete-squashed/blob/f36f2c0843c2adf01b3d2d80a03884e181f9f8ba/bin/git-delete-squashed.js

一言でいうと、ローカルで`Squash and merge`と同等のコミットを作成し、
同じコミットが`master`にないかを探すという方法。

例えば`master`というベースブランチと`foo`というフィーチャーブランチがある時、
`foo`が`master`に`Squash and merge`されたかどうかを判定するフローを考えてみる。
もしされていれば、squash された以上`foo`で作られたコミット自体は`master`に1つも残っていないが、
代わりに`foo`でなされた修正を全て含む1つのコミットがある事になる。
よって`master`の履歴にそのようなコミットが見つかれば、
`foo`が`Squash and merge`されたのだと判断できる。
`git-delete-squashed`は以下の方法でそのようなコミットを探している:

1. `foo`と`master`の分岐点となるコミット (共通の祖先) を探す
1. その祖先から`foo`の HEAD までを squash した一時コミットを作る
1. その一時コミットと**内容的に同一な**コミットを`master`の履歴から探す

重要なのは 3 の処理で、
コミットの SHA-1 ではなく内容でコミットの同一性を判定する方法が必要になる。
ここで使われるのが[`git cherry`][git-cherry]というコマンドである。

### `git cherry`の概要

例えば`master`と`foo`の履歴が以下のようになっているものとする。

```
        a   b   master
  ... - * - * - *
        |
        - - * - *
            b'  foo
```

- コミット`a`から`foo`が分岐している
- `b`と`b'`は別コミットだがコミット内容は同じである
- `master`と`foo`の HEAD の内容は異なる

内容は同じなのにコミットは別物という状況は、
[`cherry-pick`][git-cherry-pick]などを使うと簡単に起こる。
この状態で`foo`を`checkout`して`git cherry master`を実行すると、
`master`に対するコミット単位の差分を表示できる。

```sh
$ git cherry master
- hash-of-b'
+ hash-of-head
```

`hash-of-xx`の部分には実際にはコミットの SHA-1 ハッシュが表示される。
`-`と`+`はそれぞれ以下を意味する。

- `-`: 同じ内容のコミットが`master`にもある
- `+`: 同じ内容のコミットは`master`にはない

これでコミット`b`と`b'`は別物であっても内容的には同じ、という事がわかる。
このように、`git cherry`を使うとブランチ同士の内容的な差分をコミット単位で確認する事ができる。

つまり、`Squash and merge`されたコミットと同じ内容のコミットを作って`git cherry`で比較すれば、
`-`になるかどうかで`Squash and merge`されたかどうかを判定できる事になる。

### `Squash and merge`と同じコミットを作る

では、どのようにして`Squash and merge`されたコミットを再現するか。
`git-delete-squashed`では、
[`git merge-base`][git-merge-base]と[`git commit-tree`][git-commit-tree]というコマンドを使ってこれを実現している。

`merge-base`はそのままで、`git merge-base master foo`で共通祖先を見つける事ができる。
後はその祖先コミットから`foo`の HEAD までを squash した一時コミットを作ればいい。
しかし普通にやるとそのコミットはカレントブランチの履歴に追加されてしまう。
そこで`git commit-tree`という内部向けのコマンドが活用される。
これを使うと特定の[ツリーオブジェクト][git-objects] (リポジトリのファイル構成のスナップショット) に紐づくコミットを新たに作る事ができるのだが、
このコミットは単に作られるだけで、カレントブランチの履歴には追加されない。
そのため履歴を汚さずに一時コミットを作成できる。これで`Squash and merge`コミットを再現する事ができた。

ちなみに作られた一時コミットはどのブランチやタグにも紐付いていないため、時間が経てば自動で git が削除してくれるはず。

[git-merge-base]: https://git-scm.com/docs/git-merge-base
[git-commit-tree]: https://git-scm.com/docs/git-commit-tree
[git-cherry]: https://git-scm.com/docs/git-cherry
[git-cherry-pick]: https://git-scm.com/docs/git-cherry-pick
[git-objects]: https://git-scm.com/book/en/v2/Git-Internals-Git-Objects

## まとめ

まとめると、まず`git merge-base`で祖先コミットを探し、
次に`git commit-tree`で`Squash and merge`と同等のコミットを作り、
最後に`git cherry`でそのコミットと同等のものが`master`にあるかどうかを確認する、という手順になる。
これを各ブランチごとに繰り返して、該当するコミットがあるブランチは削除すれば良い。

`git-delete-squashed`は Node.js で実装されているが、
README には以上の処理を bash スクリプトで実行する方法も記載されており、
これを使えば`git-delete-squashed`をインストールする事さえなく目的を達成できる。
必要な処理は全てこのスクリプトに書かれているので、まとめとしてここにも載せておく。

```bash
git checkout -q master && \
  git for-each-ref refs/heads/ "--format=%(refname:short)" | \
  while read branch; do
    mergeBase=$(git merge-base master $branch) && \
      [[ $(git cherry master $(git commit-tree $(git rev-parse $branch^{tree}) -p $mergeBase -m _)) == "-"* ]] && \
      git branch -D $branch;
done
```

`git-delete-squshed`が完璧に動作するかは未検証だが、少し試した限りでは期待通りに動いてくれた。
`git cherry`や`git commit-tree`を知らなかった自分には、こんな実装自体とても思いつかない。
