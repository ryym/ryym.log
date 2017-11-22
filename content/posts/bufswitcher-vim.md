+++
date = "2017-07-26T21:49:22+09:00"
description = ""
draft = true
tags = ["vim"]
title = "Vim: バッファ切り替え用の簡易プラグインを作った"
updated = "2017-07-26T21:49:22+09:00"

+++

## TL;DR

- [bufswitcher.vim][bufswitcher.vim]というバッファ切り替え用のVimプラグインを作った
- ないよりはあった方が便利、程度には使えて良かった

[bufswitcher.vim]: github.com/ryym/bufswitcher.vim

## WIP

- 基本は unite.vim (や ctrlp.vim) で充分
- 3つ以上のバッファを頻繁に切り替える時にちょっとダルい
- もっとステップ数少なくバッファを切り替える方法を併用する
    - ステータスラインを利用してバッファを切り替え、「バッファを選ぶ」動作を不要にする
    - 少ない数のバッファを行き来する時はそれなりに使える
- 余談 1,2年前に作りかけてたやつを完成させた テストもあって割とちゃんとしてた
- vim-submode と併用する事でより便利になる
