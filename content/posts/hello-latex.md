+++
date = "2016-04-26T23:29:00+09:00"
description = "LaTexを使ってノートを取りたかったので、OS Xにインストールしてみた。"
draft = false
tags = ["latex"]
title = "OS Xに LaTex を導入する手順"
updated = "2016-04-26T23:29:00+09:00"

+++

* https://texwiki.texjp.org/?LaTeX入門#d4959206
* http://tug.org/mactex/
* http://konn-san.com/prog/why-not-latexmk.html
* http://skim-app.sourceforge.net/

## インストール

### pkgファイルのダウンロード

https://texwiki.texjp.org/?MacTeX

いくつか種類があるようだが、とりあえず`MacTex`というのをインストールしてみる。
ダウンロードにかなり時間がかかる。

### アップグレード

```sh
sudo tlmgr update --self --all
```

これもだいぶ時間がかかった。
TexWikiのサイトだと他にも日本語の設定についていくつかの手順が記されているが、それらは飛ばしても問題なかった。

## Hello World

hello.tex:

```
\document{jaarticle}
\begin{document}

Hello, world!
% This is a comment.
\end{docuemnt}
```

```sh
# Compile and generate PDF
platex hello.tex
dvipdfmx hello.dvi

# Generate PDF By one command
ptex2pdf -l hello.tex
```

`latexmk`コマンドを使うと色々簡単にできてすごい!

http://konn-san.com/prog/why-not-latexmk.html

準備:

* 上記サイトを参考にして`~/.latexmkrc`を作成する
* [Skim](http://skim-app.sourceforge.net/)をインストールしておく
* Skimの[同期設定](http://skim-app.sourceforge.net/manual/SkimHelp_39.html)を有効にしておく

```sh
# latexmk is awesome!

# Generate PDF
latexmk hello.tex

# Watch and (re-)generate PDF when the file changed
latexmk -pvc hello.tex
```


