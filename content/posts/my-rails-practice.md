+++
date = "2019-07-09T10:18:03+09:00"
description = ""
draft = true
tags = ["ruby", "rails"]
title = "Rails 開発で意識していること"
updated = "2019-07-09T10:18:03+09:00"
+++

Hanami
あのスライド

- DSL とロジックを分ける
- 責務を分離する
- 依存を差し替え可能にしてテスタブルにする

## ??

- Mixin は使わない
- Model は基本 Entity + Repository に専念
- ビジネスロジック用の層をめっちゃざっくり`lib`
- Controller はやっぱり薄く ActionHandler で書けたらいいのに

## 補足: そもそもなぜ？

(前回の要点を)

## Tips

- `timestamp`は前に
- RuboCop 使う
- modifier prefix ありかも

## よく使う gem

- `rubocop`
- `enum_help`
- `activerecord-import`

