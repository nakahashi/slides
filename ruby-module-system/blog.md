# Railsの風が吹いたら確認したい、Rubyのモジュールシステム

こんにちは、入社して3ヶ月くらい経ちましたnakahashiです。

六本木のポケモンGOプレイヤーが気前よくて、職場周辺のポケストップはいつも花びらが舞っているので、キャッキャウフフな生活を過ごしています。

さて、UUUM攻殻機動隊では、最近Railsの風が吹き始めています。業務でRubyをどっぷり使い込んだことは個人的にはないので、言語の基本を見なおしているところです。

Rubyの基礎を勉強するにあたり非常に重要、且つRailsしか使っていないと勘違いする可能性のある、Rubyのモジュールシステムについてまとめてみました。

<!-- more -->

## モジュールシステムって何？

ここでいうモジュールシステムとは、あるファイルが他のファイルの機能を取り込んで動作させる仕組みのことです。

Rubyでは、この仕組みの実現を、主に`Kernel.#require`メソッドにて実現しています。今回は、この`require`メソッド周辺の事情をまとめてみました。

### このポストでわかること

 * requireって何？
 * loadとの違いは？
 * require_relativeって？
 * Bundlerとrequire
 * autoload
 * Railsとrequire
 * includeとの違い（全然別物）

### このポストでわからないこと

 * requireの内部実装
 * Rubyのオブジェクト指向に関する詳細

## requireって何？

requireは、自分以外のファイルから機能を取り込むために利用します。

基本的には、取り込み対象のファイルの内容をその場で実行するイメージです。指定では、拡張子なしのファイル名を渡すようにします。

```ruby
require "./hoge"
```

上記は、カレントディレクトリにあるhoge.rbを取り込んでいる例です。簡単ですね。

## ロード対象パス

さて、カレントディレクトリ以外にあるファイルを取り込むにはどのようにしたらいいのでしょうか？ システムにインストールしたgemのrequireは？

requireが取り込み対象として捜査するディレクトリは実は色々あって、組み込み変数の`$LOAD_PATH`に格納されています。以下のコマンドを実行すると確認できます。

```shell
ruby -e 'puts $LOAD_PATH'
```

私の環境（OS X 10.11, ruby: 2.3.1, rbenv）では、`$LOAD_PATH`には以下のパスが格納されていました。

```
$ ruby -e 'puts $LOAD_PATH'
/Users/nakahashi/.rbenv/plugins/rbenv-gem-rehash
/usr/local/Cellar/rbenv/1.0.0/rbenv.d/exec/gem-rehash
/Users/nakahashi/.rbenv/versions/2.3.1/lib/ruby/gems/2.3.0/gems/did_you_mean-1.0.2/lib
/Users/nakahashi/.rbenv/versions/2.3.1/lib/ruby/site_ruby/2.3.0
/Users/nakahashi/.rbenv/versions/2.3.1/lib/ruby/site_ruby/2.3.0/x86_64-darwin15
/Users/nakahashi/.rbenv/versions/2.3.1/lib/ruby/site_ruby
/Users/nakahashi/.rbenv/versions/2.3.1/lib/ruby/vendor_ruby/2.3.0
/Users/nakahashi/.rbenv/versions/2.3.1/lib/ruby/vendor_ruby/2.3.0/x86_64-darwin15
/Users/nakahashi/.rbenv/versions/2.3.1/lib/ruby/vendor_ruby
/Users/nakahashi/.rbenv/versions/2.3.1/lib/ruby/2.3.0
/Users/nakahashi/.rbenv/versions/2.3.1/lib/ruby/2.3.0/x86_64-darwin15
```

---

## require_relative

requireはロードパスを起点として、相対パスでファイルを検索します。

ですが場合によっては、require呼び出し元のファイルを起点としてファイルを探索したくなることもあります。その場合に利用するのがrequire_relativeです。

```ruby
require_relative "../../sub"
```

## load

ここまではrequireの紹介をしてきましたが、ほとんど同じ機能を提供するメソッドとして`Kernel.#load`が存在します。

requireとの違いは、requireは一度取り込んだファイルを再度読み込もうとすると失敗するのに対し、loadは何度でも読み込む点です。

C言語では、同じヘッダが何度も読み込まれないよう、ヘッダファイルにプリプロセッサによるインクルードガードをわざわざ書いていましたが、requireは呼び側でコントロールしてくれるんですね。ありがたい。

loadの挙動が嬉しい場面は少なくとも製品レベルでのコードではあまりなく、基本的にはrequireを使うべきです。

使いどころとしては、irb/pryなどの対話型シェル実行時が挙げられます。読み込み対象ファイルを更新してから再度読み込みたくなった場合、requireでは失敗しますがloadだと成功します。

## require/load先のローカル変数

requireは読み込み対象ファイルの処理を実行しますが、対象ファイルの中身をその場に展開するのとは微妙に異なる動作があります。

その1つが、ローカル変数はrequire元から見えなくなるという点です。

例えば以下のローカル変数がトップレベルにあるファイルがあったとします。

sub.rb
```ruby
local_var = "hoge"
```

これをrequireしても、require元から`local_var`はスコープ外となります。

```shell
$ pry
[1] pry(main)> require "./sub.rb"
=> true
[2] pry(main)> puts local_var
NameError: undefined local variable or method `local_var' for main:Object
Did you mean?  local_variables
from (pry):2:in `__pry__'
```

そもそもrequireでローカル変数を受け渡すというのは、カプセル化の観点からするとかなりの悪手と思われますし、その悪手を防いでいるという意味でわかりやすい仕様かと思います。

ちなみに先程の例では、`local_bar`をクラス変数に変更すると、requireを使った受け渡しができます。（すべきではないでしょうが）

sub.rb
```ruby
@local_var = "hoge"
```

```shell
$ pry
[1] pry(main)> require "./sub.rb"
=> true
[2] pry(main)> puts @local_var
hoge
=> nil
```

## autoload

requireは即座にファイルの中身を展開しますが、パフォーマンス的にこれを許容できないケースもあるでしょう。その場合は`Kernel.#autoload`を利用できます。

autoloadでは、指定したClassやModuleが初めて参照したときにrequireをしてくれます。それまで実際の読み取りが遅延するとも言い換えることが出来るでしょう

sub.rb
```
class Dog
  def cry
    puts "ワンワン"
  end
  puts "Dog has loaded"
end
```

```shell
$ pry
[1] pry(main)> autoload :Dog, "./lib"
=> nil
[2] pry(main)> Dog.new.cry
"Dog has loaded"
ワンワン
=> nil
```

## Bundlerとrequire

Bundlerは、Ruby用のライブラリ管理ツールです。よほど書き捨てなスクリプトでない場合は、外部のライブラリはBundler経由で利用すべきです。

さて、BundlerではGemfileに必要なgem（外部ライブラリ）を記述し、`bundle install`コマンドで一括インストールできます。gemはシステム全体で共有するよりも、プロジェクトごとに個別のディレクトリにインストールするのがベストプラクティスとされています。

Hashieというgemをインストールしてみましょう。

Gemfile
```
source 'http://rubygems.org'

gem 'hashie'
```

指定したgemをインストールします。`--path vendor/bundle`とすることで、プロジェクト固有のディレクトリにインストールしました。

```shell
bundle install --path vendor/bundle
```

さて、Bundlerでインストールしたgemは、どのようにrequireすればいいのでしょうか？

Bundlerには、Bundlerがインストールしてくれたgem群を一括でrequireするためのBundler.requireメソッドが用意されています。

従って、以下のように記述すれば、Gemfileに指定したgemを簡単にrequireすることができます。

```ruby
require 'bundler'
Bundler.require

employee = Hashie::Mash.new
```

## Railsの場合

Railsには、require関連で興味深い機能があります。

まず、自分で定義したModelやControllerなどのClassは、Railsが自動的にrequire対象としてくれます。いちいちrequireを書く必要がありません。

その上で、上記以外で何らかの機能をRailsアプリに追加したい場合、Rails 4までは以下のようにするのが定石でした。

まずconfig/application.rbに以下の記述を追加し、`lib`フォルダをautoload対象に追加します。

```
# to auto load lib/ directory
config.autoload_paths += %W(#{config.root}/lib)
```

そして、`lib`フォルダ内にRailsの規約に沿ったファイル名で、任意の処理を実装します。

広く流通しているこの手法ですが、[Rails 5からは任意のクラスのオートロード機能は制限されているようです。](https://techblog.recruitjobs.net/development/rails5_release_note_first_part) Rails 5を利用する場合は気をつけましょう。

## includeって？

一瞬似たような機能に見えるincludeですが、requireとincludeは全く異なる機能です。

includeは、指定したModuleの機能をClass or Moduleにミックスインする機能です。requireによる読み込みは（基本的には）ファイルの中身を展開するだけですが、includeの場合、定義済みのModule内のメソッドやClassを呼び元に合成する処理が動きます。

当然のことながら、includeしたいModuleが別ファイルで定義されている場合（その場合がほとんどだと思いますが）、先にrequireしておかないといけません。

## まとめ

UUUMでは、Ruby好きなエンジニア大歓迎です。
