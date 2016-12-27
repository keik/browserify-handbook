# はじめに

このドキュメントは[Browserify](http://browserify.org)を使ってモジュール化されたアプリケーションを作る方法を説明します。

[![cc-by-3.0](http://i.creativecommons.org/l/by/3.0/80x15.png)](http://creativecommons.org/licenses/by/3.0/)

Browserifyは[Nodeのモジュールシステム](http://nodejs.org/docs/latest/api/modules.html)であるCommon JSモジュールをブラウザ向けにコンパイルするツールです。

Nodeでは、Node向けパッケージのホスティングサービスであると同時にそれらのパッケージマネージャであるコマンドラインツール[npm](https://npmjs.org)が使用できます。
npmを使ってインストールしたモジュールを使ったソースコードを書き、それをBrowserifyでビルドすることで、ブラウザ上で動作するコードが生成できるのです。ビルド処理とnpmの使用以外にはNodeの機能を使う必要はありません。

npmに公開されているモジュールのうち、Nodeとブラウザのどちらの環境でも動作するものはますます増えています。中にはブラウザ上でのみ動作するモジュールすら存在します。今や[npmはバックエンドJavaScriptのためのものだけでなくフロントエンドのためのものでもある](http://maxogden.com/node-packaged-modules.html)のです。


# 目次

- [はじめに](#はじめに)
- [目次](#目次)
- [ドキュメントのインストール](#ドキュメントのインストール)
- [Nodeのモジュールシステム](#Nodeのモジュールシステム)
  - [require](#require)
  - [exports](#exports)
  - [ブラウザ用にバンドルする](#ブラウザ用にバンドルする)
  - [Browserifyの動作原理](#Browserifyの動作原理)
  - [how node_modules works](#how-node_modules-works)
  - [why concatenate](#why-concatenate)
- [development](#development)
  - [source maps](#source-maps)
    - [exorcist](#exorcist)
  - [auto-recompile](#auto-recompile)
    - [watchify](#watchify)
    - [beefy](#beefy)
    - [wzrd](#wzrd)
    - [browserify-middleware, enchilada](#browserify-middleware-enchilada)
    - [livereactload](#livereactload)
    - [browserify-hmr](#browserify-hmr)
    - [budo](#budo)
  - [using the api directly](#using-the-api-directly)
  - [grunt](#grunt)
  - [gulp](#gulp)
- [builtins](#builtins)
  - [Buffer](#Buffer)
  - [process](#process)
  - [global](#global)
  - [__filename](#__filename)
  - [__dirname](#__dirname)
- [transforms](#transforms)
  - [writing your own](#writing-your-own)
- [package.json](#package.json)
  - [browser field](#browser-field)
  - [browserify.transform field](#browserifytransform-field)
- [finding good modules](#finding-good-modules)
  - [module philosophy](#module-philosophy)
- [organizing modules](#organizing-modules)
  - [../../../../../../..と書かずに済ます方法](#../../../../../../..と書かずに済ます方法)
  - [non-javascript assets](#non-javascript-assets)
  - [reusable components](#reusable-components)
- [testing in node and the browser](#testing-in-node-and-the-browser)
  - [testing libraries](#testing-libraries)
  - [code coverage](#code-coverage)
  - [testling-ci](#testling-ci)
- [bundling](#bundling)
  - [saving bytes](#saving-bytes)
  - [standalone](#standalone)
  - [external bundles](#external-bundles)
  - [ignoring and excluding](#ignoring-and-excluding)
  - [browserify cdn](#browserify-cdn)
- [shimming](#shimming)
  - [browserify-shim](#browserify-shim)
- [partitioning](#partitioning)
  - [factor-bundle](#factor-bundle)
  - [partition-bundle](#partition-bundle)
- [compiler pipeline](#compiler-pipeline)
  - [build your own browserify](#build-your-own-browserify)
  - [labeled phases](#labeled-phases)
    - [deps](#deps)
      - [insert-module-globals](#insert-module-globals)
    - [json](#json)
    - [unbom](#unbom)
    - [syntax](#syntax)
    - [sort](#sort)
    - [dedupe](#dedupe)
    - [label](#label)
    - [emit-deps](#emit-deps)
    - [debug](#debug)
    - [pack](#pack)
    - [wrap](#wrap)
  - [browser-unpack](#browser-unpack)
- [plugins](#plugins)
  - [using plugins](#using-plugins)
  - [authoring plugins](#authoring-plugins)

# ドキュメントのインストール

このドキュメントはnpmでインストールできます。

```
npm install -g browserify-handbook
```

インストール後は、 `browserify-handbook` コマンドでこのドキュメントを既定の `$PAGER` で開けます。

# Node モジュール

Browserifyの使い方や動作の仕組みの前に、まずは[Nodeが採用しているCommon JSモジュールシステム](http://nodejs.org/docs/latest/api/modules.html)について理解することが重要です。

## require

Nodeには他のファイルからコードを読み込むための `require` 関数があります。

[npm](https://npmjs.org)を使って次のように `uniq` モジュールをインストールした場合、

```
npm install uniq
```

`nums.js` ファイルを作成してそこに `require('uniq')` と書くことで、 `uniq` モジュールを読み込むことができます。

```
var uniq = require('uniq');
var nums = [ 5, 2, 1, 3, 2, 5, 4, 2, 0, 1 ];
console.log(uniq(nums));
```

このプログラムをNodeで実行すると、次の出力が得られます。

```
$ node nums.js
[ 0, 1, 2, 3, 4, 5 ]
```

相対パス上のファイルを読み込むためには、`require` 関数のパラメータを `.` から始めます。
`main.js` ファイルから `foo.js` ファイルを読み込む場合は、次のようにします。

``` js
var foo = require('./foo.js');
console.log(foo(4));
```

`foo.js` が親ディレクトリにある場合には、 `../foo.js` と書きます。

``` js
var foo = require('../foo.js');
console.log(foo(4));
```

相対パスは呼び出し元のファイルの場所から解決されます。

上の例において、`require('uniq')` は戻り値として関数を返し、それを `uniq` という変数に代入していますが、代入する変数名には他の名前をつけることもできます。 `require()` は、単に指定したモジュール内でエクスポートされた値を返すだけだからです。

他のプラットフォームのモジュールシステムだと、インポートされたモジュール内で定義された変数がグローバルスコープやファイルローカルスコープで使用可能になるといったような動作が多いですが、Nodeの `require` 関数はこうした動作とは違っています。
この仕組みによって書かれたプログラムであれば、ある機能がどこから読み込まれたものかを簡単に知ることができるため、アプリケーションの規模が増え大量のモジュールが読み込まれるようになっても破綻しにくいのです。

## exports

ファイル内のコードを他のファイルからインポート可能にするためには、エクスポートする値を `module.exports` に代入します。

``` js
module.exports = function (n) {
    return n * 111
};
```

上で作成した `foo.js` を他のモジュール `main.js` から読み込む場合、 `require('./foo.js')` と書くことでエクスポートされた関数が得られます。

``` js
var foo = require('./foo.js');
console.log(foo(5));
```

このプログラムの出力結果は次のようになります。

```
555
```

また次の例のように、関数以外のどんな値でも `module.exports` に代入してエクスポートできます。


``` js
module.exports = 555
```

``` js
var numbers = [];
for (var i = 0; i < 100; i++) numbers.push(i);

module.exports = numbers;
```

オブジェクトにエクスポートする別の方法として、次の例のように `module.exports` の代わりに `exports` のプロパティに代入することもできます。

``` js
exports.beep = function (n) { return n * 1000 }
exports.boop = 555
```

次のようにしても同様です。

``` js
module.exports.beep = function (n) { return n * 1000 }
module.exports.boop = 555
```

なぜなら `module.exports` と `exports` は、デフォルトで同じ空のオブジェクトを参照しているからです。

注意点として、次のようにはできません。

``` js
// this doesn't work
exports = function (n) { return n * 1000 }
```

最終的にエクスポートされる値は `module` オブジェクト上にある必要があるため、 `exports` 自体に新しい値を代入すると `module.exports` への参照が失われてしまうためです。

よって一つの値をエクスポートしたい場合には、次のようにします。

``` js
// instead
module.exports = function (n) { return n * 1000 }
```

どういうことかよくわからなければ、次のような仕組みで動作していると理解するよう努めてください。

``` js
var module = {
  exports: {}
};

// If you require a module, it's basically wrapped in a function
(function(module, exports) {
  exports = function (n) { return n * 1000 };
}(module, module.exports))

console.log(module.exports); // it's still an empty object :(
```

大抵の場合、モジュールは一つのことを行うように設計されているほうが便利なので、`module.exports` を使って一つの関数またはコンストラクタをエクスポートするといいです。

Nodeの歴史上、はじめのころのエクスポートの主な手段は `exports` でしたが、後に `module.exports` が使えるようになってからは、こちらのほうが便利で直感的で無駄がないと思われるようになりました。

どういうことかというと、昔は次のような書き方が一般的でした。

foo.js:

``` js
exports.foo = function (n) { return n * 111 }
```

main.js:

``` js
var foo = require('./foo.js');
console.log(foo.foo(5));
```

しかし `foo.foo` というコードは少し冗長です。 `module.exports` を使えばもっときれいになります。

foo.js:

``` js
module.exports = function (n) { return n * 111 }
```

main.js:

``` js
var foo = require('./foo.js');
console.log(foo(5));
```

## ブラウザ用にバンドルする

Nodeでは、 `node` コマンドにファイルを渡して実行します。

```
$ node robot.js
beep boop
```

Browserifyでも同じようにコマンドにファイルを渡しますが、その結果はファイルを実行するのではなく、ビルドされたJavaScriptを標準出力に出力します。 `>` 演算子を使えばファイルに出力できます。

```
$ browserify robot.js > bundle.js
```

上のコマンドの結果、 `robot.js` の動作に必要なすべてのコードが `bundle.js` に書き込まれます。
そうしたらこのファイルをHTML内から `script` タグを使って読み込むだけです。

``` html
<html>
  <body>
    <script src="bundle.js"></script>
  </body>
</html>
```

TIPS: `script` タグを `</ body>` の直前に書くことで、DOMの `onready` イベントを待たなくてもページ上のすべてのDOM要素を使うことができます。

バンドル処理ではもっとたくさんのことができます。詳細は[バンドル](#バンドル)セクションを参照してください。

## Browserifyの動作原理

Browserifyは、エントリポイントのファイルを[抽象構文木](https://en.wikipedia.org/wiki/Abstract_syntax_tree)を用いて静的解析することで `require` の呼び出し箇所を[探し出します](http://npmjs.org/package/detective)。

`require` が文字列の引数とともに呼び出されている箇所ごとに、そのモジュールのファイルパスを解決し、さらにそのファイル内での `require` 呼び出しを依存グラフ全体に対して再帰的に検索します。

そうして解決された各ファイルは一つのJavaScriptファイルとして結合されます。結合後のファイルには独自の小さな `require` 関数の実装が含まれており、それは各ファイルに内部的に割り当てられたIDに基いて依存性を解決するように動作します。

このようにして生成されるバンドルには、動作に必要なコードが全て含まれ、動作のオーバーヘッドもごくわずかなものとなるのです。

より詳細な動作については[コンパイラパイプライン](#コンパイラパイプライン)を参照してください。

## how node_modules works

Nodeには、他のプラットフォームにはない賢いモジュールの解決アルゴリズムを持っています。

Nodeでは、コマンドラインツールのように `$PATH` などのシステムのパス上からパッケージを検索するのではなく、ローカルから探し出します。

`/beep/boop/bar.js` ファイル内で `require('./foo.js')` というコードが呼び出されると、Node は `./foo.js` の場所を `/beep/boop/foo.js` と解決して探し出します。
`./` または `../` から始まるパスは、`require()` を呼び出したファイルからの相対パスになります。

`require('xyz')` のように相対パスではないパスで呼び出された場合は、モジュールが見つかるまで次の順で探し出し、見つからなかった場合はエラーを発生させます。

```
/beep/boop/node_modules/xyz
/beep/node_modules/xyz
/node_modules/xyz
```

`xyz` ディレクトリが存在する場合は、 `xyz/package.json` ファイル内の `"main"` フィールドを見に行きます。 `"main"` フィールドが定義されている場合は、そこで指定されたファイルが `require()` の結果として返されます。

例えば最初に `/beep/node_modules/xyz` が見つかり、さらに `/beep/node_modules/xyz/package.json` ファイルが次のように定義されていた場合、

```
{
  "name": "xyz",
  "version": "1.2.3",
  "main": "lib/abc.js"
}
```

`require('xyz')` の結果として `/beep/node_modules/xyz/lib/abc.js` のエクスポートした値が得られます。

もし `package.json` や `"main"` フィールドの定義が存在しない場合は、 `index.js` ファイルを探し出します。

```
/beep/node_modules/xyz/index.js
```

パッケージ内の特定のファイルを指定して読み込むこともできます。 `dat` パッケージ内にある `lib/clone.js` ファイルを読み込む場合は次のようにします。

```
var clone = require('dat/lib/clone.js')
```

この場合、 `node_modules` 内を探して最初に見つかった `dat` パッケージのディレクトリを起点とし、 `lib/clone.js` を解決します。

Nodeにはパス上から探し出す機能もありますが非推奨です。よほどの理由がない限りは `node_modules` 内から解決する方法を使うべきです。

Nodeのモジュール解決アルゴリズムとnpmによるパッケージインストールの仕組みは、他のプラットフォームで起こりがちなバージョンの競合を起こすことがありません。npmはパッケージの依存モジュールを `node_modules` 内にインストールします。さらにそれぞれのライブラリは、依存モジュールを自分の `node_modules` に持つという再帰的な構造を持ちます。

つまり同じアプリケーション内においても異なるバージョンのライブラリを正しく使い分けることができるため、バージョン間でのAPIの互換性問題に対してとても容易に対処できるのです。この機能は、中央集権的ではなく自治的にパッケージ管理を行うnpmのようなエコシステムにおいてはかかせません。パッケージの作者は、自分のパッケージにとって正しいバージョンの依存ライブラリを使うだけで、他のライブラリの依存関係を気にする必要はないからです。

パッケージ化されていないアプリケーション内の独自モジュールを、 `node_moduels` の仕組みを使って整理することもできます。[`../../../../../../..と書かずに済ます方法`](../../../../../../..と書かずに済ます方法)も見てみてください。

## ファイルを結合する目的

Browserifyによるビルドはサーバ側で行われます。その結果、動作に必要なすべてのソースコードを含むひとかたまりのファイルを生成します。

他にもブラウザ向けにモジュールシステムを実装する方法はありますが、それぞれに長所・短所があります。

### グローバルなwindow

モジュールシステムを使わない代わりに、各ファイルがグローバルの `window` オブジェクトにプロパティや独自の名前空間を定義するという方法があります。

この方法では、読み込むスクリプトを増やすたびすべてのHTMLに対して `<script>` タグを追記していく必要があるので、アプリケーションをスケールしていくのが面倒です。さらにグローバルに特定のプロパティが定義済みであることを前提としたスクリプトがあると、ファイルの読み込み順序も考慮しなくてはならなくなります。

この方法はリファクタリングもメンテナンスも難しくなりがちです。メリットを挙げるとすれば、すべてのブラウザで動作することと、サーバで何のツールも使わずに済むことです。

その他、各 `<script>` タグごとにHTTPリクエストを行うのでロードが非常に遅くなりがちです。

## ファイル結合

グローバルな `window` を使わない代わりに、すべてのスクリプトファイルを結合する方法です。この方法でもやはりファイルを結合する順番を考慮する必要があるため、メンテナンス性はよくありません。ただし `<script>` タグが一つで済むのでロードは早くなります。

またsource mapを活用しないと、例外の発生箇所を結合前のファイル内に対応付けるのが面倒です。

### AMD

`<script>` タグを使うかわりに、すべてのファイルを `define()` 関数とコールバックでラップする方法で、[AMD](http://requirejs.org/docs/whyamd.html) と呼ばれています。

第一引数にはモジュール名の配列を渡します。これらのモジュールがロードされたのち、コールバック関数がそれらのモジュールを引数に受け取った形で呼び出されます。

``` js
define(['jquery'] , function ($) {
    return function () {};
});
```

第一引数には、他のモジュールからこのモジュールを呼び出すための名前を定義することもできます。

コールバック関数内ではCommonJS形式の `require()` 関数が使えるようになっています。これは[正規表現によるソースコードの変換](https://github.com/jrburke/requirejs/blob/master/require.js#L17)がなされているためです。

この方法ではモジュール間の依存性が明示的に記述できるので、グローバル名前空間やファイル結合を用いた方法のようにファイル読み込み順序を気にしなくても済みます。

パフォーマンスを向上するため、通常はAMD形式のコードはサーバ上で一つのファイルにビルドされて配布されます。一方、開発中はビルドせずにAMDの非同期ロード機能を使うことが多いです。

### bundling commonjs server-side

効率よくビルドするためにAMDをやめてCommon JSを使ってみませんか？ ツールを使えばスクリプトの依存性を正しく解決できるし、開発時の環境をほぼそのままにプロダクション環境に反映できます。Commmon JSのシンタックスは優れている上に、Nodeとnpmを使用できるのでエコシステムの活用や発展にも役立ちます。

ソースコードをビルドするステップし、それに関連してSource Mapsや自動再ビルドの
Node環境とブラウザ環境でコードを共有できます。
You can seamlessly share code between node and the browser. You just need a
build step and some tooling for source maps and auto-rebuilding.

Plus, we can use node's module lookup algorithms to save us from version
mismatch insanity so that we can have multiple conflicting versions of different
required packages in the same application and everything will still work. To
save bytes down the wire you can dedupe, which is covered elsewhere in this
document.

# development

Concatenation has some downsides, but these can be very adequately addressed
with development tooling.

## source maps

Browserify supports a `--debug`/`-d` flag and `opts.debug` parameter to enable
source maps. Source maps tell the browser to convert line and column offsets for
exceptions thrown in the bundle file back into the offsets and filenames of the
original sources.

The source maps include all the original file contents inline so that you can
simply put the bundle file on a web server and not need to ensure that all the
original source contents are accessible from the web server with paths set up
correctly.

### exorcist

The downside of inlining all the source files into the inline source map is that
the bundle is twice as large. This is fine for debugging locally but not
practical for shipping source maps to production. However, you can use
[exorcist](https://npmjs.org/package/exorcist) to pull the inline source map out
into a separate `bundle.map.js` file:

``` sh
browserify main.js --debug | exorcist bundle.js.map > bundle.js
```

## 自動再ビルド

再ビルドするたびにコマンドを再実行するのは非効率的なので、ツールで自動化しましょう。幸いにも自動化に使えるツールはたくさんあります。中にはビルド以外にも再読み込みなどを自動化する機能を持つものもあります。

実際に使うこととなるツールの数は少ないかもしれませんが、npmには実にたくさんのツールがあります！ そうしたツールの機能性の違いは、開発スタイルにも影響を与えるものです。たくさんのツールの中からあなたの期待や感覚にマッチするものを見つけ出すのは大変かもしれませんが、それらのツールの多様性に触れることこそが、プログラマとしてのスキルとセンスを磨くのに役立つはずです。Browserify自体が他の有力なライブラリを取り入れれば、もっとシンプルな使い方を提供することもできるでしょう。しかしそれよりも、Browserifyのコア機能を小さく保つことでツールの使い方に多様性を持たせることのほうが、長い目で見たときに健全と言えると思います。

できるだけ少ないツールでビルドプロセスを構築したいと思うかもしれませんが、ここで紹介していないツールについてもぜひ調べてみてください！

### [watchify](https://npmjs.org/package/watchify)

Browserifyはビルドプロセスを一回しか実行しませんが、watchifyを使えばビルド後も依存ツリー上のファイル変更を検知して自動再ビルドすることが可能となります。使い方は、 `browserify` コマンドを使うかわりに `watchify` コマンドを使うだけです。再ビルド時にキャッシュを使うので高速です。
`-v` オプションを使うと再ビルド時にログを出力します。

```
$ watchify browser.js -d -o static/bundle.js -v
610598 bytes written to static/bundle.js  0.23s
610606 bytes written to static/bundle.js  0.10s
610597 bytes written to static/bundle.js  0.14s
610606 bytes written to static/bundle.js  0.08s
610597 bytes written to static/bundle.js  0.08s
610597 bytes written to static/bundle.js  0.19s
```

設定例として、package.jsonの `"scripts"` フィールドに次のようにコマンドを定義してみましょう。

``` json
{
  "build": "browserify browser.js -o static/bundle.js",
  "watch": "watchify browser.js -o static/bundle.js --debug --verbose",
}
```

この設定の上で、プロダクション用のビルドには `npm run build` を、開発時の自動ビルドには `npm run watch` を使用すると効率的でしょう。

[`npm run` の詳細はこちら](http://substack.net/task_automation_with_npm_run)を見てください。

### [beefy](https://www.npmjs.org/package/beefy)

Webサーバ上で自動再ビルドをしたい場合、[beefy](http://didact.us/beefy/)が使えます。

使い方はbeefyコマンドにエントリファイルを渡すだけです。

```
beefy main.js
```

これでビルドしたファイルを配信するHTTPサーバが起動します。

### [wzrd](https://github.com/maxogden/wzrd)

beefyに似てますがよりシンプルにしたものが[wzrd](https://github.com/maxogden/wzrd)です。

`npm install -g wzrd` でインストールしたら次のように実行します。

```
wzrd app.js
```

そしたらウェブブラウザで http://localhost:9966 にアクセスしてください。

### browserify-middleware, enchilada

Expressを使っているなら[browserify-middleware](https://www.npmjs.org/package/browserify-middleware)や[enchilada](https://www.npmjs.org/package/enchilada)が使えます。

どちらもBrowserifyのバンドルを配信するためのミドルウェアです。

### [LiveReactLoad](https://github.com/milankinen/livereactload)

LiveReactLoadは[React](https://github.com/facebook/react)コンポーネントの変更を表示中のウェブページに自動反映するツールです。

使い方は[LiveReactLoadのREADME](https://github.com/milankinen/livereactload#livereactload)を参照してください。

### [browserify-hmr](https://github.com/AgentME/browserify-hmr)

browserify-hmrは Hot Module Replacement (HMR) を可能にするプラグインです。

HMRによる更新に対応しているファイルや、それの依存ファイルを更新すると、更新後のファイルが再実行されます。

例として、次のようなファイルがあるとします。

`main.js`:

``` js
document.body.textContent = require('./msg.js')

if (module.hot) module.hot.accept()
```

`msg.js`:

``` js
module.exports = 'hey'
```

次に `browserify-hmr` を使って `main.js` の変更を監視します。

```
$ watchify main.js -p browserify-hmr -o public/bundle.js -dv
```

さらに `public/` 内のファイルをホストするウェブサーバを起動します。

```
$ ecstatic public -p 8000
```

この状態で http://localhost:8000 にアクセスすると `hey` というメッセージが表示されるでしょう。

続けて `msg.js` を次のように変更します。

``` js
module.exports = 'wow'
```

変更してすぐに、画面には自動的に `wow` というメッセージが表示されるでしょう。

Browserify-HMR can be used with
[react-hot-transform](https://github.com/AgentME/react-hot-transform) to
automatically allow all React components to be updated live in addition to code
using the `module.hot` API. Unlike
[livereactload](https://github.com/milankinen/livereactload), only modified
files are re-executed instead of the whole bundle on each modification.

### [budo](https://github.com/mattdesl/budo)

budo is a browserify development server with a stronger focus on incremental bundling and LiveReload integration (including CSS injection).

Install it like so:

```sh
npm install budo -g
```

And run it on your entry file:

```
budo app.js
```

This starts the server at [http://localhost:9966](http://localhost:9966) with a default `index.html`, incrementally bundling your source on filesave. The requests are delayed until the bundle has finished, so you won't be served stale or empty bundles if you refresh the page mid-update.

To enable LiveReload and have the browser refresh on JS/HTML/CSS changes, you can run it like so:

```
budo app.js --live
```

## using the api directly

You can just use the API directly from an ordinary `http.createServer()` for
development too:

``` js
var browserify = require('browserify');
var http = require('http');

http.createServer(function (req, res) {
    if (req.url === '/bundle.js') {
        res.setHeader('content-type', 'application/javascript');
        var b = browserify(__dirname + '/main.js').bundle();
        b.on('error', console.error);
        b.pipe(res);
    }
    else res.writeHead(404, 'not found')
});
```

## grunt

If you use grunt, you'll probably want to use the
[grunt-browserify](https://www.npmjs.org/package/grunt-browserify) plugin.

## gulp

If you use gulp, you should use the browserify API directly.

Here is
[a guide for getting started](http://viget.com/extend/gulp-browserify-starter-faq)
with gulp and browserify.

Here is a guide on how to [make browserify builds fast with watchify using
gulp](https://github.com/gulpjs/gulp/blob/master/docs/recipes/fast-browserify-builds-with-watchify.md)
from the official gulp recipes.

# builtins

In order to make more npm modules originally written for node work in the
browser, browserify provides many browser-specific implementations of node core
libraries:

* [assert](https://npmjs.org/package/assert)
* [buffer](https://npmjs.org/package/buffer)
* [console](https://npmjs.org/package/console-browserify)
* [constants](https://npmjs.org/package/constants-browserify)
* [crypto](https://npmjs.org/package/crypto-browserify)
* [domain](https://npmjs.org/package/domain-browser)
* [events](https://npmjs.org/package/events)
* [http](https://npmjs.org/package/http-browserify)
* [https](https://npmjs.org/package/https-browserify)
* [os](https://npmjs.org/package/os-browserify)
* [path](https://npmjs.org/package/path-browserify)
* [punycode](https://npmjs.org/package/punycode)
* [querystring](https://npmjs.org/package/querystring)
* [stream](https://npmjs.org/package/stream-browserify)
* [string_decoder](https://npmjs.org/package/string_decoder)
* [timers](https://npmjs.org/package/timers-browserify)
* [tty](https://npmjs.org/package/tty-browserify)
* [url](https://npmjs.org/package/url)
* [util](https://npmjs.org/package/util)
* [vm](https://npmjs.org/package/vm-browserify)
* [zlib](https://npmjs.org/package/browserify-zlib)

events, stream, url, path, and querystring are particularly useful in a browser
environment.

Additionally, if browserify detects the use of `Buffer`, `process`, `global`,
`__filename`, or `__dirname`, it will include a browser-appropriate definition.

So even if a module does a lot of buffer and stream operations, it will probably
just work in the browser, so long as it doesn't do any server IO.

If you haven't done any node before, here are some examples of what each of
those globals can do. Note too that these globals are only actually defined when
you or some module you depend on uses them.

## [Buffer](http://nodejs.org/docs/latest/api/buffer.html)

In node all the file and network APIs deal with Buffer chunks. In browserify the
Buffer API is provided by [buffer](https://www.npmjs.org/package/buffer), which
uses augmented typed arrays in a very performant way with fallbacks for old
browsers.

Here's an example of using `Buffer` to convert a base64 string to hex:

```
var buf = Buffer('YmVlcCBib29w', 'base64');
var hex = buf.toString('hex');
console.log(hex);
```

This example will print:

```
6265657020626f6f70
```

## [process](http://nodejs.org/docs/latest/api/process.html#process_process)

In node, `process` is a special object that handles information and control for
the running process such as environment, signals, and standard IO streams.

Of particular consequence is the `process.nextTick()` implementation that
interfaces with the event loop.

In browserify the process implementation is handled by the
[process module](https://www.npmjs.org/package/process) which just provides
`process.nextTick()` and little else.

Here's what `process.nextTick()` does:

```
setTimeout(function () {
    console.log('third');
}, 0);

process.nextTick(function () {
    console.log('second');
});

console.log('first');
```

This script will output:

```
first
second
third
```

`process.nextTick(fn)` is like `setTimeout(fn, 0)`, but faster because
`setTimeout` is artificially slower in javascript engines for compatibility reasons.

## [global](http://nodejs.org/docs/latest/api/all.html#all_global)

In node, `global` is the top-level scope where global variables are attached
similar to how `window` works in the browser. In browserify, `global` is just an
alias for the `window` object.

## [__filename](http://nodejs.org/docs/latest/api/all.html#all_filename)

`__filename` is the path to the current file, which is different for each file.

To prevent disclosing system path information, this path is rooted at the
`opts.basedir` that you pass to `browserify()`, which defaults to the
[current working directory](https://en.wikipedia.org/wiki/Current_working_directory).

If we have a `main.js`:

``` js
var bar = require('./foo/bar.js');

console.log('here in main.js, __filename is:', __filename);
bar();
```

and a `foo/bar.js`:

``` js
module.exports = function () {
    console.log('here in foo/bar.js, __filename is:', __filename);
};
```

then running browserify starting at `main.js` gives this output:

```
$ browserify main.js | node
here in main.js, __filename is: /main.js
here in foo/bar.js, __filename is: /foo/bar.js
```

## [__dirname](http://nodejs.org/docs/latest/api/all.html#all_dirname)

`__dirname` is the directory of the current file. Like `__filename`, `__dirname`
is rooted at the `opts.basedir`.

Here's an example of how `__dirname` works:

main.js:

``` js
require('./x/y/z/abc.js');
console.log('in main.js __dirname=' + __dirname);
```

x/y/z/abc.js:

``` js
console.log('in abc.js, __dirname=' + __dirname);
```

output:

```
$ browserify main.js | node
in abc.js, __dirname=/x/y/z
in main.js __dirname=/
```

# transforms

Instead of browserify baking in support for everything, it supports a flexible
transform system that are used to convert source files in-place.

This way you can `require()` files written in coffee script or templates and
everything will be compiled down to javascript.

To use [coffeescript](http://coffeescript.org/) for example, you can use the
[coffeeify](https://www.npmjs.org/package/coffeeify) transform.
Make sure you've installed coffeeify first with `npm install coffeeify` then do:

```
$ browserify -t coffeeify main.coffee > bundle.js
```

or with the API you can do:

```
var b = browserify('main.coffee');
b.transform('coffeeify');
```

The best part is, if you have source maps enabled with `--debug` or
`opts.debug`, the bundle.js will map exceptions back into the original coffee
script source files. This is very handy for debugging with firebug or chrome
inspector.

## writing your own

Transforms implement a simple streaming interface. Here is a transform that
replaces `$CWD` with the `process.cwd()`:

``` js
var through = require('through2');

module.exports = function (file) {
    return through(function (buf, enc, next) {
        this.push(buf.toString('utf8').replace(/\$CWD/g, process.cwd()));
        next();
    });
};
```

The transform function fires for every `file` in the current package and returns
a transform stream that performs the conversion. The stream is written to and by
browserify with the original file contents and browserify reads from the stream
to obtain the new contents.

Simply save your transform to a file or make a package and then add it with
`-t ./your_transform.js`.

For more information about how streams work, check out the
[stream handbook](https://github.com/substack/stream-handbook).

# package.json

## browser field

You can define a `"browser"` field in the package.json of any package that will
tell browserify to override lookups for the main field and for individual
modules.

If you have a module with a main entry point of `main.js` for node but have a
browser-specific entry point at `browser.js`, you can do:

``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browser": "browser.js"
}
```

Now when somebody does `require('mypkg')` in node, they will get the exports
from `main.js`, but when they do `require('mypkg')` in a browser, they will get
the exports from `browser.js`.

Splitting up whether you are in the browser or not with a `"browser"` field in
this way is greatly preferrable to checking whether you are in a browser at
runtime because you may want to load different modules based on whether you are
in node or the browser. If the `require()` calls for both node and the browser
are in the same file, browserify's static analysis will include everything
whether you use those files or not.

You can do more with the "browser" field as an object instead of a string.

For example, if you only want to swap out a single file in `lib/` with a
browser-specific version, you could do:

``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browser": {
    "lib/foo.js": "lib/browser-foo.js"
  }
}
```

or if you want to swap out a module used locally in the package, you can do:

``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browser": {
    "fs": "level-fs-browser"
  }
}
```

You can ignore files (setting their contents to the empty object) by setting
their values in the browser field to `false`:

``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browser": {
    "winston": false
  }
}
```

The browser field *only* applies to the current package. Any mappings you put
will not propagate down to its dependencies or up to its dependents. This
isolation is designed to protect modules from each other so that when you
require a module you won't need to worry about any system-wide effects it might
have. Likewise, you shouldn't need to wory about how your local configuration
might adversely affect modules far away deep into your dependency graph.

## browserify.transform field

You can configure transforms to be automatically applied when a module is loaded
in a package's `browserify.transform` field. For example, we can automatically
apply the [brfs](https://npmjs.org/package/brfs) transform with this
package.json:

``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browserify": {
    "transform": [ "brfs" ]
  }
}
```

Now in our `main.js` we can do:

``` js
var fs = require('fs');
var src = fs.readFileSync(__dirname + '/foo.txt', 'utf8');

module.exports = function (x) { return src.replace(x, 'zzz') };
```

and the `fs.readFileSync()` call will be inlined by brfs without consumers of
the module having to know. You can apply as many transforms as you like in the
transform array and they will be applied in order.

Like the `"browser"` field, transforms configured in package.json will only
apply to the local package for the same reasons.

### configuring transforms

Sometimes a transform takes configuration options on the command line. To apply these
from package.json you can do the following.

**on the command line**
```
browserify -t coffeeify \
           -t [ browserify-ngannotate --ext .coffee --bar ] \
           index.coffee > index.js
```

**in package.json**
``` json
"browserify": {
  "transform": [
    "coffeeify",
    ["browserify-ngannotate", {"ext": ".coffee", bar: true}]
  ]
}
```


# finding good modules

Here are [some useful heuristics](http://substack.net/finding_modules)
for finding good modules on npm that work in the browser:

* I can install it with npm

* code snippet on the readme using require() - from a quick glance I should see
how to integrate the library into what I'm presently working on

* has a very clear, narrow idea about scope and purpose

* knows when to delegate to other libraries - doesn't try to do too many things itself

* written or maintained by authors whose opinions about software scope,
modularity, and interfaces I generally agree with (often a faster shortcut
than reading the code/docs very closely)

* inspecting which modules depend on the library I'm evaluating - this is baked
into the package page for modules published to npm

Other metrics like number of stars on github, project activity, or a slick
landing page, are not as reliable.

## module philosophy

People used to think that exporting a bunch of handy utility-style things would
be the main way that programmers would consume code because that is the primary
way of exporting and importing code on most other platforms and indeed still
persists even on npm.

However, this
[kitchen-sink mentality](https://github.com/substack/node-mkdirp/issues/17)
toward including a bunch of thematically-related but separable functionality
into a single package appears to be an artifact for the difficulty of
publishing and discovery in a pre-github, pre-npm era.

There are two other big problems with modules that try to export a bunch of
functionality all in one place under the auspices of convenience: demarcation
turf wars and finding which modules do what.

Packages that are grab-bags of features
[waste a ton of time policing boundaries](https://github.com/jashkenas/underscore/search?q=%22special-case%22&ref=cmdform&type=Issues)
about which new features belong and don't belong.
There is no clear natural boundary of the problem domain in this kind of package
about what the scope is, it's all
[somebody's smug opinion](http://david.heinemeierhansson.com/2012/rails-is-omakase.html).

Node, npm, and browserify are not that. They are avowedly ala-carte,
participatory, and would rather celebrate disagreement and the dizzying
proliferation of new ideas and approaches than try to clamp down in the name of
conformity, standards, or "best practices".

Nobody who needs to do gaussian blur ever thinks "hmm I guess I'll start checking
generic mathematics, statistics, image processing, and utility libraries to see
which one has gaussian blur in it. Was it stats2 or image-pack-utils or
maths-extra or maybe underscore has that one?"
No. None of this. Stop it. They `npm search gaussian` and they immediately see
[ndarray-gaussian-filter](https://npmjs.org/package/ndarray-gaussian-filter) and
it does exactly what they want and then they continue on with their actual
problem instead of getting lost in the weeds of somebody's neglected grand
utility fiefdom.

# organizing modules

## ../../../../../../..と書かずに済ます方法

Not everything in an application properly belongs on the public npm and the
overhead of setting up a private npm or git repo is still rather large in many
cases. Here are some approaches for avoiding the `../../../../../../../`
relative paths problem.

### symlink

The simplest thing you can do is to symlink your app root directory into your
node_modules/ directory.

Did you know that [symlinks work on windows
too](http://www.howtogeek.com/howto/windows-vista/using-symlinks-in-windows-vista/)?

To link a `lib/` directory in your project root into `node_modules`, do:

```
ln -s ../lib node_modules/app
```

and now from anywhere in your project you'll be able to require files in `lib/`
by doing `require('app/foo.js')` to get `lib/foo.js`.

### node_modules

People sometimes object to putting application-specific modules into
node_modules because it is not obvious how to check in your internal modules
without also checking in third-party modules from npm.

The answer is quite simple! If you have a `.gitignore` file that ignores
`node_modules`:

```
node_modules
```

You can just add an exception with `!` for each of your internal application
modules:

```
node_modules/*
!node_modules/foo
!node_modules/bar
```

Please note that you can't *unignore* a subdirectory,
if the parent is already ignored. So instead of ignoring `node_modules`,
you have to ignore every directory *inside* `node_modules` with the
`node_modules/*` trick, and then you can add your exceptions.

Now anywhere in your application you will be able to `require('foo')` or
`require('bar')` without having a very large and fragile relative path.

If you have a lot of modules and want to keep them more separate from the
third-party modules installed by npm, you can just put them all under a
directory in `node_modules` such as `node_modules/app`:

```
node_modules/app/foo
node_modules/app/bar
```

Now you will be able to `require('app/foo')` or `require('app/bar')` from
anywhere in your application.

In your `.gitignore`, just add an exception for `node_modules/app`:

```
node_modules/*
!node_modules/app
```

If your application had transforms configured in package.json, you'll need to
create a separate package.json with its own transform field in your
`node_modules/foo` or `node_modules/app/foo` component directory because
transforms don't apply across module boundaries. This will make your modules
more robust against configuration changes in your application and it will be
easier to independently reuse the packages outside of your application.

### custom paths

You might see some places talk about using the `$NODE_PATH` environment variable
or `opts.paths` to add directories for node and browserify to look in to find
modules.

Unlike most other platforms, using a shell-style array of path directories with
`$NODE_PATH` is not as favorable in node compared to making effective use of the
`node_modules` directory.

This is because your application is more tightly coupled to a runtime
environment configuration so there are more moving parts and your application
will only work when your environment is setup correctly.

node and browserify both support but discourage the use of `$NODE_PATH`.

## non-javascript assets

There are many
[browserify transforms](https://github.com/substack/node-browserify/wiki/list-of-transforms)
you can use to do many things. Commonly, transforms are used to include
non-javascript assets into bundle files.

### brfs

One way of including any kind of asset that works in both node and the browser
is brfs.

brfs uses static analysis to compile the results of `fs.readFile()` and
`fs.readFileSync()` calls down to source contents at compile time.

For example, this `main.js`:

``` js
var fs = require('fs');
var html = fs.readFileSync(__dirname + '/robot.html', 'utf8');
console.log(html);
```

applied through brfs would become something like:

``` js
var fs = require('fs');
var html = "<b>beep boop</b>";
console.log(html);
```

when run through brfs.

This is handy because you can reuse the exact same code in node and the browser,
which makes sharing modules and testing much simpler.

`fs.readFile()` and `fs.readFileSync()` accept the same arguments as in node,
which makes including inline image assets as base64-encoded strings very easy:

``` js
var fs = require('fs');
var imdata = fs.readFileSync(__dirname + '/image.png', 'base64');
var img = document.createElement('img');
img.setAttribute('src', 'data:image/png;base64,' + imdata);
document.body.appendChild(img);
```

If you have some css you want to inline into your bundle, you can do that too
with the assistence of a module such as
[insert-css](https://npmjs.org/package/insert-css):

``` js
var fs = require('fs');
var insertStyle = require('insert-css');

var css = fs.readFileSync(__dirname + '/style.css', 'utf8');
insertStyle(css);
```

Inserting css this way works fine for small reusable modules that you distribute
with npm because they are fully-contained, but if you want a more wholistic
approach to asset management using browserify, check out
[atomify](https://www.npmjs.org/package/atomify) and
[parcelify](https://www.npmjs.org/package/parcelify).

### hbsify

### jadeify

### reactify

## reusable components

Putting these ideas about code organization together, we can build a reusable UI
component that we can reuse across our application or in other applications.

Here is a bare-bones example of an empty widget module:

``` js
module.exports = Widget;

function Widget (opts) {
    if (!(this instanceof Widget)) return new Widget(opts);
    this.element = document.createElement('div');
}

Widget.prototype.appendTo = function (target) {
    if (typeof target === 'string') target = document.querySelector(target);
    target.appendChild(this.element);
};
```

Handy javascript constructor tip: you can include a `this instanceof Widget`
check like above to let people consume your module with `new Widget` or
`Widget()`. It's nice because it hides an implementation detail from your API
and you still get the performance benefits and indentation wins of using
prototypes.

To use this widget, just use `require()` to load the widget file, instantiate
it, and then call `.appendTo()` with a css selector string or a dom element.

Like this:

``` js
var Widget = require('./widget.js');
var w = Widget();
w.appendTo('#container');
```

and now your widget will be appended to the DOM.

Creating HTML elements procedurally is fine for very simple content but gets
very verbose and unclear for anything bigger. Luckily there are many transforms
available to ease importing HTML into your javascript modules.

Let's extend our widget example using [brfs](https://npmjs.org/package/brfs). We
can also use [domify](https://npmjs.org/package/domify) to turn the string that
`fs.readFileSync()` returns into an html dom element:

``` js
var fs = require('fs');
var domify = require('domify');

var html = fs.readFileSync(__dirname + '/widget.html', 'utf8');

module.exports = Widget;

function Widget (opts) {
    if (!(this instanceof Widget)) return new Widget(opts);
    this.element = domify(html);
}

Widget.prototype.appendTo = function (target) {
    if (typeof target === 'string') target = document.querySelector(target);
    target.appendChild(this.element);
};
```

and now our widget will load a `widget.html`, so let's make one:

``` html
<div class="widget">
  <h1 class="name"></h1>
  <div class="msg"></div>
</div>
```

It's often useful to emit events. Here's how we can emit events using the
built-in `events` module and the [inherits](https://npmjs.org/package/inherits)
module:

``` js
var fs = require('fs');
var domify = require('domify');
var inherits = require('inherits');
var EventEmitter = require('events').EventEmitter;

var html = fs.readFileSync(__dirname + '/widget.html', 'utf8');

inherits(Widget, EventEmitter);
module.exports = Widget;

function Widget (opts) {
    if (!(this instanceof Widget)) return new Widget(opts);
    this.element = domify(html);
}

Widget.prototype.appendTo = function (target) {
    if (typeof target === 'string') target = document.querySelector(target);
    target.appendChild(this.element);
    this.emit('append', target);
};
```

Now we can listen for `'append'` events on our widget instance:

``` js
var Widget = require('./widget.js');
var w = Widget();
w.on('append', function (target) {
    console.log('appended to: ' + target.outerHTML);
});
w.appendTo('#container');
```

We can add more methods to our widget to set elements on the html:

``` js
var fs = require('fs');
var domify = require('domify');
var inherits = require('inherits');
var EventEmitter = require('events').EventEmitter;

var html = fs.readFileSync(__dirname + '/widget.html', 'utf8');

inherits(Widget, EventEmitter);
module.exports = Widget;

function Widget (opts) {
    if (!(this instanceof Widget)) return new Widget(opts);
    this.element = domify(html);
}

Widget.prototype.appendTo = function (target) {
    if (typeof target === 'string') target = document.querySelector(target);
    target.appendChild(this.element);
};

Widget.prototype.setName = function (name) {
    this.element.querySelector('.name').textContent = name;
}

Widget.prototype.setMessage = function (msg) {
    this.element.querySelector('.msg').textContent = msg;
}
```

If setting element attributes and content gets too verbose, check out
[hyperglue](https://npmjs.org/package/hyperglue).

Now finally, we can toss our `widget.js` and `widget.html` into
`node_modules/app-widget`. Since our widget uses the
[brfs](https://npmjs.org/package/brfs) transform, we can create a `package.json`
with:

``` json
{
  "name": "app-widget",
  "version": "1.0.0",
  "private": true,
  "main": "widget.js",
  "browserify": {
    "transform": [ "brfs" ]
  },
  "dependencies": {
    "brfs": "^1.1.1",
    "inherits": "^2.0.1"
  }
}
```

And now whenever we `require('app-widget')` from anywhere in our application,
brfs will be applied to our `widget.js` automatically!
Our widget can even maintain its own dependencies. This way we can update
dependencies in one widgets without worrying about breaking changes cascading
over into other widgets.

Make sure to add an exclusion in your `.gitignore` for
`node_modules/app-widget`:

```
node_modules/*
!node_modules/app-widget
```

You can read more about [shared rendering in node and the
browser](http://substack.net/shared_rendering_in_node_and_the_browser) if you
want to learn about sharing rendering logic between node and the browser using
browserify and some streaming html libraries.

# testing in node and the browser

Testing modular code is very easy! One of the biggest benefits of modularity is
that your interfaces become much easier to instantiate in isolation and so it's
easy to make automated tests.

Unfortunately, few testing libraries play nicely out of the box with modules and
tend to roll their own idiosyncratic interfaces with implicit globals and obtuse
flow control that get in the way of a clean design with good separation.

People also make a huge fuss about "mocking" but it's usually not necessary if
you design your modules with testing in mind. Keeping IO separate from your
algorithms, carefully restricting the scope of your module, and accepting
callback parameters for different interfaces can all make your code much easier
to test.

For example, if you have a library that does both IO and speaks a protocol,
[consider separating the IO layer from the
protocol](https://www.youtube.com/watch?v=g5ewQEuXjsQ#t=12m30)
using an interface like [streams](https://github.com/substack/stream-handbook).

Your code will be easier to test and reusable in different contexts that you
didn't initially envision. This is a recurring theme of testing: if your code is
hard to test, it is probably not modular enough or contains the wrong balance of
abstractions. Testing should not be an afterthought, it should inform your
whole design and it will help you to write better interfaces.

## testing libraries

### [tape](https://npmjs.org/package/tape)

Tape was specifically designed from the start to work well in both node and
browserify. Suppose we have an `index.js` with an async interface:

``` js
module.exports = function (x, cb) {
    setTimeout(function () {
        cb(x * 100);
    }, 1000);
};
```

Here's how we can test this module using [tape](https://npmjs.org/package/tape).
Let's put this file in `test/beep.js`:

``` js
var test = require('tape');
var hundreder = require('../');

test('beep', function (t) {
    t.plan(1);

    hundreder(5, function (n) {
        t.equal(n, 500, '5*100 === 500');
    });
});
```

Because the test file lives in `test/`, we can require the `index.js` in the
parent directory by doing `require('../')`. `index.js` is the default place that
node and browserify look for a module if there is no package.json in that
directory with a `main` field.

We can `require()` tape like any other library after it has been installed with
`npm install tape`.

The string `'beep'` is an optional name for the test.
The 3rd argument to `t.equal()` is a completely optional description.

The `t.plan(1)` says that we expect 1 assertion. If there are not enough
assertions or too many, the test will fail. An assertion is a comparison
like `t.equal()`. tape has assertion primitives for:

* t.equal(a, b) - compare a and b strictly with `===`
* t.deepEqual(a, b) - compare a and b recursively
* t.ok(x) - fail if `x` is not truthy

and more! You can always add an additional description argument.

Running our module is very simple! To run the module in node, just run
`node test/beep.js`:

```
$ node test/beep.js
TAP version 13
# beep
ok 1 5*100 === 500

1..1
# tests 1
# pass  1

# ok
```

The output is printed to stdout and the exit code is 0.

To run our code in the browser, just do:

```
$ browserify test/beep.js > bundle.js
```

then plop `bundle.js` into a `<script>` tag:

```
<script src="bundle.js"></script>
```

and load that html in a browser. The output will be in the debug console which
you can open with F12, ctrl-shift-j, or ctrl-shift-k depending on the browser.

This is a bit cumbersome to run our tests in a browser, but you can install the
`testling` command to help. First do:

```
npm install -g testling
```

And now just do `browserify test/beep.js | testling`:

```
$ browserify test/beep.js | testling

TAP version 13
# beep
ok 1 5*100 === 500

1..1
# tests 1
# pass  1

# ok
```

`testling` will launch a real browser headlessly on your system to run the tests.

Now suppose we want to add another file, `test/boop.js`:

``` js
var test = require('tape');
var hundreder = require('../');

test('fraction', function (t) {
    t.plan(1);

    hundreder(1/20, function (n) {
        t.equal(n, 5, '1/20th of 100');
    });
});

test('negative', function (t) {
    t.plan(1);

    hundreder(-3, function (n) {
        t.equal(n, -300, 'negative number');
    });
});
```

Here our test has 2 `test()` blocks. The second test block won't start to
execute until the first is completely finished, even though it is asynchronous.
You can even nest test blocks by using `t.test()`.

We can run `test/boop.js` with node directly as with `test/beep.js`, but if we
want to run both tests, there is a minimal command-runner we can use that comes
with tape. To get the `tape` command do:

```
npm install -g tape
```

and now you can run:

```
$ tape test/*.js
TAP version 13
# beep
ok 1 5*100 === 500
# fraction
ok 2 1/20th of 100
# negative
ok 3 negative number

1..3
# tests 3
# pass  3

# ok
```

and you can just pass `test/*.js` to browserify to run your tests in the
browser:

```
$ browserify test/* | testling

TAP version 13
# beep
ok 1 5*100 === 500
# fraction
ok 2 1/20th of 100
# negative
ok 3 negative number

1..3
# tests 3
# pass  3

# ok
```

Putting together all these steps, we can configure `package.json` with a test
script:

``` json
{
  "name": "hundreder",
  "version": "1.0.0",
  "main": "index.js",
  "devDependencies": {
    "tape": "^2.13.1",
    "testling": "^1.6.1"
  },
  "scripts": {
    "test": "tape test/*.js",
    "test-browser": "browserify test/*.js | testlingify"
  }
}
```

Now you can do `npm test` to run the tests in node and `npm run test-browser` to
run the tests in the browser. You don't need to worry about installing commands
with `-g` when you use `npm run`: npm automatically sets up the `$PATH` for all
packages installed locally to the project.

If you have some tests that only run in node and some tests that only run in the
browser, you could have subdirectories in `test/` such as `test/server` and
`test/browser` with the tests that run both places just in `test/`. Then you
could just add the relevant directory to the globs:

``` json
{
  "name": "hundreder",
  "version": "1.0.0",
  "main": "index.js",
  "devDependencies": {
    "tape": "^2.13.1",
    "testling": "^1.6.1"
  },
  "scripts": {
    "test": "tape test/*.js test/server/*.js",
    "test-browser": "browserify test/*.js test/browser/*.js | testling"
  }
}
```

and now server-specific and browser-specific tests will be run in addition to
the common tests.

If you want something even slicker, check out
[prova](https://www.npmjs.org/package/prova) once you have gotten the basic
concepts.

### assert

The core assert module is a fine way to write simple tests too, although it can
sometimes be tricky to ensure that the correct number of callbacks have fired.

You can solve that problem with tools like
[macgyver](https://www.npmjs.org/package/macgyver) but it is appropriately DIY.

## code coverage

### coverify

A simple way to check code coverage in browserify is to use the
[coverify](https://npmjs.org/package/coverify) transform.

```
$ browserify -t coverify test/*.js | node | coverify
```

or to run your tests in a real browser:

```
$ browserify -t coverify test/*.js | testling | coverify
```

coverify works by transforming the source of each package so that each
expression is wrapped in a `__coverageWrap()` function.

Each expression in the program gets a unique ID and the `__coverageWrap()`
function will print `COVERED $FILE $ID` the first time the expression is
executed.

Before the expressions run, coverify prints a `COVERAGE $FILE $NODES` message to
log the expression nodes across the entire file as character ranges.

Here's what the output of a full run looks like:

```
$ browserify -t coverify test/whatever.js | node
COVERAGE "/home/substack/projects/defined/test/whatever.js" [[14,28],[14,28],[0,29],[41,56],[41,56],[30,57],[95,104],[95,105],[126,146],[126,146],[115,147],[160,194],[160,194],[152,195],[200,217],[200,218],[76,220],[59,221],[59,222]]
COVERED "/home/substack/projects/defined/test/whatever.js" 2
COVERED "/home/substack/projects/defined/test/whatever.js" 1
COVERED "/home/substack/projects/defined/test/whatever.js" 0
COVERAGE "/home/substack/projects/defined/index.js" [[48,49],[55,71],[51,71],[73,76],[92,104],[92,118],[127,139],[120,140],[172,195],[172,196],[0,204],[0,205]]
COVERED "/home/substack/projects/defined/index.js" 11
COVERED "/home/substack/projects/defined/index.js" 10
COVERED "/home/substack/projects/defined/test/whatever.js" 5
COVERED "/home/substack/projects/defined/test/whatever.js" 4
COVERED "/home/substack/projects/defined/test/whatever.js" 3
COVERED "/home/substack/projects/defined/test/whatever.js" 18
COVERED "/home/substack/projects/defined/test/whatever.js" 17
COVERED "/home/substack/projects/defined/test/whatever.js" 16
TAP version 13
# whatever
COVERED "/home/substack/projects/defined/test/whatever.js" 7
COVERED "/home/substack/projects/defined/test/whatever.js" 6
COVERED "/home/substack/projects/defined/test/whatever.js" 10
COVERED "/home/substack/projects/defined/test/whatever.js" 9
COVERED "/home/substack/projects/defined/test/whatever.js" 8
COVERED "/home/substack/projects/defined/test/whatever.js" 13
COVERED "/home/substack/projects/defined/test/whatever.js" 12
COVERED "/home/substack/projects/defined/test/whatever.js" 11
COVERED "/home/substack/projects/defined/index.js" 0
COVERED "/home/substack/projects/defined/index.js" 2
COVERED "/home/substack/projects/defined/index.js" 1
COVERED "/home/substack/projects/defined/index.js" 5
COVERED "/home/substack/projects/defined/index.js" 4
COVERED "/home/substack/projects/defined/index.js" 3
COVERED "/home/substack/projects/defined/index.js" 7
COVERED "/home/substack/projects/defined/index.js" 6
COVERED "/home/substack/projects/defined/test/whatever.js" 15
COVERED "/home/substack/projects/defined/test/whatever.js" 14
ok 1 should be equal

1..1
# tests 1
# pass  1

# ok
```

These COVERED and COVERAGE statements are just printed on stdout and they can be
fed into the `coverify` command to generate prettier output:

```
$ browserify -t coverify test/whatever.js | node | coverify
TAP version 13
# whatever
ok 1 should be equal

1..1
# tests 1
# pass  1

# ok

# /home/substack/projects/defined/index.js: line 6, column 9-32

          console.log('whatever');
          ^^^^^^^^^^^^^^^^^^^^^^^^

# coverage: 30/31 (96.77 %)
```

To include code coverage into your project, you can add an entry into the
`package.json` scripts field:

``` json
{
  "scripts": {
    "test": "tape test/*.js",
    "coverage": "browserify -t coverify test/*.js | node | coverify"
  }
}
```

There is also a [covert](https://npmjs.com/package/covert) package that
simplifies the browserify and coverify setup:

``` json
{
  "scripts": {
    "test": "tape test/*.js",
    "coverage": "covert test/*.js"
  }
}
```

To install coverify or covert as a devDependency, run
`npm install -D coverify` or `npm install -D covert`.

# bundling

This section covers bundling in more detail.

Bundling is the step where starting from the entry files, all the source files
in the dependency graph are walked and packed into a single output file.

## saving bytes

One of the first things you'll want to tweak is how the files that npm installs
are placed on disk to avoid duplicates.

When you do a clean install in a directory, npm will ordinarily factor out
similar versions into the topmost directory where 2 modules share a dependency.
However, as you install more packages, new packages will not be factored out
automatically. You can however use the `npm dedupe` command to factor out
packages for an already-installed set of packages in `node_modules/`. You could
also remove `node_modules/` and install from scratch again if problems with
duplicates persist.

browserify will not include the same exact file twice, but compatible versions
may differ slightly. browserify is also not version-aware, it will include the
versions of packages exactly as they are laid out in `node_modules/` according
to the `require()` algorithm that node uses.

You can use the `browserify --list` and `browserify --deps` commands to further
inspect which files are being included to scan for duplicates.

## standalone

You can generate UMD bundles with `--standalone` that will work in node, the
browser with globals, and AMD environments.

Just add `--standalone NAME` to your bundle command:

```
$ browserify foo.js --standalone xyz > bundle.js
```

This command will export the contents of `foo.js` under the external module name
`xyz`. If a module system is detected in the host environment, it will be used.
Otherwise a window global named `xyz` will be exported.

You can use dot-syntax to specify a namespace hierarchy:

```
$ browserify foo.js --standalone foo.bar.baz > bundle.js
```

If there is already a `foo` or a `foo.bar` in the host environment in window
global mode, browserify will attach its exports onto those objects. The AMD and
`module.exports` modules will behave the same.

Note however that standalone only works with a single entry or directly-required
file.

## external bundles

## ignoring and excluding

In browserify parlance, "ignore" means: replace the definition of a module with
an empty object. "exclude" means: remove a module completely from a dependency graph.

Another way to achieve many of the same goals as ignore and exclude is the
"browser" field in package.json, which is covered elsewhere in this document.

### ignoring

Ignoring is an optimistic strategy designed to stub in an empty definition for
node-specific modules that are only used in some codepaths. For example, if a
module requires a library that only works in node but for a specific chunk of
the code:

``` js
var fs = require('fs');
var path = require('path');
var mkdirp = require('mkdirp');

exports.convert = convert;
function convert (src) {
    return src.replace(/beep/g, 'boop');
}

exports.write = function (src, dst, cb) {
    fs.readFile(src, function (err, src) {
        if (err) return cb(err);
        mkdirp(path.dirname(dst), function (err) {
            if (err) return cb(err);
            var out = convert(src);
            fs.writeFile(dst, out, cb);
        });
    });
};
```

browserify already "ignores" the `'fs'` module by returning an empty object, but
the `.write()` function here won't work in the browser without an extra step like
a static analysis transform or a runtime storage fs abstraction.

However, if we really want the `convert()` function but don't want to see
`mkdirp` in the final bundle, we can ignore mkdirp with `b.ignore('mkdirp')` or
`browserify --ignore mkdirp`. The code will still work in the browser if we
don't call `write()` because `require('mkdirp')` won't throw an exception, just
return an empty object.

Generally speaking it's not a good idea for modules that are primarily
algorithmic (parsers, formatters) to do IO themselves but these tricks can let
you use those modules in the browser anyway.

To ignore `foo` on the command-line do:

```
browserify --ignore foo
```

To ignore `foo` from the api with some bundle instance `b` do:

``` js
b.ignore('foo')
```

### excluding

Another related thing we might want is to completely remove a module from the
output so that `require('modulename')` will fail at runtime. This is useful if
we want to split things up into multiple bundles that will defer in a cascade to
previously-defined `require()` definitions.

For example, if we have a vendored standalone bundle for jquery that we don't want to appear in
the primary bundle:

```
$ npm install jquery
$ browserify -r jquery --standalone jquery > jquery-bundle.js
```

then we want to just `require('jquery')` in a `main.js`:

``` js
var $ = require('jquery');
$(window).click(function () { document.body.bgColor = 'red' });
```

defering to the jquery dist bundle so that we can write:

``` html
<script src="jquery-bundle.js"></script>
<script src="bundle.js"></script>
```

and not have the jquery definition show up in `bundle.js`, then while compiling
the `main.js`, you can `--exclude jquery`:

```
browserify main.js --exclude jquery > bundle.js
```

To exclude `foo` on the command-line do:

```
browserify --exclude foo
```

To exclude `foo` from the api with some bundle instance `b` do:

``` js
b.exclude('foo')
```

## browserify cdn

# shimming

Unfortunately, some packages are not written with node-style commonjs exports.
For modules that export their functionality with globals or AMD, there are
packages that can help automatically convert these troublesome packages into
something that browserify can understand.

## browserify-shim

One way to automatically convert non-commonjs packages is with
[browserify-shim](https://npmjs.org/package/browserify-shim).

[browserify-shim](https://npmjs.org/package/browserify-shim) is loaded as a
transform and also reads a `"browserify-shim"` field from `package.json`.

Suppose we need to use a troublesome third-party library we've placed in
`./vendor/foo.js` that exports its functionality as a window global called
`FOO`. We can set up our `package.json` with:

``` json
{
  "browserify": {
    "transform": "browserify-shim"
  },
  "browserify-shim": {
    "./vendor/foo.js": "FOO"
  }
}
```

and now when we `require('./vendor/foo.js')`, we get the `FOO` variable that
`./vendor/foo.js` tried to put into the global scope, but that attempt was
shimmed away into an isolated context to prevent global pollution.

We could even use the [browser field](#browser-field) to make `require('foo')`
work instead of always needing to use a relative path to load `./vendor/foo.js`:

``` json
{
  "browser": {
    "foo": "./vendor/foo.js"
  },
  "browserify": {
    "transform": "browserify-shim"
  },
  "browserify-shim": {
    "foo": "FOO"
  }
}
```

Now `require('foo')` will return the `FOO` export that `./vendor/foo.js` tried
to place on the global scope.

# partitioning

Most of the time, the default method of bundling where one or more entry files
map to a single bundled output file is perfectly adequate, particularly
considering that bundling minimizes latency down to a single http request to
fetch all the javascript assets.

However, sometimes this initial penalty is too high for parts of a website that
are rarely or never used by most visitors such as an admin panel.
This partitioning can be accomplished with the technique covered in the
[ignoring and excluding](#ignoring-and-excluding) section, but factoring out
shared dependencies manually can be tedious for a large and fluid dependency
graph.

Luckily, there are plugins that can automatically factor browserify output into
separate bundle payloads.

## factor-bundle

[factor-bundle](https://www.npmjs.org/package/factor-bundle) splits browserify
output into multiple bundle targets based on entry-point. For each entry-point,
an entry-specific output file is built. Files that are needed by two or more of
the entry files get factored out into a common bundle.

For example, suppose we have 2 pages: /x and /y. Each page has an entry point,
`x.js` for /x and `y.js` for /y.

We then generate page-specific bundles `bundle/x.js` and `bundle/y.js` with
`bundle/common.js` containing the dependencies shared by both `x.js` and `y.js`:

```
browserify x.js y.js -p [ factor-bundle -o bundle/x.js -o bundle/y.js ] \
  -o bundle/common.js
```

Now we can simply put 2 script tags on each page. On /x we would put:

``` html
<script src="/bundle/common.js"></script>
<script src="/bundle/x.js"></script>
```

and on page /y we would put:

``` html
<script src="/bundle/common.js"></script>
<script src="/bundle/y.js"></script>
```

You could also load the bundles asynchronously with ajax or by inserting a
script tag into the page dynamically but factor-bundle only concerns itself with
generating the bundles, not with loading them.

## partition-bundle

[partition-bundle](https://www.npmjs.org/package/partition-bundle) handles
splitting output into multiple bundles like factor-bundle, but includes a
built-in loader using a special `loadjs()` function.

partition-bundle takes a json file that maps source files to bundle files:

```
{
  "entry.js": ["./a"],
  "common.js": ["./b"],
  "common/extra.js": ["./e", "./d"]
}
```

Then partition-bundle is loaded as a plugin and the mapping file, output
directory, and destination url path (required for dynamic loading) are passed
in:

```
browserify -p [ partition-bundle --map mapping.json \
  --output output/directory --url directory ]
```

Now you can add:

``` html
<script src="entry.js"></script>
```

to your page to load the entry file. From inside the entry file, you can
dynamically load other bundles with a `loadjs()` function:

``` js
a.addEventListener('click', function() {
  loadjs(['./e', './d'], function(e, d) {
    console.log(e, d);
  });
});
```

# compiler pipeline

Browserifyバージョン5以降では、コード変換処理を行うパイプラインにアクセス可能となりました。
パイプラインとは、変換の目的ごとに分割された複数のストリームが連結されたもので、[labeled-stream-splicer](https://www.npmjs.org/package/labeled-stream-splicer)で実装されています。

パイプラインのAPIを用いれば、コード変換処理の内部的なストリームに対して、直に transform の追加や削除ができます。
これを活用することで、ファイル変更監視や複数エントリポイントに応じた分割ビルドといった、高度なカスタマイズができるのです。

たとえば、依存ファイルのIDに整数を用いるという標準動作のかわりに、ファイルのハッシュ値を用いるというカスタマイズを考えましょう。
まずは `deps` という名前のパイプラインに対し、ストリームに流れてきたファイルのハッシュ値を計算し変数にキャッシュする transfrom を追加します。
次に `label` という名前のパイプラインに対しては、デフォルトの transform を削除する代わりに、ストリームに流れてきた依存ファイルのIDをハッシュ値で上書きするという transfrom を追加します。

``` js
var browserify = require('browserify');
var through = require('through2');
var shasum = require('shasum');

var b = browserify('./main.js');

var hashes = {};
var hasher = through.obj(function (row, enc, next) {
    hashes[row.id] = shasum(row.source);
    this.push(row);
    next();
});
b.pipeline.get('deps').push(hasher);

var labeler = through.obj(function (row, enc, next) {
    row.id = hashes[row.id];

    Object.keys(row.deps).forEach(function (key) {
        row.deps[key] = hashes[row.deps[key]];
    });

    this.push(row);
    next();
});
b.pipeline.get('label').splice(0, 1, labeler);

b.bundle().pipe(process.stdout);
```

これで、ファイルのIDにファイルのハッシュ値が使われた出力が得られました。

```
$ node bundle.js
(function e(t,n,r){function s(o,u){if(!n[o]){if(!t[o]){var a=typeof require=="function"&&require;if(!u&&a)return a(o,!0);if(i)return i(o,!0);var f=new Error("Cannot find module '"+o+"'");throw f.code="MODULE_NOT_FOUND",f}var l=n[o]={exports:{}};t[o][0].call(l.exports,function(e){var n=t[o][1][e];return s(n?n:e)},l,l.exports,e,t,n,r)}return n[o].exports}var i=typeof require=="function"&&require;for(var o=0;o<r.length;o++)s(r[o]);return s})({"5f0a0e3a143f2356582f58a70f385f4bde44f04b":[function(require,module,exports){
var foo = require('./foo.js');
var bar = require('./bar.js');

console.log(foo(3) + bar(4));

},{"./bar.js":"cba5983117ae1d6699d85fc4d54eb589d758f12b","./foo.js":"736100869ec2e44f7cfcf0dc6554b055e117c53c"}],"cba5983117ae1d6699d85fc4d54eb589d758f12b":[function(require,module,exports){
module.exports = function (n) { return n * 100 };

},{}],"736100869ec2e44f7cfcf0dc6554b055e117c53c":[function(require,module,exports){
module.exports = function (n) { return n + 1 };

},{}]},{},["5f0a0e3a143f2356582f58a70f385f4bde44f04b"]);
```

注意点として、 `label` パイプラインのデフォルト実装では除外すべきファイルのチェックなどをしているため、
上のサンプルのようにデフォルト実装を置き換えてしまうと他の機能を壊す可能性があります。
パイプラインの編集方法を説明するための簡略化したサンプルであることに気をつけてください。

## build your own browserify

## labeled phases

Each phase in the browserify pipeline has a label that you can hook onto. Fetch
a label with `.get(name)` to return a
[labeled-stream-splicer](https://npmjs.org/package/labeled-stream-splicer)
handle at the appropriate label. Once you have a handle, you can `.push()`,
`.pop()`, `.shift()`, `.unshift()`, and `.splice()` your own transform streams
into the pipeline or remove existing transform streams.

### recorder

The recorder is used to capture the inputs sent to the `deps` phase so that they
can be replayed on subsequent calls to `.bundle()`. Unlike in previous releases,
v5 can generate bundle output multiple times. This is very handy for tools like
watchify that re-bundle when a file has changed.

### deps

The `deps` phase expects entry and `require()` files or objects as input and
calls [module-deps](https://npmjs.org/package/module-deps) to generate a stream
of json output for all of the files in the dependency graph.

module-deps is invoked with some customizations here such as:

* setting up the browserify transform key for package.json
* filtering out external, excluded, and ignored files
* setting the default extensions for `.js` and `.json` plus options configured
in the `opts.extensions` parameter in the browserify constructor
* configuring a global [insert-module-globals](#insert-module-globals)
transform to detect and implement `process`, `Buffer`, `global`, `__dirname`,
and `__filename`
* setting up the list of node builtins which are shimmed by browserify

### json

This transform adds `module.exports=` in front of files with a `.json`
extension.

### unbom

This transform removes byte order markers, which are sometimes used by windows
text editors to indicate the endianness of files. These markers are ignored by
node, so browserify ignores them for compatibility.

### syntax

This transform checks for syntax errors using the
[syntax-error](https://npmjs.org/package/syntax-error) package to give
informative syntax errors with line and column numbers.

### sort

This phase uses [deps-sort](https://www.npmjs.org/package/deps-sort) to sort
the rows written to it in order to make the bundles deterministic.

### dedupe

The transform at this phase uses dedupe information provided by
[deps-sort](https://www.npmjs.org/package/deps-sort) in the `sort` phase to
remove files that have duplicate contents.

### label

This phase converts file-based IDs which might expose system path information
and inflate the bundle size into integer-based IDs.

The `label` phase will also normalize path names based on the `opts.basedir` or
`process.cwd()` to avoid exposing system path information.

### emit-deps

This phase emits a `'dep'` event for each row after the `label` phase.

### debug

If `opts.debug` was given to the `browserify()` constructor, this phase will
transform input to add `sourceRoot` and `sourceFile` properties which are used
by [browser-pack](https://npmjs.org/package/browser-pack) in the `pack` phase.

### pack

This phase converts rows with `'id'` and `'source'` parameters as input (among
others) and generates the concatenated javascript bundle as output
using [browser-pack](https://npmjs.org/package/browser-pack).

### wrap

This is an empty phase at the end where you can easily tack on custom post
transformations without interfering with existing mechanics.

## browser-unpack

[browser-unpack](https://npmjs.org/package/browser-unpack) converts a compiled
bundle file back into a format very similar to the output of
[module-deps](https://npmjs.org/package/module-deps).

This is very handy if you need to inspect or transform a bundle that has already
been compiled.

For example:

``` js
$ browserify src/main.js | browser-unpack
[
{"id":1,"source":"module.exports = function (n) { return n * 100 };","deps":{}}
,
{"id":2,"source":"module.exports = function (n) { return n + 1 };","deps":{}}
,
{"id":3,"source":"var foo = require('./foo.js');\nvar bar = require('./bar.js');\n\nconsole.log(foo(3) + bar(4));","deps":{"./bar.js":1,"./foo.js":2},"entry":true}
]
```

This decomposition is needed by tools such as
[factor-bundle](https://www.npmjs.org/package/factor-bundle)
and [bundle-collapser](https://www.npmjs.org/package/bundle-collapser).

# plugins

When loaded, plugins have access to the browserify instance itself.

## using plugins

Plugins should be used sparingly and only in cases where a transform or global
transform is not powerful enough to perform the desired functionality.

You can load a plugin with `-p` on the command-line:

```
$ browserify main.js -p foo > bundle.js
```

would load a plugin called `foo`. `foo` is resolved with `require()`, so to load
a local file as a plugin, preface the path with a `./` and to load a plugin from
`node_modules/foo`, just do `-p foo`.

You can pass options to plugins with square brackets around the entire plugin
expression, including the plugin name as the first argument:

```
$ browserify one.js two.js \
  -p [ factor-bundle -o bundle/one.js -o bundle/two.js ] \
  > common.js
```

This command-line syntax is parsed by the
[subarg](https://npmjs.org/package/subarg) package.

To see a list of browserify plugins, browse npm for packages with the keyword
"browserify-plugin": http://npmjs.org/browse/keyword/browserify-plugin

## authoring plugins

To author a plugin, write a package that exports a single function that will
receive a bundle instance and options object as arguments:

``` js
// example plugin

module.exports = function (b, opts) {
  // ...
}
```

Plugins operate on the bundle instance `b` directly by listening for events or
splicing transforms into the pipeline. Plugins should not overwrite bundle
methods unless they have a very good reason.
