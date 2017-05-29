+++
date = "2017-05-21T22:22:36+09:00"
description = "リベースがちょっと楽になる。"
draft = false
tags = ["git"]
title = "Gitの初回コミットは空にしとくと便利"
updated = "2017-05-21T22:22:36+09:00"
+++

Gitを使う場合、ローカルで雑にコミットを積み上げておいて、`push`前に履歴を整理する、というのは割とよくあるフローだと思う。
この時基本は`git rebase`を使うはずだけど、初回コミットに関しては`git rebase`が使えない。
最初のコミットにはリベース元となる親コミットが存在しないから当たり前だけど、それでも初回コミットを含めて履歴を整理したい事もある。
[`--root`][git-rebase-root]というオプションをつける方法はあるが、できれば初回コミットとか気にせず普通にリベースしたい。

そこで、例えば初回コミットでは空の`.gitignore`だけを追加するようにするとちょっと楽になる。
こうすればソースコードなどの管理対象ファイルは常に2つ目以降のコミットになるから、リベースがしやすい。  
またこの初回コミットはどのリポジトリでも共通なので自動化できる。
自分は[`setup`][git-setup]という名のエイリアスをGitに用意して、`git init`の代わりにこれで初期化を行うようにしている。内容は以下:

```sh
git init \
    && touch .gitignore \
    && git add .gitignore \
    && git commit -m 'Init repository'
```

後は普通にコミットを重ねていけば、何も考えずリベースできる。  
最初は`--allow-empty`で初回コミットの内容を本当に空にしてたけど、`.gitignore`はどんなリポジトリでもほぼ必ず使うので最初から作るようにした。

そもそも最初のコミットには履歴のスタート地点を作るという責務が既に1つある気がするので、
初回コミットにはそれだけをやらせて、管理対象のファイルを追加していくのは2回目以降のコミットでやっていく方が綺麗なのでは、という思いもある。

[git-rebase-root]: https://git-scm.com/docs/git-rebase#git-rebase---root
[git-setup]: https://github.com/ryym/dotfiles/blob/e8f1814bb34cf7698cf477aa449442db842930a9/dotfiles/git/gitconfig#L17

