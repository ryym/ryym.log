+++
date = "2017-06-23T08:01:54+09:00"
description = "Vimのプラグイン管理をNeoBundleからminpacに移行した"
draft = false
tags = ["vim"]
title = "Vim: minpac+αで快適プラグイン管理"
updated = "2017-06-23T08:01:54+09:00"

+++

Vimのプラグイン管理にはずっと[NeoBundle][shougo-neobundle]を使っていたが、
しばらく前からアクティブな開発がストップしていたため、Vimのプラグイン管理方法を再構成する事にした。  
最初はNeoBundleの後続である[dein.vim][shougo-dein]を使うつもりだったが、
Vimバージョン8.0向けの[minpac][takata-minpac]というライブラリの方がシンプルで好みだったため、こちらを使う事に決めた。
ただ (2017-02時点の) minpacには最低限の機能しかなかったので、minpacをラップして足りない機能を補うようにした。
minpacはそのシンプルさゆえ、カスタマイズもしやすかった。

[shougo-neobundle]: https://github.com/Shougo/neobundle.vim
[shougo-dein]: https://github.com/Shougo/dein.vim
[takata-minpac]: https://github.com/k-takata/minpac

## minpacとは

そもそもNeoBundleのようなパッケージマネージャがユーザの手で作られていたのは、
Vimにビルトインのプラグイン管理機能がなかったため。
しかし 8.0 でようやく[packages][vimdoc-packages]という機能が追加された。
これがあれば、特定のディレクトリに使いたいプラグインを置いておくだけで簡単にそれらを使えるようになるものの、
プラグイン自体のダウンロードは以前として手動で行う必要がある。
自身の`~/.vim`ディレクトリをGitで管理しているなら、Gitの[submodule][git-submodule]を使ってプラグインを管理する手もあるが、
自分はなるべくVim内で管理を完結させたいし、他のモダンエディタのように必要に応じてダウンロードするようにしたい。

minpacはまさにそれをやってくれるライブラリで、以下のように使いたいプラグインを宣言すれば、
プラグインを自動でダウンロードしてくれる (詳細は[README][takata-minpac]参照)。
つまりminpacを使えば、プラグインのインストールは自動化しつつ、そのロードにはVimのビルトイン機能を使う事ができる。

```vim
call minpac#init()
call minpac#add('k-takata/minpac', {'type': 'opt'})
call minpac#add('Shougo/unite.vim')
call minpac#add('mattn/emmet-vim')
" ...

" Install or update
call minpac#update()
```

[git-submodule]: https://git-scm.com/docs/git-submodule
[vimdoc-packages]: http://vim-jp.org/vimdoc-en/repeat.html#packages

## minpacにないフック機能は自作する

しかし、minpacにはプラグインのロード前後に処理をフックする機能がない
(プラグインのインストール・アップデート後に処理をフックする機能はある)。
自分は以下の用途でNeoBundleのフック機能を使っていたため、
今後もフック機能は使い続けたかった。

- ロード前フック: プラグインの挙動を決める設定変数を定義する
- ロード後フック: プラグイン内で定義された関数などを使う

そこで、フックに関しては後述する方法で自作する事にした。

ただ、実際にはロード後フックが本当に必要なケースは少なく、ロード前フックだけで足りる場合が多いと思う。
そういう意味では、全プラグインのロード前処理をどこかにまとめて記述するだけでもいいかもしれないが、
プラグインごとに処理を個別定義できるならその方が管理はしやすい。フックがあればそれが可能になる。

### dein.vimではダメなのか?

しかしフックを自作するくらいならdein.vimを使えばいい話かもしれない。
確かにdein.vimにもフック機能は備わっているものの、ちょっとした不満があった。

- `dein#config`関数へのオプションとして渡す方法
    - 渡す関数を一度定義し、その参照を渡す必要があるのが面倒  
      (ただその面倒さは無名関数がないVim script自体の問題)
- TOMLファイルでプラグインリストを管理し、フック処理もそこに記述する方法
    - Vim scriptをTOML内に書くのがイヤ
    - 処理は`autoload`内に関数として記述し、TOML内ではそれを呼び出すコードだけを書く手もあるが、
      フック処理の記述だけ場所が離れてしまう

それに、やはり自分が欲しい機能は限られているから、あまり多機能なものを使うより、
最低限の機能の上に欲しいものだけ追加していく方法を採りたかった。

### 自作フック機能の仕組み

というわけで、以下のようにフックを定義できる仕組みを作った。

```vim
" Declare and configure foldCC.vim
let def = my#pack#add('LeafCage/foldCC.vim')
function! def.before_load()
  let g:foldCCtext_head = "printf('%s %d: ', repeat(v:folddashes, v:foldlevel), v:foldlevel)"
  let g:foldCCtext_tail = "printf('%d lines ', v:foldend - v:foldstart + 1)"
  set foldtext =FoldCCtext()
endfunction

" Declare and configure vim-cursorword
let def = my#pack#add('itchyny/vim-cursorword')
function! def.after_load()
  augroup vim-cursorword
    autocmd!
    autocmd FileType vimfiler,unite let b:cursorword = 0
  augroup END
endfunction

" And so on...
```

`my#pack#add`は`minpac#add`をラップした関数で、
`minpac#add`と同じように使いたいプラグインを宣言する。
`minpac#add`と違うのは、`my#pack#add`がDictionaryを返す点。
呼び出す側は、必要に応じてそのDictionaryに`before_load`と`after_load`という辞書関数を定義できる。
`my#pack#add`内ではこのDictionaryの参照を渡されたプラグイン名とセットで保持しておき、
プラグインをロードする前後でそれらの関数を呼ぶようにすれば、簡易的なフック機能となる。
呼び出し側が戻り値を変更するルールは設計としてあまり綺麗ではないが、
個人の用途だし簡単に書ける事を優先した。

### この方法の問題点

- フック関数をDictionaryに持っとく必要があり、ちょっとメモリの無駄遣い感がある。
- [作者さんの紹介記事][minpac-qiita]にあるように、minpacは本来プラグインをインストールする時以外は必要ない。
  しかしロード前後のフック機能と統合している現状では、毎回minpacをロードする形になる。

が、今のところ特に問題なくVimを使えているので、ひとまずこの方法で落ち着いている。

ちなみに、minpac本体に同様の機能を取り込んで欲しいという思いは特にない。
今のようにインストールの自動化に特化している方がわかりやすいし、必要な人だけ自作すれば良い気がする。
それにライブラリ側でフック機能を提供する場合は、前述のように関数の参照を受け取るよりクリーンな方法になるはず。
VimのLambdaがもっと高機能になれば、わざわざ別個に関数を定義せずに済むので一番いいけど...。

## まとめ

こんな感じで、プラグインのインストールなど面倒な部分はminpacに任せつつ、
ちょっと足りない部分は自分で拡張するようにしてみた。
高機能なパッケージマネージャに比べてminpacはブラックボックス感も少ないし、
プラグインロード時にエラーが起きても原因調査がしやすい。
おかげで、自分が求めていたプラグイン管理の仕組みを構築できた。

- シンプルな仕組みで動作を予測しやすい
- 使用するプラグインの宣言とその設定の記述を1箇所で行える
- ロード前後に行う処理を指定できる
- コードがVim scriptで完結する

ちなみに、minpacをラップしたコードの全貌は[こんな感じ][vim-mypack]。

自分が使っているプラグインの数は60ほどだが、もっとたくさんのプラグインを使う人は、
キャッシュやロードタイミングの細かな制御ができるdein.vimを使う方がいいかもしれない。
しかしバージョン8.0以降のVimを使っていて、シンプルな仕組みの方が好みな人には、minpacは良い選択肢だと思う。

[minpac-qiita]: http://qiita.com/k-takata/items/36c240a23f88d699ce86
[vim-mypack]: https://github.com/ryym/dotfiles/blob/5f7bd796049f72daea80f99d63824a59e9d024be/dotfiles/vim/autoload/my/pack.vim
