+++
date = "2016-03-21T00:00:00+09:00"
description = "package.jsonの依存関係定義に漏れがないかをチェックする、depcopというパッケージを作った。"
draft = false
tags = ["javascript"]
title = "package.jsonの依存関係をチェックするライブラリ"
updated = "2016-03-21T00:00:00+09:00"

+++

Node.jsを使うプロジェクトでは、そのプロジェクトで使用するライブラリを依存関係として
`package.json`の`dependencies`や`devDependencies`に定義します。
この依存関係定義に漏れがないかをチェックする、`depcop`というnpmモジュールを作りました。

https://github.com/ryym/node-depcop


package.json の例:

```javascript
{
  "name": "my-module",
  "version": "1.0.0",
  "dependencies": {
    "chalk": "^1.1.1",
    "espree": "^3.0.2",
    "glob": "^7.0.0"
  },
  "devDependencies": {
    "gulp": "^3.9.1",
    "gulp-babel": "^6.1.2"
  }
}
```

## 内容

このライブラリで調べられるようにしたかったのは、主に以下の3点です。

1. `dependencies`に定義されていないのに、コード内で使用されているモジュールがないか
2. `dependencies`に定義されているのに、コード内で使用されていないモジュールがないか
3. `devDependencies`に定義されているのに、公開されるコードの中で使用されているモジュールがないか

3の「公開されるコード」というのは、テストコードやビルドスクリプトを除いたメインのソースコードの事です。
大抵は`lib`や`src`ディレクトリの下に置かれていると思います。

## 動機

上記のうち、1はCIを回していればそこでテストが落ちるので簡単に気づけます(CI環境では普通、毎回依存関係をインストールするところから始まるため)。
2に関しては無駄な依存関係を持っているという点で問題はあるものの、ライブラリの機能自体には影響しないはずです。
厄介なのは3のパターンで、このケースだとローカル環境やCI環境でのテストは通ってしまいます。なぜなら、それらの環境では普通
`dependencies`と`devDependencies`の両方の依存関係がインストールされるからです。しかし、実際にそのライブラリが使われるタイミング、
つまり公開されたそのライブラリをユーザーがインストールするタイミング(`npm install my-module`)では`dependencies`の方しか一緒にインストールされないため、
そこで初めて必要な依存関係がインストールされずエラーになる、という事が起き得ます。

## 使い方

インストールする。

```sh
npm install --save-dev depcop
```

プロジェクトのルートディレクトリに`.depcoprc.js`という設定ファイルを作る。

```javascript
module.exports = {
  libSources: ['lib/**/*.js'],
  devSources: ['test/**/*.js', 'gulpfile.js'],
  checks: {
    missing: {},
    strayed: {}
    unused: {
      ignore: [/babel/, /mocha/, /istanbul/]
    }
  }
};
```

コマンドもしくはJavaScriptでチェックを実行する。

```sh
# CLI
$(npm bin)/depcop
```

```javascript
// JavaScript
import { makeDepcop } from 'depcop';

const depcop = makeDepcop();
const result = depcop.runValidations();
const format = depcop.getFormatter();
console.log(format(result));
```

設定ファイルの内容は全てコマンドラインでも指定できます。
詳しくは[リポジトリのドキュメント](https://github.com/ryym/node-depcop)に書いてあります。

## 類似ライブラリ

これを作った後で、[eslint-plugin-node](https://github.com/mysticatea/eslint-plugin-node)というライブラリの存在を知りました。
これはNode.js向けのルールを集めた[ESLint](http://hatenablog.com/g/11696248318754550877)プラグインで、その中の
`no-unpublished-import`ルールなどで`内容`に書いた1と3のチェックは実現できました。実は最初はこのライブラリも
ESLintのプラグインとして作ろうかという案もあったのですが、例えば「定義されているのに未使用のモジュールがないかをチェックする」というルールを実現するためには、複数のファイルを調べて使用されているモジュールの情報を集める必要があります。ESLintでは1つのファイル内で完結する
ルールしか定義できないので、別のライブラリとして作る事にしました。結果的には丸かぶりせずに済んで良かったです。
