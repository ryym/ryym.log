+++
draft = false
date = "2019-12-02T12:10:34+09:00"
description = ""
tags = ["vim"]
title = "Vim: 編集可能なファイラを作った"
updated = "2019-12-09T10:08:00+09:00"
+++

## TL;DR

[viler]: https://github.com/ryym/vim-viler

[Viler][viler] という Vim 用ファイラプラグインを作りました。
目玉機能は、普通のテキストと同じように編集してファイルの移動や削除などができる点です。

![demo](https://raw.githubusercontent.com/ryym/vim-viler/master/docs/demo.gif)

Vim 上で快適にファイル操作したい方にオススメ。

## モチベーション

[vimfiler]: https://github.com/Shougo/vimfiler.vim

Vim のファイラは長らく [vimfiler][vimfiler] を使っていましたが、もう随分前に開発停止が宣言されている事もあり、先日ようやく別のファイラに移行する事にしました。
しかしいくつかのプラグインや Vim の組み込みファイラである Netrw を試してみたものの、いずれも欲しい機能の一部が欠けているか、ファイル操作のインターフェイスに満足いきませんでした。

ファイラに求めるもの:

- 動作が軽い。
- ツリー表示が出来る。
- 複数ウィンドウに別々のディレクトリを開ける。
- ファイル操作 (追加、移動、コピー、削除) がしやすい。
- (できれば) 複数ファイルを一度に操作できる。

特にファイル操作に関しては、もともと使っていた vimfiler も含めてどこかしっくり来ていない感覚を持っていました。
単一ファイルの削除などちょっとした操作なら出来るものの、例えばファイル名の一括変換など少し複雑な処理になると、途端に Vim を出てシェルスクリプトや OS のファイラに頼る必要がある事が多かったからです。
これは単にプラグインを使いこなせていないという面もあるのですが、一方でもっと直感的なインターフェイスにならないものかという思いもありました。
そこでじゃあどんなインターフェイスなら良いのかをこの機に考えてみた時、突き詰めると自分の欲求は「**普通にテキストを編集する感覚でファイルを操作したい**」というものだと気づきました。
すなわち:

- ファイルを追加するには行を追加すればいい。
- ファイルを削除するには行を削除すればいい。
- ファイルを移動・コピーするには行を移動・コピーすればいい。

といったインターフェイス。
つまりは普段のテキスト操作が、そのままファイル操作になる。
これならいつも使っているキーマッピングを使って、いつも通りにテキストをいじるだけでファイルを操作できます。
いち Vim ユーザとしてそんなファイラがあったら使ってみたいと思えたので、自作する事にしました。  
[Viler][viler] はまさしくそのようなインターフェイスの実現を目指すファイラプラグインです。
現時点では以下の機能を持っています。

- ファイルをツリー表示できる。
- 複数ウィンドウ (バッファ) にファイラを開き、別々に閲覧できる。
- ファイラのテキストを編集してファイル構成を操作できる。

ファイルの移動やコピーはウィンドウをまたいで行う事もできます。
これは離れた場所にあるファイルの移動・コピーに便利です。  
編集機能に関しては冒頭にある GIF の通り、ファイラのテキストを一通り編集した後、それを保存すると実際のファイルシステムに変更が適用される仕組みになっています。
以降ではこのプラグインについてもう少し詳しく説明します。

## ステータス

現在 v0.0.1 です。
自分で試す限り、単純なファイル操作に関しては問題なく動いていますが、対処できてないエッジケースがまだまだあるかもしれません。
Windows 対応もまだです。

自分の動作確認環境:

- MacOS Mojave 10.14.6
- Vim 8.1, Neovim 0.3.8

## 使い方

[viler-help]: https://github.com/ryym/vim-viler/blob/master/doc/viler.txt

インストール方法などの概要は [README][viler] を参照してください。

前述の通りファイル操作に関してはただのテキスト編集なので、特に設定は不要です。
よって主に必要なのはファイルの閲覧に関するキーマッピングとなります。

- ディレクトリの移動
- ディレクトリツリーの開閉
- カーソル上のファイルオープン
- etc

あとは `viler#open()` 関数でファイラバッファを開けば、定義したキーマッピングでファイルツリーを閲覧しつつ、必要に応じて編集しては保存する、という感じに使えます。  
自分は Vim の左側に細いファイラを常に表示しとくような使い方をよくしますが、編集機能だけをメインに使うのも良さそうです。
例えばファイル名を一括変更したい場合に Viler を開けば、 Vim の矩形選択や置換、マクロといった豊富なテキスト操作を用いてファイル構成を編集できます。

ただし、編集機能に関してはいくつかの注意事項や制限があります。例を挙げると:

- ファイルシステムに適用済みの変更を undo する事は現状できません。すなわち一度変更を保存した後、 undo して再度保存する事で変更前のファイル構成に戻す、という操作はできません。
- ファイルを追加する場合、どのパスに作成するかが一意に定まるよう、インデントを正確につける必要があります。

他にもあるので、こういった注意事項や詳しい使い方やについては [help ドキュメント][viler-help]を参照してください。


### 閲覧操作のキーマッピングについて

キーマッピングに関して他のファイラプラグインと違うのは、編集機能を考慮すると空いているキーがかなり限られるという点です。
基本的なカーソル移動や編集のしやすさを考えると、少なくとも `a` ~ `z` のキーは全部残しておきたいところです。
となるとマッピングには`K` (ヘルプ) や パラグラフ移動などファイラではまず使わないキーか、`Ctrl` や `<Leader>` あたりを活用する必要が出てきます。  
ちなみに現時点での自分のマッピングを抜粋するとこんな感じです:

- 親ディレクトリに移動 - `Ctrl-h`
- 子ディレクトリに移動 or ファイルオープン - `Ctrl-l`
- ツリーの開閉 - `f`
- リフレッシュ - `L`

`f` は本当は潰したくないのですが、ツリーの開閉は頻繁に行うため `Ctrl` や `Shift` が必要になるのも辛く、一旦こうしています。

このようなキーマッピングの不便さがあるため、将来的には閲覧モードと編集モードを分けて、別々にキーマッピングを登録できるようにする方が良いかもしれないと考えています。
そうすれば閲覧モードではもっとリッチな表示を行う (例えばファイルの種類ごとにアイコンを表示したり)、といった機能追加の可能性も開けます。

## 使ってみた感想

最低限の機能は実装できたのでここ数日はドッグフーディングしてみています。
感想としては期待通りで、いつものテキスト操作でファイルの移動や削除がサクサクできるのは中々に心地よいです。
ファイラの表示内容については、編集機能と両立させやすいように何の飾り付けもないため、他のファイラプラグインに比べると質素な形式になっているものの、実用上は特に問題ないと感じます。
現段階でも自分がファイラに求めたものはほぼ実現できており、割と満足しています。

とはいえ本当は Git と連携して untracked なファイルをハイライトしたり、 LSP と連携してエラーのあるファイルをハイライトしたりできても便利だなとは思うので、そういう拡張が可能になるとより良いなとも思います。

## 実装について

あとは実装に関する話を何点か、備忘録がてらまとめておきます。
主にはファイラ上でのテキスト編集を実際のファイル操作にマッピングするにあたり考えた事などです。

### 行ごとのメタデータは conceal で持つ

ファイラ上の各行はそれぞれがファイルシステム上のパスに対応しており、この対応づけを基にしてファイラでの編集を実際のファイル操作にマッピングします。
例えば `~/foo` をファイラで開いて以下のようになる時、

```js
src/
  index.js
package.json
```

2 行目を削除して保存すると、実際に `~/foo/src/index.js` というファイルが削除されます。
これは 2 行目に「この行は `~/foo/src/index.js` に対応している」というメタデータを埋め込む事で実現しています。
他の行も同様のメタデータを持っています。
行を移動した場合も同じようにメタデータを参照することで、移動元のパスを知る事が出来るわけです。

[vim-conceal]: https://vimhelp.org/syntax.txt.html#conceal
[vim-text-props]: https://vimhelp.org/textprop.txt.html

このメタデータの管理方法にはいくつかの候補がありましたが、現時点では Vim の [conceal][vim-conceal] という機能を使っています。
conceal は Vim のシンタックスハイライト機能の一部で、バッファ上の一部のテキストを非表示にできます。
 Viler ではこれを使って各行の末尾に見えないメタデータをこっそり置いています。
ただ conceal で隠れるテキストは単に見えないだけで普通に編集できてしまうため、活用するには面倒な側面があります。
例えば conceal 内のテキストを誤って消してしまうとメタデータが失われ、意図しないファイル操作につながります。
これが嫌だったため、当初は[テキストプロパティ][vim-text-props]という Vim 8 から導入された機能で実装する事を考えました。
これを使うと任意の位置のテキストにメタデータを付与する事ができ、 conceal のように余計な文字列をバッファに置く必要もありません。
そのためテキストプロパティを使って行ごとに一意な ID を持たせ、対応するメタデータを ID に紐づけて裏側で保持すれば良いと考えました。

```js
// Add ID for each line using text property.
[1] src/
[2]   index.js

// Manage metadata in Vim script (in memory).
let metadata = {
  \ '1': { 'path': '~/foo/src', 'is_dir': 1 },
  \ '2': { 'path': '~/foo/src/index.js', 'is_dir': 0 },
  \ }
```

一見良さそうなものの、今回のユースケースでは以下の点が問題になりました:

- テキストプロパティを付与されたテキストをコピーもしくは移動しても、ペースト先のテキストにテキストプロパティは引き継がれない。
- undo のハンドリングがしづらい。

いつも通りに編集できる操作感を目指す以上、行のコピーや移動がそのままファイルのコピー・移動となるようにしたいです。
しかしテキストプロパティを持つ行を移動しても、そのプロパティは移動先の行には引き継がれません。
これでは移動元のファイル情報がわからなくなり困ります。
となると行のコピー・移動時にプロパティを引き継ぐ何らかの工夫が必要になりますが、多様な編集方法を持つ Vim において、あらゆる方法でのコピー・移動を正しくカスタマイズするのは良いやり方ではなさそうに思えました。

次に undo への対処も問題です。
編集可能なファイラである以上、 undo/redo もいつも通り動いてほしいです。
しかし前述したようなメタデータの実態をバッファの外に持つ実装だと、 undo で状態を正しく戻す事が難しくなります。
例えば行ごとのメタデータには、パスの他にも「ツリーが開いているかどうか」という状態があります (無論ディレクトリのみ)。
以下なら `src/` は開いている状態であり、それゆえ内部の `index.js` ファイルが表示されています。

```js
src/
  index.js
```

このツリーの開閉状態は、例えばそのディレクトリの表示形式を変えるのに必要です。
単に `src/` という文字列だけだと、このディレクトリが空だった場合開いているのかどうかわからず不便なので、視覚的に判別できるようにしています (今は開いてると下線がつきます。本当は一般的なファイラ同様ファイル名の左にアイコンを置きたいのですが、編集機能と相性が悪いため一旦こうしています) 。
この表示形式の決定のためにツリーの開閉状態をメタデータに持たせたいわけです。
しかし例えばファイラで一度ツリーを開いた後、すぐに undo するとどうなるでしょう？
バッファ上の表示は undo によりツリーが閉じた状態に戻りますが、バッファ外で管理されるメタデータの実態は変わらないため、こちらでは依然としてツリーが開いている事になっています。
このように、メタデータ (状態) をバッファ外に持つ方法だと簡単に不整合な状態に陥ってしまいます。
これに対処するには undo 単位で状態をロールバックできるような、複雑な仕組みが必要になると思います。

そこで改めて conceal を使う方法を考えてみると、これらの問題がすべて片付く事に気づきます。
まずメタデータはテキストとして各行に埋め込まれているので、行単位でコピー・移動する限りメタデータも常に一緒です。
また undo に関しても、メタデータの ID だけをバッファに持つのではなく、状態に関してはまるごとバッファに埋め込んでしまえば良いです。
先程のツリー開閉の例でいうと、ツリーを開くタイミングでバッファ上のメタデータ内の開閉フラグも 1 (open) に書き換えます。
すると undo されたらツリーが閉じるだけでなく、開閉フラグも元の 0 (closed) に戻ってくれるわけです。
このようにステートフルなメタデータは直接バッファ上に持たせ、それを一次ソースにしてしまえば、特別な仕組みを用意しなくても undo/redo に対応できます。
こういった利点から、今回は conceal を使う実装方法を選びました。  
とはいえ、 conceal にも問題はあります。それは先にも触れた通り、見えないだけで存在するので普通に編集できてしまう点です。特に以下のようなケースだと、誤って編集してしまった事にも気づきにくいです:

- `C` や `d$` などを使ってメタデータを含む行末まで削除してしまうケース。
  - ファイルをリネームする目的でこれをやってしまうと、該当のファイルは削除されてしまいます。今は使う側に気をつけてもらうしかないのですが、 `C` などのよく使うキーは上書きできるよう、ファイル名だけ削除するキーマッピングを用意しようかと考えています。
- 置換処理で conceal 内の文字列まで変わってしまうケース。
  - 現状だと conceal 内には普通に数字を置いているので、ファイル名を一括置換する目的で `:%s/\d\+/` などとされるとメタデータの内容まで変わってしまいアウトです。これは普通に起きうるので、 conceal 内で使う文字列は絵文字など、なるべくファイル名に使われにくい文字に変更する予定です。

### 変更の適用では順序に気をつける

次はファイラ上の編集内容を実際のファイルシステムに適用する手順についてです。
Viler を使ったファイル操作が通常のファイル操作と大きく違う点は、複数の変更を (見かけ上は) 一度に適用する点です。
ファイル `a` を削除し、 `b` を `b2` にリネームし、新たに `c` を追加、といった一連の変更をした上で、ファイラを保存すると初めてそれらの変更がまとめて適用されます。
とはいえ実際に複数の変更をアトミックに適用する手段があるわけではないため、保存時にはファイラの変更内容から必要なファイル操作を決め、それらを順に適用していく事になります。
今の例なら以下のようになるでしょう:

```bash
# 1. a を削除する。
rm a
# 2. b をリネームする。
mv b b2
# 3. c を追加する。
touch c
```

しかし、変更内容を単純なコマンド実行にマッピングするだけだと問題のあるケースがあります。
わかりやすいのは、ファイル名をスワップする場合です。
ファイル `a` と `b` の名前を入れ替えたい場合、ファイラ上のテキスト編集としては単にそれらの名前を変更するだけになります。
ではこれらの変更を素直にコマンドに変換するとどうなるでしょう？

```bash
# 1. a を b にリネームする。
mv a b
# 2. b を a にリネームする。
mv b a
```

もちろんこれは上手くいきません。最初の `mv` で `b` を上書きしてしまうので、結果的には `b` が削除されてしまいます。
もし Bash などでファイル名を入れ替えるなら、どちらかを一時ファイルに退避する必要があるでしょう。

```bash
mv a _a
mv b a
mv _a b
```

複数の変更をまとめて適用する際には、こういったファイル名 (パス) の重複が起きない仕組みが必要になります。
スワップ以外にも、ファイル `a` を削除した上で新たに `a` という同名のファイルを作ったり、別のファイルを `a` にコピーするような操作もありえます。
そのようなケースにも対応できるよう、 Viler では必要なファイル操作を決定後、以下の手順で変更を適用します。

1. 削除対象のファイルを全て削除する。
1. 移動対象の全ファイルを一時ディレクトリに移動する。
1. 移動対象の全ファイルを一時ディレクトリから実際の移動先に移す。
1. 新規ファイルを追加する。

つまり本来は 1 ステップである移動処理を、一時ディレクトリへの退避と移動という 2 ステップに分けて行います。
こうする事で、操作の途中でパスが重複する事態を回避しています (ちなみに実装のしやすさゆえコピーも移動とほぼ同じ要領でやります。この場合 2 の手順が移動ではなくコピーになります)。
なお、最終的なファイル構成にパスの重複がないかは変更適用を始める前にチェックし、重複があればエラーとして変更適用を中止します。

### 未保存ディレクトリのコンテンツ編集は一旦なし

前述した方法でシンプルなファイル操作には対応できますが、実際にはまだまだ考慮すべきケースがあります。

- ディレクトリを削除しつつ、その中の一部ファイルだけ別のパスに移動できる？
- ディレクトリを移動して、移動先のディレクトリ内に別のファイルをコピーしてこれる？
- ディレクトリをコピーして、コピー元とコピー先のそれぞれのコンテンツを別々に編集できる？
- ディレクトリのコンテンツを変更しつつ、そのディレクトリを別ファイラでリネームできる？

これらは要するに、変更を保存していないディレクトリの中身を編集できるか、という問題です。
それも複数のファイラで別々に編集されうる事を考慮に入れる必要があります 
(複数ファイラを開く場合、未保存の変更内容を全ファイラで共有できる方が良い気もするのですが、ちょっと面倒そうなので現在はそれぞれ個別に編集でき、保存時に変更内容をマージして適用するという方針にしています)。  
当初はファイルツリーを木構造で表してごにょごにょやれば出来るんじゃないかと思ってたのですが、いざ取り掛かるとカバーできないケースが色々見えてきて中々終わりが見えませんでした。
あまり時間がかかると飽きてしまうのが何より怖かったので、まずは形にする事を優先し、初期リリースでは以下のどちらかを諦める事にしました。

- 保存していないディレクトリの中身を編集できる機能
- 複数のファイラをまたがってファイルを移動・コピーできる機能
  (= 複数ファイラの変更をマージして適用できる機能)

2 つ目の機能がないと離れたパスのファイルを移動する際に不便なので、なくても使えはする 1 つ目の方を切り捨てる事にしました。  
ちなみに、保存してないディレクトリ内の編集をサポートするには具体的にどんな難しさがあったのか、もう忘れてしまいました...。さっさと動かすところまでいきたくてメモを怠ったのは良くなかったです。

### テストは themis.vim で

[themis.vim]: https://github.com/thinca/vim-themis

この先の機能修正やリファクタリングがしやすいよう、最低限のテスト環境は構築しました。
テストには [themis.vim][themis.vim] というプラグインを使っています。
themis.vim を使うと普通の Vim script でテストを記述できるし、プラグイン自体も Vim script だけで書かれているので、 Vim とこのプラグインさえあればテストを実施でき、とても便利です。   
まだカバー出来てない範囲も多いですが、例えば以下のようなテストを書いています:

- ファイラバッファ上で何らかの操作をした結果、バッファの表示内容が期待通りになるか？ (移動やツリーの開閉など)
- ファイラから抽出した変更内容を基に、期待するファイル操作を導出・適用できるか？

後者のテストは、テスト中に実際にファイルを作ってそれらに導出された変更を適用し、適用後のファイルツリーが意図する結果になるかをテストします。
実際のファイル操作はせずスタブ化する手もありますが、テストの実行速度を気にする段階ではないし、より信頼できるテストにするためそうしました。

またテストをするなら CI も欲しいので、 [Vim と themis.vim が入った Docker イメージ][docker-vim-ci-themis] を用意して使っています。
最初は CircleCI で動かしてましたが、開発中に [GitHub Actions][github-actions] が正式リリースされたため、途中でこちらに乗り換えました。
便利すぎてビビりました。
CircleCI 同様 Docker ベースで環境を作れる上、多様なサードパーティ製 Action を使ったり作ったり出来てしまうとは。安いし。

[docker-vim-ci-themis]: https://hub.docker.com/repository/docker/ryym/vim-ci-themis
[github-actions]: https://help.github.com/en/actions

## まとめ

Vim 用の編集可能なファイラプラグイン Viler について紹介しました。
まだだいぶ粗いですが、自分が欲しかった形にはかなり近づけられました。  
最近はエディタというと Visual Studio Code の勢いが凄まじく実際めちゃ良いですが、これほど自由にカスタマイズできる Vim もまた最高だなと改めて感じました。
