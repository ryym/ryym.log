+++
date = "2022-12-24T10:36:00+09:00"
tags = ["keyboard", "linux"]
title = "Linux (GNOME) で Mac ライクにキー操作"
updated = "2022-12-24T10:36:00+09:00"
+++


## 一言で

[Xremap][xremap] を使い、 GNOME 環境で Mac に近いキーバインドを実現してみた。  
<https://github.com/ryym/dotfiles/blob/91c72ac1b5f42b3a89b899bce83fcccdedad83a5/dotfiles/arch/xremap.yml>

## Mac ユーザーからみた他 OS の壁

もう何年も仕事・プライベートともに Mac をメイン PC として使っているが、各種 Linux や Windows など他 OS の PC (デスクトップ環境) もちゃんと使えるようになりたいという思いが常々ある。
しかし過去何度か思い立ってセットアップしてみても、あまり本質的でない部分が障壁になって長続きしない。
その障壁というのがキーバインドの違いで、 Linux / Windows 系と Mac では各種ショートカットが異なる点にどうしてもストレスを感じる。

**Mac**

- `Super` (= `Cmd` , `Win` ) がメインのショートカット修飾キーである
    - `Super-c/v/x` でコピペ、 `Super-f` で検索など
- いわゆる Emacs 風キー操作が OS レベルでサポートされている
    - `Ctrl-f/b/n/p` でカーソル移動など
- (JIS キーボードなら) 英字・かなの切り替えがステートレスにできる

**Linux / Windows 系**

- `Ctrl` がメインのショートカット修飾キーである
    - `Ctrl-c/v/x` でコピペ、 `Ctrl-f` で検索など
- Emacs 風キー操作はターミナルなど一部のアプリでのみサポートされている
- 英字・かなの切り替えがステートフルになる

Mac は継続して使いつつ、並行して Linux / Windows も触って学びたい。
しかし OS によって基本的なキー操作を切り替えるのがめんどい。
そこでこのキーバインド問題を迂回できないか試してみたところ、ひとまず Linux (GNOME) に関してはほぼストレスなく操作できる形にたどり着けたので、ここに記録しておく。 

## 目指すもの

理想は OS によらず同じキー操作で作業できること。
今回は現状 Mac ユーザーである自分のために、他 OS で Mac のキーバインドを出来るだけ再現することを目指してみる。
ここでは Linux 、中でも手元にある Arch Linux の GNOME 環境 (version 43.2) のみを対象にする。

## GNOME 環境で Mac 風キー操作を実現する

### キーマッピングソフトウェア - Xremap

まずはキーマッピングをカスタマイズするソフトウェアを選ぶ。
キーリマッパーはどの OS でも便利なツールがありそうだし、あまり躊躇せず依存していく。
今回は [Xremap][xremap] を使った。

[xremap]: https://github.com/k0kubun/xremap

良さげな点:

- 比較的新しくてメンテナンスもアクティブそう
- 特定のアプリケーションでだけリマップを有効・無効にできる
- Rust 製なので (自分にとっては) いざという時コードを読んだりデバッグしたりしやすい

以降は Xremap を使って実施した各種カスタマイズをまとめる。

### `Super` キーをメインの修飾キーにする

いくつか方法はありえるが、以下のように `Super-c/v/x/...` を個別に `Ctrl-c/v/x/...` にマッピングする形で対応した:


```yaml
# Super をメイン修飾キーにする例
keymap:
  - remap:
      Super-A: Ctrl-A
      Super-C: Ctrl-C
      Super-E: Ctrl-E
      Super-F: Ctrl-F
      Super-K: Ctrl-K
      Super-L: Ctrl-L
      Super-N: Ctrl-N
      # ... etc
```

これで Mac と同じ操作感でコピペなどができる...と思いきや、 GNOME だとこれだけでは上手くいかなかった。
Xremap で設定していてもなお `Super` を押すと Activities overview が表示されてしまい、期待通りに動かない。

調べたところ、この挙動は GNOME の設定を変えれば無効化できた。これで上記のキーリマップが動くようになる。

```bash
% gsettings set org.gnome.mutter overlay-key '' # デフォルトは 'Super_L'
```

参考: <https://ict4g.net/adolfo/notes/admin/gnome-tips-and-tricks.html>

#### 他の検討案 - `Super` を `Ctrl` にマッピングする

他に検討した方法として `Super` 自体を `Ctrl` にマッピングする案もあったが、これだと `Super-Tab` によるアプリのスイッチング (Windows でいう `Alt-Tab` ) が難しくなるので止めた。

- 単純に `Ctrl-Tab` を `Super-Tab` にマッピングするだけだと、 `Tab` から指を離した途端 `Super-Tab` によるスイッチングだと認識されなくなり、 `Super` ( `Ctrl` ) を押しつつ `Tab` をポチポチして切り替えていくあの動作が再現できない。
- GNOME 側で `Switch Applications` のショートカットを `Ctrl-Tab` に変更する手はあるが、 `Ctrl-Tab` はブラウザなど複数のアプリでタブ切り替えのショートカットに使われており、なるべく奪いたくない。


### Emacs 風キー操作を常に使用可能にする

これもいくつかやり方はあるが、 `Ctrl-f` などのショートカットを対応するカーソル操作へ愚直にマッピングする方法を採った。

```yaml
# Emacs 風キー操作を再現する例
keymap:
  - remap:
      Ctrl-B: Left
      Ctrl-F: Right
      Ctrl-P: Up
      Ctrl-N: Down
      Ctrl-A: Home
      Ctrl-E: End
      Ctrl-K: [Shift-End, Delete]
      # ... etc
```

Xremap は複数キーへのリマップもサポートしてるので、 `Ctrl-k` (カーソル以降の文字列削除) のように少し複雑なアクションも難なく再現できる。

ただしこのマッピングをグローバルに適用してしまうと以下 2 つのケースで問題があり、対策が必要だった。

1. ターミナル上の操作
2. ブラウザ上の Vim 風キー操作

#### ターミナル上の操作を邪魔しない

他のアプリと違いターミナルだけはデフォルトで Emacs 風のキー操作をサポートしており、特にリマップは必要ない。
逆に `Home` キーなどは効かないので、上記マッピングを適用するとターミナルでの操作はむしろ壊れてしまう。
これは Xremap のアプリごとにマッピングを変えられる機能を使って解決できる。

```yaml
# ターミナル以外のアプリにのみリマップを適用する例
keymap:
  - application:
      not: gnome-terminal-server
    remap:
      Ctrl-B: Left
      # ...
```

こうすると、ターミナルでも他のアプリでも同じ感覚で Emacs 風キー操作が使える。

#### ブラウザ上の Vim 風キー操作を調整する

自分は Vim が好きなので、ブラウザ上の各種操作を Vim 風に行えるエクステンションを常用しており、 Firefox では [Tridactyl][tridactyl] を使っている。
この手のエクステンションを使うと `j`, `k` で上下にスクロール、 `Ctrl-f` , `Ctrl-b` で上下にページ単位のスクロール、などが出来るようになる。
しかし Xremap で `Ctrl-f/b` を矢印キー `Right`, `Left` に割り当ててしまうと、ページ単位のスクロールが出来なくなる。
一方でブラウザ内のテキスト入力では Emacs 風キー操作を使いたいので、ターミナルのように Xremap のリマップを丸ごと無効にもできない。

仕方がないのでこれについては Tridactyl 側で追加の設定を行い、リマップしていない時と同じように動くようにした。

[tridactyl]: https://github.com/tridactyl/tridactyl/


```vim
" .tridactylrc
js -r .tridactylrc.js
```

```js
// .tridactylrc.js
// (see `:help js` for details)
tri.browserBg.runtime.getPlatformInfo().then(os => {
  if (os.os === 'linux') {
    tri.excmds.bind('<ArrowRight>', 'scrollpage 1') // <C-f>
    tri.excmds.bind('<ArrowLeft>', 'scrollpage -1') // <C-b>
  }
});
```

このように矢印左右を再度ページ単位のスクロールに割り当てる事で、元々と同じ挙動を実現している。

```shell
Keyboard -[Ctrl-f]-> Xremap -[Right]-> Tridactyl -[scrollpage 1]-> Firefox
```

これで Emacs 風キー操作を保ちつつ、ブラウザの Vim 風キー操作も動くようになった。

ちなみに Xremap だとマッピング後の入力が更にリマップされはしないので、以下を併用しても問題ない。

```yaml
Super-f: Ctrl-f
Ctrl-f: Right
```

`Super-f` -> `Ctrl-f` -> `Right` のように連続して変換はされず、 `Super-f` はあくまで `Ctrl-f` として認識される。そのためキーボード上の `Ctrl-f` (= `Right` 入力) とは分けて使える。



#### 他の検討案 - GNOME が提供する Emacs モードを使う

GNOME には Emacs 風のキーバインド設定が用意されており、以下で有効にできる。
これがあれば Xremap によるカスタマイズは不要かと思われたが、結局使わなかった。

```bash
% gsettings set org.gnome.desktop.interface gtk-key-theme Emacs
```

というのもどうも期待する挙動にならない。この設定よりも Web アプリのショートカット設定の方が優先されてしまい、使いづらい。
例えば Slack や Discord では `Ctrl-k` でチャンネルのスイッチャーを開けるので、ブラウザでこれらを開いてる際に `Ctrl-k` を入力した時の期待動作は以下となる。

- テキスト入力時 ... Emacs 風キー操作 (カーソル以降の文字列を削除する)
- それ以外 ... アプリのショートカット (チャンネルのスイッチャーを開く)

しかし `gtk-key-theme Emacs` だとそうはならず、テキスト入力時でもチャンネルのスイッチャーが開いてしまう。
これでは正直使い物にならない。

一方で Xremap を使えばテキスト入力時は Emacs 風キー操作になるうえ、 `Super-k` 押下で (Xremap により `Ctrl-k` に変換され) スイッチャーが開くという、 Mac と同じ挙動を実現できる。

### 英字・ひらがなをステートレスに切り替える

Arch Linux での日本語入力には現状 [Fcitx5][fcitx5] と [Mozc][mozc] を使っている。
インプットメソッドを順番に切り替えるキーは自由に設定できるものの、 Mac の JIS キーボードにある `英数` , `かな` キーのように、特定のキーを特定の入力ソースに紐付ける方法は (たぶん) 無い。
しかし `fcitx5-remote` というコマンドラインツールと Xremap を組み合わせれば、 Mac の `英数` , `かな` と同等の操作感を再現できた。

[fcitx5]: https://wiki.archlinux.org/title/Fcitx5
[mozc]: https://wiki.archlinux.org/title/Mozc

```yaml
# 無変換・変換キーを Mac の英数・かな相当にする例
keymap:
  - remap:
      # 無変換を押したら常に英字へ
      Muhenkan:
        launch: ["fcitx5-remote", "-s", "keyboard-us"]
      # 変換を押したら常にひらがなへ
      Henkan:
        launch: ["fcitx5-remote", "-s", "mozc"]
```

このようにキー入力をコマンド実行に割り当てる事で、ステートレスに英字・ひらがなを切り替え可能になった。


## 所感と残課題

以上の設定により、ほぼ Mac と同じ感覚で Arch Linux でもキー入力できるようになる。
間違って `Ctrl-a` で全選択したり `Super-p` で印刷画面を開いたりすることがなくなり、 Mac ユーザーには快適な状態になった。

ただし完全に同じでは当然なく、特にマウスクリックとキー入力を組み合わせるような操作は Xremap では対応できない。
例えばブラウザのリンクをマウスクリックにより別タブで開きたい場合、 Mac では `Super` を押しつつクリックだが Linux / Windows では `Ctrl` を押しつつクリックとなる。このようなキーボードのみで完結しない操作も Mac と同じにしようとすると、現状の方法では難しい。
けれど個人的にはこの程度の差異なら許容範囲なので、特に対応は考えていない。

## Windows でやるなら？

似たようなカスタマイズをすれば Windows でも同様の操作感を実現できるかもしれないが、手元に Windows PC がないため未確認。
Windows だと [AutoHotkey][autohotkey] というリッチなキーリマッパーがあるので、これで頑張れば何とかなるかもしれない。

[autohotkey]: https://www.autohotkey.com/

## その他 - Xremap 導入でハマった点

最後に Linux 初心者が Xremap を正しく使えるまでにハマった点を記録しておく。

### キーボードの接続に応じて Xremap を起動する

Xremap はキーボードが未接続の状態で起動しても上手く動かないので、キーボードが接続されたタイミングで Xremap を起動する必要がある。
これは以下の方法で行った。

1. [udev][udev] rule を使い、キーボードの接続に応じて systemd device unit を作る。
2. Xremap を systemd service として立ち上げ、キーボードの device unit に応じて start/stop する。

[udev]: https://wiki.archlinux.org/title/udev

udev というものの存在を今回初めて知り、正しく rule を記述できるまでに苦労したが、勉強になった。

- [xremap-udev.rules](https://github.com/ryym/dotfiles/blob/91c72ac1b5f42b3a89b899bce83fcccdedad83a5/dotfiles/arch/xremap-udev.rules)
- [xremap.service](https://github.com/ryym/dotfiles/blob/91c72ac1b5f42b3a89b899bce83fcccdedad83a5/dotfiles/arch/xremap.service)


### GNOME Shell Extension と通信する

Xremap のコア機能は単体の CLI として使えるが、 GNOME で特定のアプリが forground の時のみリマップを有効・無効にするには、別途用意されている [GNOME Shell Extension][xremap-gnome-shell-ext] と併用する必要がある。
しかし shell extension をインストールして有効にしても最初は上手くいかなかった。
この時に Xremap が Rust 製である事が役に立ち、ローカルに clone して動かしてデバッグして、という調査が簡単にできた。
調べてみると原因は単純で、 Xremap 本体を root 権限で動かしていたせいだった。
Xremap と shell extension は [D-Bus][dbus] という仕組みで通信しており、 Xremap は root 権限で、 shell extension はログインユーザ権限で動いていたため、正しく通信できていない状態だった。
[Xremap の README][xremap-readme] に `sudo` なしで動かす手順が書いてあるので、それを参考に Xremap もログインユーザ権限で動かすようにしたところ、期待通りに動くようになった。

試しに `busctl` などで Xremap の shell extension に直接問い合わせると、今ウィンドウがアクティブなアプリの情報を確認でき、正しく通信できているとわかる。

```bash
% busctl --user call org.gnome.Shell /com/k0kubun/Xremap com.k0kubun.Xremap ActiveWindow
s "{\"wm_class\":\"gnome-terminal-server\",\"title\":\"Terminal\"}"
```

[xremap-gnome-shell-ext]: https://extensions.gnome.org/extension/5060/xremap/
[dbus]: https://www.freedesktop.org/wiki/Software/dbus/
[xremap-readme]: https://github.com/k0kubun/xremap/tree/49b3f11b3c9575a0cf9ae0dd2635d73d16c6bfc3#usage

### 未解決の問題 - `/dev/uinput` の permission

`sudo` なしで Xremap を動かすために udev rule で `/dev/uinput` の permission を設定しているのだが、稀にこれが上手くいかない。
udev rule 自体はちゃんと動いてて `dev/uinput` は `root` ではなく `uinput` group の持ち物になるし、ログインユーザは `uinput` group に所属しているのだが、なぜか `/dev/uinput` へのアクセスが permission denied になる事がたまーにある。
再起動で直ったり直らなかったりして、いまいち原因がわかっていない。
`ls` してみると `/dev/uinput` には `+` マークがついてて、こいつは ACL (access control list) という追加のアクセス制限を設定するものらしい。
なのでその辺に原因があるのかもと思いつつ、稀にしか発生しないので原因を調査できていない。

