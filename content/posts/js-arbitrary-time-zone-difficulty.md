+++
date = "2021-05-06T23:21:00+09:00"
description = ""
tags = ["javascript"]
title = "JS の Date で任意のタイムゾーンを表現するのは難しい"
updated = "2021-05-06T23:21:00+09:00"
+++

## 一言で

JS ビルトインの `Date` はタイムゾーンが環境依存で不便。上手いこと任意のタイムゾーンを設定可能にできると良いが、 DST (サマータイム) が絡むと厳密にやるのは難しかった。

## JS `Date` の使いづらい点

JavaScript の `Date` は以下の理由で使いづらい:

1. 不変じゃない。 `setTime` などでインスタンスの日時を変更できる。
2. 文字列パース (`new Date(string)`, `Date.parse`) の挙動が実装依存 ([MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/Date#timestamp_string))。
3. タイムゾーンを指定できない。常にローカル (ホストシステム) のタイムゾーンになる。

1,2 は `Date` の使用方法を制限したり、 `Date` のラッパーを使ったりすれば対処はできる。  
3 はローカルのタイムゾーンを固定できると安心だが、ブラウザのようなユーザ環境ではそうもいかない。
複数のタイムゾーンを扱いたい時や、常に特定のタイムゾーンで日時を表現したい時などに困りうる。
そこで試しに任意のタイムゾーンを指定できる仕組みを作れないかと思ったが、 `Date` ベースの実装だと厳密にやるのは難しかった。
どんな問題があるのかをここにメモしとく。

## `Date` の挙動おさらい

`Date` のインスタンスは特定の UTC 日時を表し、 `getDate`, `getHours` などの各種メソッドはローカルのタイムゾーンを考慮した値を返す。
UTC における日時を取得したい時は `getUTCDate`, `getUTCHours` 系のメソッドが使える。

```js
process.env.TZ = "Asia/Tokyo"; // +09:00

// コンストラクタで渡す値はローカルの日時として解釈される (month は 0 始まり)。
const d = new Date(2020, 3, 1, 10, 0, 0);

console.log(d.toISOString()); //=> 2020-04-01T01:00:00.000Z
console.log(d.getUTCHours()); //=> 1
console.log(d.getHours()); //=> 10

process.env.TZ = "Asia/Dubai"; // +04:00

// タイムゾーンが変わっても d が保持する UTC 日時は変わらない。
console.log(d.toISOString()); //=> 2020-04-01T01:00:00.000Z
console.log(d.getUTCHours()); //=> 1

// getHours 系メソッドは新しいタイムゾーンに応じた値を返すようになる。
console.log(d.getHours()); //=> 5
```

(なお `process.env.TZ` で `Date` のタイムゾーンを変更できるのは Node.js v14 以降:
<https://zenn.dev/dora1998/articles/node-process-env-tz>)

## 見かけの日時を調整して別のタイムゾーンを表現してみる

このような挙動である `Date` を使い、任意のタイムゾーンにおける日時を表現できるか？  
`Date` の [`getTimezoneOffset`][date-gettimezoneoffset] メソッドは、そのインスタンスが使う UTC へのオフセットを分単位で返す。
`Asia/Tokyo` なら -540 になる。
これを使えば、せめて任意のオフセットを指定しての日時処理は出来るはず。
ローカルのオフセットと望むオフセットとの差分を取る事で、 `getHours` などが返す値を調整すれば良い。

[date-gettimezoneoffset]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/getTimezoneOffset

オフセットを調整する実装例:

```js
// UTC からのオフセットと Date を受け取り、「指定されたオフセットにおける
// originalDate の日時」を保持してるっぽく見える Date を返す。
const withUTCOffset = (desiredOffset, originalDate) => {
  const utcTime = originalDate.getTime();
  const date = new Date(utcTime);
  const offsetDiff = desiredOffset + date.getTimezoneOffset();
  date.setTime(utcTime + offsetDiff * 1000 * 60);
  return date;
};
```

(ちなみに、 `getTimezoneOffset` が常にローカルのタイムゾーンに基づくオフセットを返すなら、別にインスタンスメソッドである必要はなさそうに見えるが、 DST のあるタイムゾーンではオフセットが時期により変わる。つまりインスタンスが表す日時によって同じタイムゾーンでもオフセットは異なりうるため、インスタンスメソッドになっている)

### 上手くいく例

```js
const tokyoOffset = 540; // +09:00

const okPattern = () => {
  process.env.TZ = "Asia/Dubai"; // +04:00

  const d1 = new Date(2021, 3, 1, 10, 0, 0);
  console.log(d1); //=> 2021-04-01T06:00:00.000Z
  console.log(d1.getHours()); //=> 10

  const d2 = withUTCOffset(tokyoOffset, d1);
  console.log(d2); //=> 2021-04-01T11:00:00.000Z

  // hours == 15 (UTC 06:00 == Dubai 10:00 == Tokyo 15:00)
  console.log(d2.getHours()); //=> 15
};
okPattern();
```

d2 が指す実際の UTC 日時はデタラメだが、 `getHours()` などのメソッドは「`d1` が `Asia/Dubai` で指していた日時の `Asia/Tokyo` における値」を返すように調整されている。
この方法を使えば、ローカルのタイムゾーンに関わらずタイムゾーン (オフセット) を指定可能な日時クラスを作れそうに見える。
見かけ上の日時だけでなく実際の UTC 日時も必要なら、 `d1.getTime()` が返すミリ秒を合わせて保持すれば良い ([`getTime`][date-gettime] はタイムゾーン関係なく Unix 時刻からの経過時間を返す)。

[date-gettime]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/getTime

実際これは割と上手くいくが、ローカルのタイムゾーンが DST を持つ場合、日時を正しく調整できないケースが存在する。

### DST により上手くいかない例

例えば `America/Chicago` には DST があり、2021 年は 3月14日 02:00 から DST が始まる。つまり 01:59 の次は 03:00 となる。
そのため `America/Chicago` においては 2021-03-14 02:00 ~ 02:59 という日時が存在しない。

```js
process.env.TZ = "America/Chicago";
const d1 = new Date(2021, 2, 14, 2, 0, 0);
console.log(d1.getHours()); //=> 3
```

JS の Date は存在しない月日や時刻を指定してもエラーにならず、超過分を繰り越す。
例えば `new Date(2000, 15, 33)` は 2001-05-03 となる。
なので 02:00 を指定している上記の例もエラーにはならず、 03:00 となる。

こうなると、先程示した `withUTCOffset` には問題があるとわかる。
ローカルのタイムゾーンが `America/Chicago` の状態で `Asia/Tokyo` における 2021-03-14 02:00 を表現しようとしても、 `getHours` が 3 を返すため期待する日時と 1 時間ズレてしまう。

```js
const impossiblePattern = () => {
  process.env.TZ = "America/Chicago"; // -06:00, -05:00 in DST

  const d1 = new Date(2021, 2, 13, 11, 0, 0);
  console.log(d1); //=> 2021-03-13T17:00:00.000Z
  console.log(d1.getHours()); //=> 11

  const d2 = withUTCOffset(tokyoOffset, d1);
  console.log(d2); //=> 2021-03-14T08:00:00.000Z

  // hours should be 2 but 3
  // (Chicago 11:00 == UTC 17:00 == Tokyo 02:00 (next day))
  console.log(d2.getHours()); //=> 3
};
impossiblePattern();
```

このように `withUTCOffset` 相当の実装は、 DST が絡むと正しく動かないケースがある。

`getHours` 系メソッドはローカルのタイムゾーンを自動で考慮するためこういう問題が起きるが、
UTC における日時を返す `getUTCHours` 系メソッドなら特定の時刻を返せない問題はない。ので UTC 日時の方を調整する手もあるが、結局は同じ問題にぶつかる。先程の例とは逆に `America/Chicago` のタイムゾーンにおける日時を作りたい場合、 2021-03-14 02:00 ~ 02:59 は存在してはいけないのに、 `getUTCHours` 系を使うと 02:00 という時刻を返せてしまう。

というわけで、 JS の `Date` を使って任意のタイムゾーンにおける日時を表現するのは難しい。  
この問題は DST へ切り替わる日時において発生する。逆に言うとそれ以外の日時では (たぶん) 期待通りに動くので、 `withUTCOffset` 相当の実装でも問題ないユースケースはあるかも。どこまで厳密さを求めるかによる。

### Date を使うライブラリは同じ問題を持つ

タイムゾーンを考慮した日時計算を提供するライブラリはいくつかある。
試したところ、`Date` をベースにしたライブラリはやはり同様の問題を抱えていた。
全く同じ処理でも、ローカルのタイムゾーン次第で結果が変わってしまう。

[`dayjs`](https://day.js.org/) (v1.10.4):

```js
const dayjs = require("dayjs");
const utc = require("dayjs/plugin/utc");
const timezone = require("dayjs/plugin/timezone");
dayjs.extend(utc);
dayjs.extend(timezone);

const d = new Date(Date.UTC(2021, 2, 13, 17, 0, 0));

process.env.TZ = "Asia/Dubai";
console.log(dayjs(d).tz("Asia/Tokyo").format("YYYY-MM-DD HH:mm:ss"));
//=> 2021-03-14 02:00:00

process.env.TZ = "America/Chicago";
console.log(dayjs(d).tz("Asia/Tokyo").format("YYYY-MM-DD HH:mm:ss"));
//=> 2021-03-14 03:00:00
```

[`date-fns`](https://date-fns.org/) (v2.21.1):

```js
const fns = require("date-fns");
const tz = require("date-fns-tz");

const d1 = new Date(Date.UTC(2021, 2, 13, 17, 0, 0));

process.env.TZ = "Asia/Dubai";
const d2 = tz.utcToZonedTime(d1, "Asia/Tokyo");
console.log(fns.format(d2, "yyyy-MM-dd HH:mm:ss"));
//=> 2021-03-14 02:00:00

process.env.TZ = "America/Chicago";
const d3 = tz.utcToZonedTime(d1, "Asia/Tokyo");
console.log(fns.format(d3, "yyyy-MM-dd HH:mm:ss"));
//=> 2021-03-14 03:00:00
```

一方で、 JS の `Date` を使わずに独自の `DateTime` クラスを提供する [`Luxon`][luxon] にはこの問題がなかった (v1.26.0):

[luxon]: https://moment.github.io/luxon/index.html

```js
const { DateTime } = require("luxon");

process.env.TZ = "America/Chicago";
const d1 = DateTime.local(2021, 2, 13, 11, 0, 0);
console.log(d1.toISO());
//=> 2021-02-13T11:00:00.000-06:00
//(= 2021-02-13T17:00:00.000Z)

const d2 = d1.setZone("Asia/Tokyo");
console.log(d2.toISO());
//=> 2021-02-14T02:00:00.000+09:00
```

が、bundle size が 70kB とブラウザで気軽に使うには少し重め。  
https://bundlephobia.com/result?p=luxon@1.26.0

## ざっくり方針

というわけで、厳密に複数のタイムゾーンを考慮したいなら Luxon を使う必要がありそう。

しかし単に特定のタイムゾーンだけ扱うなら `Date` で何とかなるかも。
例えば日本在住者のみが使う想定のサービスなら、ブラウザでもローカルのタイムゾーンは `Asia/Tokyo` である前提で割り切っちゃうとか。
あるいはあくまでローカルのタイムゾーンは不定という前提なら、 `Date` を UTC 日時ベースでのみ使いつつオフセットを時前で管理するか。DST があると難しそうだけど。  

ただそもそも、ローカルとは違うタイムゾーンを使いたいケース自体それほど多くない気はする。
単なる日付の表示・入力であれば、むしろローカルのタイムゾーンに則る方が大抵は UI として自然なはず。
その場合はせいぜい、日時の作成や比較時にローカル日時と UTC 日時を混同しないよう注意するくらいか。
