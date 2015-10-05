Railsテスティングガイド
=====================================

本ガイドは、アプリケーションをテストするためにRailsに組み込まれているメカニズムについて解説します。

このガイドの内容:

* Rails testing terminology.
* アプリケーションに対する単体テスト、機能テスト、結合テストの実施
* その他の著名なテスティング方法とプラグインの紹介

--------------------------------------------------------------------------------

Railsアプリケーションでテストを作成しなければならない理由
--------------------------------------------

Railsを使用すれば、テストをきわめて簡単に作成できます。テストの作成は、モデルやコントローラを作成する時点でテストコードのスケルトンを作成することから始まります。

Railsのテストが作成されていれば、後はそれを単に実行するだけで、特に大規模なリファクタリングを行なう際にコードが期待どおりに動作していることを即座に確認できます。

Railsのテストはブラウザのリクエストをシミュレートできるので、ブラウザを手動で操作せずにアプリケーションのレスポンスをテストできます。

テストを導入する
-----------------------

テスティングのサポートは最初期からRailsに組み込まれています。決して、最近テスティングが流行っていてクールだから導入してみた、というようなその場の思い付きで導入されたものではありません。Railsアプリケーションは、ほぼ間違いなくデータベースと密接なやりとりを行いますので、テスティングにもデータベースが必要となります。効率のよいテストを作成するには、データベースの設定方法とサンプルデータの導入方法を理解しておく必要があります。

### test環境

デフォルトでは、すべてのRailsアプリケーションにはdevelopment、test、productionの3つの環境があります。それぞれの環境におけるデータベース設定は`config/database.yml`で行います。

テスティング専用のデータベースがあれば、それを設定して他の環境から切り離された専用のテストデータにアクセスすることができます。テストを実行すればテストデータは確実に元の状態から変わってしまうので、development環境やproduction環境のデータベースにあるデータには決してアクセスしません。

### Railsを即座にテスト用に設定する

`rails new` _application_name_でRailsアプリケーションを作成すると、その場で`test`フォルダが作成されます。このフォルダの内容は次のようになっています。

```bash
$ ls -F test
controllers/    helpers/        mailers/        test_helper.rb
fixtures/       integration/    models/
```

`models`ディレクトリはモデル用のテストの置き場所であり、`controllers`ディレクトリはコントローラ用のテストの置き場所です。`integration`ディレクトリは任意の数のコントローラとやりとりするテストを置く場所です。

フィクスチャはテストデータを編成する方法の1つであり、`fixtures`フォルダに置かれます。

`test_helper.rb`にはテスティングのデフォルト設定を記入します。

### フィクスチャのしくみ

よいテストを作成するにはよいテストデータを準備する必要があることを理解しておく必要があります。
Railsでは、テストデータの定義とカスタマイズはフィクスチャで行うことができます。
網羅的なドキュメントについては、[フィクスチャAPIドキュメント](http://api.rubyonrails.org/classes/ActiveRecord/FixtureSet.html)を参照してください。

#### フィクスチャとは何か

_フィクスチャ (fixture)_とはサンプルデータを言い換えたものです。フィクスチャを使用することで、事前に定義したデータをテスト実行直前にtestデータベースに導入することができます。フィクスチャFixtures are database independent written in YAML. There is one file per model.

フィクスチャファイルは`test/fixtures`の下に置かれます。`rails generate model`を実行すると、モデルのフィクスチャスタブが自動的に作成され、このディレクトリに置かれます。

#### YAML

YAML形式のフィクスチャは人間にとって読みやすく、サンプルデータを容易に記述することができます。この形式のフィクスチャには**.yml**というファイル拡張子が与えられます (`users.yml`など)。

YAMLフィクスチャファイルのサンプルを以下に示します。

```yaml
# この行はYAMLのコメントである
david:
  name: David Heinemeier Hansson
  birthday: 1979-10-15
  profession: Systems development

steve:
  name: Steve Ross Kellock
  birthday: 1974-09-27
  profession: guy with keyboard 
```

各フィクスチャは名前とコロンで始まり、その後にコロンで区切られたキー/値ペアのリストがインデント付きで置かれます。[REVIEW]通常、レコード間は空行で区切られます。行の先頭に#文字を置くことで、フィクスチャファイルにコメントを追加できます。'yes'や'no'などのYAMLキーワードに似たキーについては、引用符で囲むことでYAMLパーサーが正常に動作できます。

[関連付け](/association_basics.html)を使用している場合は、2つの異なるフィクスチャの間に参照ノードを1つ定義すれば済みます。Here's an example with
a `belongs_to`/`has_many` association:

```yaml
# fixtures/categories.ymlの内容:
about:
  name: About

# fixtures/articles.ymlの内容
one:
  title: Welcome to Rails!
  body: Hello world!
  category: about
```

NOTE: 名前で互いを参照する関連付けの場合、フィクスチャで`id:`属性を指定することはできません。Railsはテストの実行中に�、一貫性を保つために自動的に主キーを割り当てます。フィクスチャで`id:`属性を指定するとこの自動割り当てがしなくなります。関連付けの詳細な動作については、[フィクスチャAPIドキュメント](http://api.rubyonrails.org/classes/ActiveRecord/FixtureSet.html)を参照してください。

#### ERB

ERBは、テンプレート内にRubyコードを埋め込むのに使用されます。YAMLフィクスチャ形式のファイルは、Railsに読み込まれたときにERBによる事前処理が行われます。ERBを活用すれば、Rubyで一部のサンプルデータを生成できます。たとえば、以下のコードを使用すれば1000人のユーザーを生成できます。

```erb
<% 1000.times do |n| %>
user_<%= n %>:
  username: <%= "user#{n}" %>
  email: <%= "user#{n}@example.com" %> 
<% end %>
```

#### フィクスチャの動作

Railsはデフォルトで、`test/fixtures`フォルダにあるすべてのフィクスチャを自動的に読み込み、モデルやコントローラのテストで使用します。フィクスチャの読み込みは主に以下の3つの手順からなります。

* フィクスチャに対応するテーブルに含まれている既存のデータをすべて削除する
* フィクスチャのデータをテーブルに読み込む
* フィクスチャに直接アクセスしたい場合はフィクスチャのデータを変数にダンプする

#### フィクスチャはActive Recordオブジェクト

フィクスチャは、実はActive Recordのインスタンスです。前述の3番目の手順で示したように、フィクスチャはテストケースのローカル変数を自動的に設定してくれるので、フィクスチャのオブジェクトに直接アクセスできます。以下に例を示します。

```ruby
# davidという名前のフィクスチャに対応するUserオブジェクトを返す
users(:david)

# idで呼び出されたdavidのプロパティを返す
users(:david).id

# Userクラスで利用可能なメソッドにアクセスすることもできる
email(david.girlfriend.email, david.location_tonight)
```

Unit Testing your Models
------------------------

Railsにおけるユニットテストとは、モデルをテストするために書いたコードを指します。

For this guide we will be using Rails _scaffolding_. It will create the model, a migration, controller and views for the new resource in a single operation. It will also create a full test suite following Rails best practices. We will be using examples from this generated code and will be supplementing it with additional examples where necessary.

NOTE: Railsの _scaffold_ の詳細については、[Railsをはじめよう](getting_started.html)を参照してください。

When you use `rails generate scaffold`, for a resource among other things it creates a test stub in the `test/models` folder:

```bash
$ bin/rails generate scaffold article title:string body:text
...
create  app/models/article.rb
create  test/models/article_test.rb
create  test/fixtures/articles.yml
...
```

The default test stub in `test/models/article_test.rb` looks like this:

```ruby
require 'test_helper'

class ArticleTest < ActiveSupport::TestCase
  # test "the truth" do
  #   assert true
  # end
end
```

A line by line examination of this file will help get you oriented to Rails testing code and terminology.

```ruby
require 'test_helper'
```

As you know by now, `test_helper.rb` specifies the default configuration to run our tests. This is included with all the tests, so any methods added to this file are available to all your tests.

```ruby
class ArticleTest < ActiveSupport::TestCase
```

The `ArticleTest` class defines a _test case_ because it inherits from `ActiveSupport::TestCase`. `ArticleTest` thus has all the methods available from `ActiveSupport::TestCase`. You'll see those methods a little later in this guide.

Any method defined within a class inherited from `Minitest::Test`
(which is the superclass of `ActiveSupport::TestCase`) that begins with `test_` (case sensitive) is simply called a test. So, `test_password` and `test_valid_password` are legal test names and are run automatically when the test case is run.

Rails adds a `test` method that takes a test name and a block. It 上を実行すると以下が生成されます。 a normal `Minitest::Unit` test with method names prefixed with `test_`. So,

```ruby
test "the truth" do
  assert true
end
```

acts as if you had written

```ruby
def test_the_truth
  assert true
end
```

only the `test` macro allows a more readable test name. You can still use regular method definitions though.

NOTE: テスト名からのメソッド名生成は、スペースをアンダースコアに置き換えることによって行われます。生成されたメソッド名はRubyの正規な識別子である必要はありません。テスト名にパンクチュエーション（句読点）などの文字が含まれていても大丈夫です。これが可能なのは、Rubyではメソッド名にどんな文字列でも使用できるようになっているからです。普通でない文字を使おうとすると`define_method`呼び出しや`send`呼び出しが必要になりますが、名前の付け方そのものには公式な制限はありません。

```ruby
assert true
```

This line of code is called an _assertion_. An assertion is a line of code that evaluates an object (or expression) for expected results. 以下に例を示します。 an assertion can check:

* does this value = that value?
* is this object nil?
* does this line of code throw an exception?
* is the user's password greater than 5 characters?

Every test contains one or more assertions. Only when all the assertions are successful will the test pass.

### Maintaining the test database schema

In order to run your tests, your test database will need to have the current
structure. The test helper checks whether your test database has any pending
migrations. If so, it will try to load your `db/schema.rb` or `db/structure.sql`
into the test database. If migrations are still pending, an error will be
raised. Usually this indicates that your schema is not fully migrated. Running
the migrations against the development database (`bin/rake db:migrate`) will
bring the schema up to date.

NOTE: If existing migrations required modifications, the test database needs to
be rebuilt. This can be done by executing `bin/rake db:test:prepare`.

### Running Tests

Running a test is as simple as invoking the file containing the test cases through `rake test` command.

```bash
$ bin/rake test test/models/article_test.rb
.

Finished tests in 0.009262s, 107.9680 tests/s, 107.9680 assertions/s.

1 tests, 1 assertions, 0 failures, 0 errors, 0 skips
```

You can also run a particular test method from the test case by running the test and providing the `test method name`.

```bash
$ bin/rake test test/models/article_test.rb test_the_truth
.

Finished tests in 0.009064s, 110.3266 tests/s, 110.3266 assertions/s.

1 tests, 1 assertions, 0 failures, 0 errors, 0 skips
```

This will run all test methods from the test case. Note that `test_helper.rb` is in the `test` directory, hence this directory needs to be added to the load path using the `-I` switch.

The `.` (dot) above indicates a passing test. When a test fails you see an `F`; when a test throws an error you see an `E` in its place. The last line of the output is the summary.

To see how a test failure is reported, you can add a failing test to the `article_test.rb` test case.

```ruby
test "should not save article without title" do
  article = Article.new
  assert_not article.save
end
```

それでは、新しく追加したテストを実行してみましょう。

```bash
$ bin/rake test test/models/article_test.rb test_should_not_save_article_without_title
F

Finished tests in 0.044632s, 22.4054 tests/s, 22.4054 assertions/s.

  1) Failure:
test_should_not_save_article_without_title(ArticleTest) [test/models/article_test.rb:6]:
Failed assertion, no message given.

1 tests, 1 assertions, 1 failures, 0 errors, 0 skips
```

出力に含まれている単独の文字`F`は失敗を表します。`1)`の後にこの失敗に対応するトレースが、失敗したテスト名とともに表示されています。次の数行はスタックトレースで、アサーションの実際の値と期待されていた値がその後に表示されています。デフォルトのアサーションメッセージには、エラー箇所を特定するのに十分な情報が含まれています。アサーションメッセージをさらに読みやすくするために、すべてのアサーションに以下のようにメッセージをオプションパラメータを渡すことができます。

```ruby
test "should not save article without title" do
  article = Article.new
  assert_not article.save, "Saved the article without a title"
end
```

テストを実行すると、以下のようにさらに読みやすいメッセージが表示されます。

```bash
  1) Failure:
test_should_not_save_article_without_title(ArticleTest) [test/models/article_test.rb:6]:
Saved the article without a title
```

今度は _title_ フィールドに対してモデルのレベルでバリデーションを行い、テストがパスするようにしてみましょう。

```ruby
class Article < ActiveRecord::Base
  validates :title, presence: true
end
```

このテストはパスするはずです。もう一度テストを実行してみましょう。

```bash
$ bin/rake test test/models/article_test.rb test_should_not_save_article_without_title
.

Finished tests in 0.047721s, 20.9551 tests/s, 20.9551 assertions/s.

1 tests, 1 assertions, 0 failures, 0 errors, 0 skips
```

お気付きになった方もいるかと思いますが、私たちは欲しい機能が未実装であるために失敗するテストをあえて最初に作成していることにご注目ください。続いてその機能を実装し、それからもう一度実行してテストがパスすることを確認しました。ソフトウェア開発の世界ではこのようなアプローチをテスト駆動開発 ( _Test-Driven Development_ : TDD) と呼んでいます。

TIP: Rails開発者の多くがTDDを日々実践しています。この手法は、アプリケーションのあらゆる部品に対して動作試験を行なうためのテストスイートを作成する方法として非常に優れています。TDDそのものについては本ガイドの範疇を超えるためこれ以上言及しませんが、TDDを初めて学ぶための資料のひとつとして[Railsアプリケーションを開発するためのTDD 15ステップ](http://andrzejonsoftware.blogspot.com/2007/05/15-tdd-steps-to-create-rails.html)をご紹介します。

エラーがどのように表示されるかを確認するために、以下のようなエラーを含んだテストを作ってみましょう。

```ruby
test "should report error" do
  # some_undefined_variable is not defined elsewhere in the test case
  some_undefined_variable
  assert true
end
```

これで、このテストを実行するとさらに多くのメッセージがコンソールに表示されるようになりました。

```bash
$ bin/rake test test/models/article_test.rb test_should_report_error
E

Finished tests in 0.030974s, 32.2851 tests/s, 0.0000 assertions/s.

  1) Error:
test_should_report_error(ArticleTest):
NameError: undefined local variable or method `some_undefined_variable' for #<ArticleTest:0x007fe32e24afe0>
    test/models/article_test.rb:10:in `block in <class:ArticleTest>'

1 tests, 0 assertions, 0 failures, 1 errors, 0 skips
```

今度は'E'が出力されます。これはエラーが発生したテストが1つあることを示しています。

NOTE: テストスイートに含まれる各テストメソッドは、エラーまたはアサーション失敗が発生するとそこで実行を中止し、次のメソッドに進みます。すべてのテストメソッドはアルファベット順に実行されます。

テストが失敗すると、それに応じたバックトレースが出力されます。Railsはデフォルトでバックトレースをフィルタし、アプリケーションに関連するバックトレースのみを出力します。これによって、フレームワークから発生する不要な情報を排除して作成中のコードに集中できます。完全なバックトレースを参照しなければならなくなった場合は、`BACKTRACE`環境変数を設定するだけで動作を変更できます。

```bash
$ BACKTRACE=1 bin/rake test test/models/article_test.rb
```

### 単体テストに含めるべき項目

テストは、不具合が生じる可能性のあるあらゆる箇所に対して1つずつ含めることができれば理想的です。少なくとも、1つのバリデーションに対して1つ以上のテスト、1つのモデルに対して1つ以上のテストを作成することをお勧めします。

### 利用可能なアサーション

ここまでにいくつかのアサーションをご紹介しましたが、これらはごく一部に過ぎません。アサーションこそは、テストの中心を担う重要な存在です。システムが計画通りに動作していることを実際に確認しているのはアサーションです。

アサーションは非常に多くの種類が使用できるようになっています。
以下で紹介するのは、[`Minitest`](https://github.com/seattlerb/minitest)で使用できるアサーションからの抜粋です。MinitestはRailsにデフォルトで組み込まれているテスティングライブラリです。`[msg]`パラメータは1つのオプション文字列メッセージであり、テストが失敗したときのメッセージをわかりやすくするにはここで指定します。これは必須ではありません。

| アサーション                                                        | 目的 |
| ---------------------------------------------------------------- | ------- |
| `assert( test, [msg] )`                                          | `test`はtrueであると主張する。|
| `assert_not( test, [msg] )`                                      | `test`はfalseであると主張する。|
| `assert_equal( expected, actual, [msg] )`                        | `expected == actual`はtrueであると主張する。|
| `assert_not_equal( expected, actual, [msg] )`                    | `expected != actual`はtrueであると主張する。|
| `assert_same( expected, actual, [msg] )`                         | `expected.equal?(actual)`はtrueであると主張する。|
| `assert_not_same( expected, actual, [msg] )`                     | `expected.equal?(actual)`はfalseであると主張する。|
| `assert_nil( obj, [msg] )`                                       | `obj.nil?`はtrueであると主張する。|
| `assert_not_nil( obj, [msg] )`                                   | `obj.nil?`はfalseであると主張する。|
| `assert_empty( obj, [msg] )`                                     | `obj`は`empty?`であると主張する。|
| `assert_not_empty( obj, [msg] )`                                 | `obj`は`empty?`ではないと主張する。|
| `assert_match( regexp, string, [msg] )`                          | stringは正規表現 (regexp) にマッチすると主張する。|
| `assert_no_match( regexp, string, [msg] )`                       | stringは正規表現 (regexp) にマッチしないと主張する。|
| `assert_includes( collection, obj, [msg] )`                      | `obj`は`collection`に含まれると主張する。|
| `assert_not_includes( collection, obj, [msg] )`                  | `obj`は`collection`に含まれないと主張する。|
| `assert_in_delta( expecting, actual, [delta], [msg] )`           | `expected`の個数と`actual`の個数の差分は`delta`以内であると主張する。|
| `assert_not_in_delta( expecting, actual, [delta], [msg] )`       | `expected`の個数と`actual`の個数の差分は`delta`以内にはないと主張する。|
| `assert_throws( symbol, [msg] ) { block }`                       | 与えられたブロックはシンボルをスローすると主張する。|
| `assert_raises( exception1, exception2, ... ) { block }`         | 渡されたブロックから、渡された例外のいずれかが発生すると主張する。|
| `assert_nothing_raised( exception1, exception2, ... ) { block }` | 渡されたブロックからは、渡されたどの例外も発生しないと主張する。|
| `assert_instance_of( class, obj, [msg] )`                        | `obj`は`class`のインスタンスであると主張する。|
| `assert_not_instance_of( class, obj, [msg] )`                    | `obj`は`class`のインスタンスではないと主張する。|
| `assert_kind_of( class, obj, [msg] )`                            | `obj`は`class`またはそのサブクラスのインスタンスであると主張する。|
| `assert_not_kind_of( class, obj, [msg] )`                        | `obj`は`class`またはそのサブクラスのインスタンスではないと主張する。|
| `assert_respond_to( obj, symbol, [msg] )`                        | `obj`は`symbol`に応答すると主張する。|
| `assert_not_respond_to( obj, symbol, [msg] )`                    | `obj`は`symbol`に応答しないと主張する|
| `assert_operator( obj1, operator, [obj2], [msg] )`               | `obj1.operator(obj2)`はtrueであると主張する。|
| `assert_not_operator( obj1, operator, [obj2], [msg] )`           | `obj1.operator(obj2)`はfalseであると主張する。|
| `assert_predicate ( obj, predicate, [msg] )`                     | `obj.predicate`はtrueであると主張する (例:`assert_predicate str, :empty?`)。|
| `assert_not_predicate ( obj, predicate, [msg] )`                 | `obj.predicate`はfalseであると主張する(例:`assert_not_predicate str, :empty?`)。|
| `assert_send( array, [msg] )`                                    | `array[0]`のオブジェクトがレシーバ、`array[1]`がメソッド、`array[2以降]`がパラメータである場合の実行結果はtrueであると主張する。(これは奇妙ではないか?)|
| `flunk( [msg] )`                                                 | 必ず失敗すると主張する。これはテストが未完成であることを示すのに便利。|

これらはMinitestがサポートするアサーションの一部に過ぎません。最新の完全なアサーションのリストについては[Minitest APIドキュメント](http://docs.seattlerb.org/minitest/)、特に[`Minitest::Assertions`](http://docs.seattlerb.org/minitest/Minitest/Assertions.html)を参照してください。

テスティングフレームワークはモジュール化されているので、アサーションを自作して利用することもできます。実際、Railsはまさにそれを行っているのです。Railsには開発を楽にしてくれる特殊なアサーションがいくつも追加されています。

NOTE: アサーションの自作は高度なトピックなので、このチュートリアルでは扱いません。

### Rails固有のアサーション

Railsは`minitest`フレームワークに以下のような独自のカスタムアサーションを追加しています。

| アサーション                                                                         | 目的 |
| --------------------------------------------------------------------------------- | ------- |
| `assert_difference(expressions, difference = 1, message = nil) {...}`             | yieldされたブロックで評価された結果である式の戻り値における数値の違いをテストする。|
| `assert_no_difference(expressions, message = nil, &amp;block)`                    | 式を評価した結果の数値は、ブロックで渡されたものを呼び出す前と呼び出した後で違いがないと主張する。|
| `assert_recognizes(expected_options, path, extras={}, message=nil)`               | [REVIEW]渡されたパスのルーティングが正しく扱われ、(expected_optionsハッシュで渡された) 解析オプションがパスと一致したことを主張する。基本的にこのアサーションでは、Railsはexpected_optionsで渡されたルーティングを認識すると主張する。|
| `assert_generates(expected_path, options, defaults={}, extras = {}, message=nil)` | 渡されたオプションは、渡されたパスの生成に使用できるものであると主張する。assert_recognizesと逆の動作。extrasパラメータは、クエリ文字列に追加リクエストがある場合にそのパラメータの名前と値をリクエストに渡すのに使用される。messageパラメータはアサーションが失敗した場合のカスタムエラーメッセージを渡すことができる。|
| `assert_response(type, message = nil)`                                            | レスポンスが特定のステータスコードを持っていることを主張する。`:success`を指定するとステータスコード200-299を指定したことになり、同様に`:redirect`は300-399、`:missing`は404、`:error`は500-599にそれぞれマッチする。ステータスコードの数字や同等のシンボルを直接渡すこともできる。詳細については[ステータスコードの完全なリスト](http://rubydoc.info/github/rack/rack/master/Rack/Utils#HTTP_STATUS_CODES-constant)および[シンボルとステータスコードの対応リスト](http://rubydoc.info/github/rack/rack/master/Rack/Utils#SYMBOL_TO_STATUS_CODE-constant)を参照のこと。|
| `assert_redirected_to(options = {}, message=nil)`                                 | 渡されたリダイレクトオプションが、最後に実行されたアクションで呼び出されたリダイレクトのオプションと一致することを主張する。このアサーションは部分マッチ可能。たとえば`assert_redirected_to(controller: "weblog")`は`redirect_to(controller: "weblog", action: "show")`というリダイレクトなどにもマッチする。`assert_redirected_to root_path`などの名前付きルートを渡したり、`assert_redirected_to @article`などのActive Recordオブジェクトを渡すこともできる。|
| `assert_template(expected = nil, message=nil)`                                    | Asserts that the request was rendered with the appropriate template file.|

You'll see the usage of some of these assertions in the next chapter.

Functional Tests for Your Controllers
-------------------------------------

Railsで1つのコントローラに含まれる複数のアクションをテストするには、コントローラに対する機能テスト (functional test) を作成します。Controllers handle the incoming web requests to your application and eventually respond with a rendered view.

### What to Include in your Functional Tests

You should test for things such as:

* was the web request successful?
* was the user redirected to the right page?
* was the user successfully authenticated?
* was the correct object stored in the response template?
* was the appropriate message displayed to the user in the view?

Now that we have used Rails scaffold generator for our `Article` resource, it has already created the controller code and tests. You can take look at the file `articles_controller_test.rb` in the `test/controllers` directory.

Let me take you through one such test, `test_should_get_index` from the file `articles_controller_test.rb`.

```ruby
class ArticlesControllerTest < ActionController::TestCase
  test "should get index" do
  get :index
    assert_response :success
    assert_not_nil assigns(:articles)
  end
end
```

In the `test_should_get_index` test, Rails simulates a request on the action called `index`, making sure the request was successful and also ensuring that it assigns a valid `articles` instance variable.

The `get` method kicks off the web request and populates the results into the response. It accepts 4 arguments:

* The action of the controller you are requesting. This can be in the form of a string or a symbol.
* アクションに渡すリクエストパラメータに含まれるオプションハッシュ1つ (クエリ文字列パラメータや記事の変数など)。
* リクエストで渡されるセッション変数のオプションハッシュ1つ。
* flashメッセージの値のオプションハッシュ1つ。

例: `:show`アクションを呼び出し、`id`に12を指定して`params`として渡し、セッションの`user_id`に5を設定する。

```ruby
get(:show, {'id' => "12"}, {'user_id' => 5})
```

別の例: `:view`アクションを呼び出し、`id`に12を指定して`params`として渡すが、セッションの代りにflashメッセージを1つ使用する。

```ruby
get(:view, {'id' => '12'}, nil, {'message' => 'booya!'})
```

NOTE: `articles_controller_test.rb`ファイルにある`test_should_create_article`テストを実行してみると、モデルレベルのバリデーションが新たに追加されることによってテストは失敗します。

`articles_controller_test.rb`ファイルの`test_should_create_article`テストを変更して、テストがパスするようにしてみましょう。

```ruby
test "should create article" do
  assert_difference('Article.count') do
    post :create, article: {title: 'Some title'}
  end

  assert_redirected_to article_path(assigns(:article))
end
```

これで、すべてのテストを実行するとパスするようになったはずです。

### 機能テストで利用できるHTTPリクエストの種類

HTTPリクエストに精通していれば、If you're familiar with the HTTP protocol, you'll know that `get` is a type of request. There are 6 request types supported in Rails functional tests:

* `get`
* `post`
* `patch`
* `put`
* `head`
* `delete`

これらはすべてメソッドとして利用できますが、実際には最初の2つでほとんどの用が足りるはずです。

NOTE: 機能テストは、そのリクエストがアクションで受け付けられるかどうかについては検証するものではありません。機能テストでこれらのリクエストの名前が使用されているのは、リクエストの種類を明示してテストを読みやすくするためです。

### 4つのハッシュ

6種類のメソッドのうち、`get`や`post`などいずれかのリクエストが行われて処理されると、以下の4種類のハッシュオブジェクトがしようできるようになります

* `assigns` - ビューで使用するためにアクションのインスタンス変数として保存されるすべてのオブジェクト。
* `cookies` - 設定されているすべてのcookies。
* `flash` - flash内のすべてのオブジェクト。
* `session` - セッション変数に含まれるすべてのオブジェクト。

これらのハッシュは、通常のHashオブジェクトと同様に文字列をキーとして値を参照できます。シンボル名による参照も可能です (ただし`assigns`は除く)。以下に例を示します。

```ruby
flash["gordon"]               flash[:gordon]
session["shmession"]          session[:shmession]
cookies["are_good_for_u"]     cookies[:are_good_for_u]

# Because you can't use assigns[:something] for historical reasons:
assigns["something"]          assigns(:something)
```

### 利用可能なインスタンス変数

機能テストでは以下の3つの専用インスタンス変数を使用できます。

* `@controller` - リクエストを処理するコントローラ
* `@request` - リクエスト
* `@response` - レスポンス

### HTTPとヘッダーとCGI変数を設定する

[HTTPヘッダー](http://tools.ietf.org/search/rfc2616#section-5.3)と[CGI変数](http://tools.ietf.org/search/rfc3875#section-4.1)は`@request`インスタンス変数で直接設定できます。

```ruby
# HTTPヘッダーを設定する
@request.headers["Accept"] = "text/plain, text/html"
get :index # ヘッダーをカスタマイズしたリクエストをシミュレートする

# CGI変数を設定する
@request.headers["HTTP_REFERER"] = "http://example.com/home"
post :create # 環境変数をカスタマイズしたリクエストをシミュレートする
```

### テンプレートとレイアウトをテストする

レスポンスが出力するテンプレートとレイアウトが正しいかどうかを確認したい場合は、`assert_template`メソッドを使用することができます。
method:

```ruby
test "index should render correct template and layout" do
  get :index
  assert_template :index
  assert_template layout: "layouts/application"
end
```

テンプレートとレイアウトのテストは、一回の`assert_template`呼び出しで同時に行なうことはできない点にご注意ください。さらに`layout`のテストでは通常の文字列に代えて正規表現を与えることもできますが、文字列を使用する方がテストの内容が明確になります。一方、テストするレイアウトを指定する際には必ずレイアウトが置かれているディレクトリ名もパスに含めなければなりません。これはRails標準の"layouts"ディレクトリを使用している場合であっても必要です。Hence,

```ruby
assert_template layout: "application"
```

上のコードは期待どおりに動作しません。

ビューでパーシャル (部分レイアウト) が使用されている場合は、レイアウトに対するアサーションを行なう際に必ずパーシャルにもアサーションを行なう必要があります。このとおりにしなかった場合、アサーションは失敗します。

Hence:

```ruby
test "new should render correct layout" do
  get :new
  assert_template layout: "layouts/application", partial: "_form"
end
```

上のコードでは`_form`というパーシャルを使用しているレイアウトに対して正しいアサーションが行われています。`assert_template`で`:partial`キーを省略してしまうと期待どおりに動作しません。

### 完全な機能テストの例

`flash`、`assert_redirected_to`、`assert_difference`を使用した別のテスト例を以下に示します。

```ruby
test "should create article" do
  assert_difference('Article.count') do
    post :create, article: {title: 'Hi', body: 'This is my first article.'}
  end
  assert_redirected_to article_path(assigns(:article))
  assert_equal 'Article was successfully created.', flash[:notice]
end
```

### ビューをテストする

アプリケーションのビューのテストで、あるページで重要なHTML要素とその内容がレスポンスに含まれていることを主張する (アサーションを行なう) のは、リクエストに対するレスポンスをテストする方法として便利です。`assert_select`というアサーションを使用すると、こうしたテストで簡潔かつ強力な文法を利用できるようになります。

NOTE: 他のドキュメントで`assert_tag`というアサーションを見かけることがあるかもしれません。このアサーションはRails 4.2で削除されました。今後は`assert_select`を使用してください。

`assert_select`には2つの書式があります。

`assert_select(セレクタ, [条件], [メッセージ])`という書式は、セレクタで指定された要素が条件に一致することを主張します。セレクタにはCSSセレクタの式 (文字列) や代入値を持つ式を使用できます。

`assert_select(要素, セレクタ, [条件], [メッセージ])` は、選択されたすべての要素が条件に一致することを主張します。選択される要素は、_element_ (`Nokogiri::XML::Node` or `Nokogiri::XML::NodeSet`のインスタンス) からその子孫要素までの範囲から選択されます。

たとえば、レスポンスに含まれるtitle要素の内容を検証するには、以下のアサーションを使用します。

```ruby
assert_select 'title', "Welcome to Rails Testing Guide" 
```

ネストした`assert_select`ブロックを使用することもできます。以下の例の場合、外側の`assert_select`で選択されたすべての要素の完全なコレクションに対して、内側の`assert_select`がアサーションを実行します。

```ruby
assert_select 'ul.navigation' do
  assert_select 'li.menu_item'
end
```

あるいは、外側の`assert_select`で選択された要素のコレクションをイテレート (列挙) し、`assert_select`が要素ごとに呼び出されるようにすることもできます。たとえば、レスポンスに2つの順序付きリストがあり、1つの順序付きリストにつき要素が4つあれば、以下のテストはどちらもパスします。

```ruby
assert_select "ol" do |elements|
  elements.each do |element|
    assert_select element, "li", 4
  end
end

assert_select "ol" do
  assert_select "li", 8
end
```

`assert_select`はきわめて強力なアサーションです。このアサーションの高度な利用法については[ドキュメント](https://github.com/rails/rails-dom-testing/blob/master/lib/rails/dom/testing/assertions/selector_assertions.rb)を参照してください。

#### その他のビューベースのアサーション

主にビューをテストするためのアサーションは他にもあります。

| アサーション                                                 | 目的 |
| --------------------------------------------------------- | ------- |
| `assert_select_email`                                     | メールの本文に対するアサーションを行なう。 |
| `assert_select_encoded`                                   | エンコードされたHTMLに対するアサーションを行なう。各要素の内容はデコードされた後にそれらをブロックとしてよびだす|
| `css_select(selector)`または`css_select(element, selector)` | _selector_で選択されたすべての要素を1つの配列にしたものを返す。2番目の書式については、最初に_element_がベース要素としてマッチし、続いてそのすべての子孫に対して_selector_のマッチを試みる。どちらの場合も、何も一致しなかった場合には空の配列を1つ返す。|

`assert_select_email`の利用例を以下に示します。

```ruby
assert_select_email do
  assert_select 'small', 'オプトアウトしたい場合は "購読停止" をクリックしてください。'
end
```

結合テスト
-------------------

結合テスト (integration test) は、複数のコントローラ同士のやりとりをテストします。一般に、アプリケーション内の重要なワークフローのテストに使用されます。

結合テストは他の単体テストや機能テストと異なり、アプリケーションの`test/integration`フォルダの下に明示的に作成する必要があります。Railsには結合テストのスケルトンを生成するためのジェネレータも用意されています。

```bash
$ bin/rails generate integration_test user_flows
      exists  test/integration/
      create  test/integration/user_flows_test.rb
```

生成直後の結合テストは以下のような内容になっています。

```ruby
require 'test_helper'

class UserFlowsTest < ActionDispatch::IntegrationTest
  # test "the truth" do
  #   assert true
  # end
end
```

結合テストは`ActionDispatch::IntegrationTest`から継承されます。これにより、結合テスト内でさまざまなヘルパーが利用できます。テストで使用するフィクスチャーも明示的に作成しておく必要がありまs

### 結合テストで使用できるヘルパー

標準のテスト用ヘルパーの他に、結合テストで利用できるヘルパーを以下に示します。

| ヘルパー                                                             | 目的 |
| ------------------------------------------------------------------ | ------- |
| `https?`                                                           | セッションがセキュアなHTTPSリクエストを模倣している場合に`true`を返す。|
| `https!`                                                           | セキュアなHTTPSリクエストを模倣できるようにする。|
| `host!`                                                            | 以後のリクエストでホスト名を設定できるようにする。|
| `redirect?`                                                        | 最後のリクエストがリダイレクトされた場合に`true`を返す。|
| `follow_redirect!`                                                 | 単一のリダイレクトレスポンスに従う。|
| `request_via_redirect(http_method, path, [parameters], [headers])` | HTTPリクエストを1つ作成し、以後のリダイレクトをすべて実行。|
| `post_via_redirect(path, [parameters], [headers])`                 | HTTP POSTリクエストを1つ作成し、以後のリダイレクトをすべて実行。|
| `get_via_redirect(path, [parameters], [headers])`                  | HTTP GETリクエストを1つ作成し、以後のリダイレクトをすべて実行。|
| `patch_via_redirect(path, [parameters], [headers])`                | HTTP PATCHリクエストを1つ作成し、以後のリダイレクトをすべて実行。|
| `put_via_redirect(path, [parameters], [headers])`                  | HTTP PUTリクエストを1つ作成し、以後のリダイレクトをすべて実行。|
| `delete_via_redirect(path, [parameters], [headers])`               | HTTP DELETEリクエストを1つ作成し、|
| `open_session`                                                     | 新しいセッションインスタンスを1つ開く。|

### 結合テストの例

複数のコントローラをたいしょう単純な結合テストは、以下のようになります。

```ruby
require 'test_helper'

class UserFlowsTest < ActionDispatch::IntegrationTest
  test "login and browse site" do
    # HTTPSでログイン
    https!
    get "/login"
    assert_response :success

    post_via_redirect "/login", username: users(:david).username, password: users(:david).password
    assert_equal '/welcome', path
    assert_equal 'Welcome david!', flash[:notice]

    https!(false)
    get "/articles/all"
    assert_response :success
    assert assigns(:articles)
  end
end
```

上のコード例で、結合テストに複数のコントローラが使用されていること、およびデータベースからディスパッチャまでスタック全体がテストされていることがおわかりいただけると思います。また、1つのテストの中で複数のセッションインスタンスを同時にオープンしたり、それらのインスタンスをアサーションメソッドで拡張することで、自分のアプリケーションだけに特化した非常に強力なテスティングDSLを作り出すこともできます。

結合テストで複数のセッションとカスタムDSLを使用した例を以下に示します。

```ruby
require 'test_helper'

class UserFlowsTest < ActionDispatch::IntegrationTest
  test "login and browse site" do
    # davidというユーザーがログイン
    david = login(:david)
    # ゲストユーザーがログイン
    guest = login(:guest)

    # どちらのユーザーも異なるセッションから利用できる
    assert_equal 'Welcome david!', david.flash[:notice]
    assert_equal 'Welcome guest!', guest.flash[:notice]

    # davidというユーザーはWebサイトをブラウズできる
    david.browses_site
    # ゲストユーザーもWebサイトをブラウズできる
    guest.browses_site

    # 他のアサーションに続く
  end

  private

    module CustomDsl
      def browses_site
        get "/products/all"
        assert_response :success
        assert assigns(:products)
      end
    end

    def login(user)
      open_session do |sess|
        sess.extend(CustomDsl)
        u = users(user)
        sess.https!
        sess.post "/login", username: u.username, password: u.password
        assert_equal '/welcome', sess.path
        sess.https!(false)
      end
    end
end
```

Rakeタスクでテストを実行する
---------------------------------

Rails comes with a number of built-in rake tasks to help with testing. The
table below lists the commands included in the default Rakefile when a Rails
project is created.

| タスク                   | 説明 |
| ----------------------- | ----------- |
| `rake test`             | Runs all tests in the `test` folder. Railsはデフォルトですべてのテストを実行するので、単に`rake`を実行するだけで済みます。
| `rake test:controllers` | `test/controllers`以下にあるすべてのコントローラ用テストを実行します。|
| `rake test:functionals` | `test/controllers`、`test/mailers`、`test/functional`以下にあるすべての機能テストをすべ実行します。|
| `rake test:helpers`     | `test/helpers`以下にあるすべてのヘルパーテストを実行します。 |
| `rake test:integration` | `test/integration`以下にあるすべての結合テストを実行します。|
| `rake test:jobs`        | Runs all the job tests from `test/jobs` |
| `rake test:mailers`     | `test/mailers`以下にあるすべてのメイラーテストを実行します。 |
| `rake test:models`      | `test/models`以下にあるすべてのモデルテストを実行します。 |
| `rake test:units`       | `test/models`、`test/helpers`、および`test/unit`以下にあるすべての単体テストを実行します。|
| `rake test:db`          | Runs all tests in the `test` folder and resets the db |


A Brief Note About Minitest
-----------------------------

Rubyには、テスティングを含むあらゆる一般的なユースケースのための膨大な標準ライブラリが付属しています。Ruby 1.9からはテスティングフレームワークとして`Minitest`が提供されるようになりました。前述した`assert_equal`などの基本的なアサーションは、実際にはすべて`Minitest::Assertions`で定義されています。テストクラスで継承している`ActiveSupport::TestCase`、`ActionController::TestCase`、`ActionMailer::TestCase`、`ActionView::TestCase`および`ActionDispatch::IntegrationTest`は`Minitest::Assertions`を含んでおり、これによってテスト内で基本的なアサーションがすべて利用できます。

NOTE: `Minitest`の詳細については[Minitest](http://ruby-doc.org/stdlib-2.1.0/libdoc/minitest/rdoc/MiniTest.html)を参照してください。

SetupとTeardown
------------------

各テストの実行前に特定のコードブロックを1つ実行したり、各テストの実行後に別のコードブロックを1つ実行したりしたい場合は、そのための特殊なコールバックを使用することができます。`Articles`コントローラの機能テストを例にとって詳しく見てみましょう。

```ruby
require 'test_helper'

class ArticlesControllerTest < ActionController::TestCase

  # 各テストが実行される直前に呼び出される
  def setup
    @article = articles(:one)
  end

  # 各テストの実行直後に呼び出される
  def teardown
    # @article変数は実際にはテストの実行のたびに直前で初期化されるので
    # ここで値をnilにする意味は実際にはないのですが、
    # teardownメソッドの動作を理解いただくためにこのようにしています
    @article = nil
  end

  test "should show article" do
    get :show, id: @article.id
    assert_response :success
  end

  test "should destroy article" do
    assert_difference('Article.count', -1) do
      delete :destroy, id: @article.id
    end

    assert_redirected_to articles_path
  end

end
```

上の`setup`メソッドは各テストの実行直前に呼び出されるので、`@article`変数はどのテストでも利用できるようになります。`setup`と`teardown`は`ActiveSupport::Callbacks`に実装されています。つまり、`setup`と`teardown`はテストの中で単なるメソッドとして使用する以外の利用法もあるということです。これらを使用する際に以下のものを指定することができます。

* ブロック1つ
* メソッド1つ (前述の例のとおり)
* シンボルで表されたメソッド名1つ
* ラムダ1つ

前述の`setup`コールバックで、メソッド名をシンボルで指定した場合の例を見てみましょう。

```ruby
require 'test_helper'

class ArticlesControllerTest < ActionController::TestCase

  # 各テストが実行される直前に呼び出される
  setup :initialize_article

  # 各テストの実行直後に呼び出される
  def teardown
    @article = nil
  end

  test "should show article" do
    get :show, id: @article.id
    assert_response :success
  end

  test "should update article" do
    patch :update, id: @article.id, article: {}
    assert_redirected_to article_path(assigns(:article))
  end

  test "should destroy article" do
    assert_difference('Article.count', -1) do
      delete :destroy, id: @article.id
    end

    assert_redirected_to articles_path
  end

  private

    def initialize_article
      @article = articles(:one)
    end
end
```

ルーティングをテストする
--------------

Railsアプリケーションの他の部分と同様、ルーティングもテストすることをお勧めします。Below are example tests for the routes of default `show` and `create` action of `Articles` controller above and it should look like:

```ruby
class ArticleRoutesTest < ActionController::TestCase
  test "should route to article" do
    assert_routing '/articles/1', { controller: "articles", action: "show", id: "1" }
  end

  test "should route to create article" do
    assert_routing({ method: 'post', path: '/articles' }, { controller: "articles", action: "create" })
  end
end
```

メイラーをテストする
--------------------

メイラークラスを十分にテストするためには特殊なツールが若干必要になります。

### メイラーのテストについて

Railsアプリケーションの他の部分と同様、メイラークラスについても期待どおり動作するかどうかをテストする必要があります。

メイラークラスをテストする目的は以下を確認することです。

* メールが処理 (作成および送信) されていること
* メールの内容 (subject、sender、bodyなど) が正しいこと
* 適切なメールが適切なタイミングで送信されていること

#### あらゆる側面からのチェック

メイラーのテストには単体テストと機能テストの2つの側面があります。単体テストでは、完全に制御された入力を与えた結果の出力と、期待される既知の値 (フィクスチャー) とを比較します。機能テストではメイラーによって作成される詳細部分についてのテストはほとんど行わず、コントローラとモデルがメイラーを正しく利用しているかどうかをテストするのがふつうですメイラーのテストは、最終的に適切なメールが適切なタイミングで送信されたことを立証するために行います。

### 単体テスト

メイラーが期待どおりに動作しているかどうかをテストするために、事前に作成しておいた出力例と、メイラーの実際の出力を比較するという単体テストを行なうことができます。

#### フィクスチャーの逆襲

メイラーの単体テストを行なうために、フィクスチャーを利用してメイラーが最終的に出力すべき外見の例を与えます。これらのフィクスチャーはメールの出力例であって、通常のフィクスチャーのようなActive Recordデータではないので、通常のフィクスチャーとは別の専用のサブディレクトリに保存します。`test/fixtures`ディレクトリの下のディレクトリ名は、メイラーの名前に対応させてください。たとえば`UserMailer`という名前のメイラーであれば、`test/fixtures/user_mailer`というスネークケースのディレクトリ名にします。

メイラーを生成すると、メイラーの各アクションに対応するスタブフィクスチャーが生成されます。Railsのジェネレータを使用しない場合は、自分でこれらのファイルを作成する必要があります。

#### 基本的なテストケース

`invite`というアクションで知人に招待状を送信する`UserMailer`という名前のメイラーに対する単体テストを以下に示します。これは、`invite`アクションをジェネレータで生成したときに作成される基本的なテストに手を加えたものです。

```ruby
require 'test_helper'

class UserMailerTest < ActionMailer::TestCase
  test "invite" do
    # メールを送信後キューに追加されるかどうかをテスト
    email = UserMailer.create_invite('me@example.com',
                                     'friend@example.com', Time.now).deliver_now
    assert_not ActionMailer::Base.deliveries.empty?

    # 送信されたメールの本文が期待どおりの内容であるかどうかをテスト
    assert_equal ['me@example.com'], email.from
    assert_equal ['friend@example.com'], email.to
    assert_equal 'You have been invited by me@example.com', email.subject
    assert_equal read_fixture('invite').join, email.body.to_s
  end
end
```

このテストでは、メールを送信し、その結果返されたオブジェクトを`email`変数に保存します。続いて、このメールが送信されたことを主張します (最初のアサーション)。次のアサーションでは、メールの内容が期待どおりであることを主張します。`read_fixture`ヘルパーを使用してこのファイルの内容を読みだしています。

`invite`フィクスチャーは以下のような内容にしておきます。

```
friend@example.comさん、こんにちは。

招待状を送付いたします。

どうぞよろしく!
```

ここでメイラーのテスト作成方法の詳細部分についてご説明したいと思います。`config/environments/test.rb`の`ActionMailer::Base.delivery_method = :test`という行で送信モードをtestに設定しています。これにより、送信したメールが実際に配信されないようにできます。そうしないと、テスト中にユーザーにスパムメールを送りつけてしまうことになります。この設定で送信したメールは、`ActionMailer::Base.deliveries`という配列についかされまs

NOTE: この`ActionMailer::Base.deliveries`という配列は、`ActionMailer::TestCase`でのテストを除き、自動的にはリセットされません。Action Mailerテストの外で配列をクリアしたい場合は、`ActionMailer::Base.deliveries.clear`で手動リセットできます。

### 機能テスト

メイラーの機能テストでは、メール本文や受取人が正しいことを確認するなど、単体テストでカバーされているようなことは扱いません。メールの機能テストでは、メール配信メソッドを呼び出し、その結果適切なメールが配信リストに追加されるかどうかをチェックします。機能テストでは配信メソッド自体は正常に動作すると仮定することになりますが、これでまず問題ありません。機能テストでは、期待されたタイミングでアプリケーションのビジネスロジックからメールが送信されるかどうかをテストすることがメインになるのが普通だからです。たとえば、友人を招待するという操作によってメールが適切に送信されるかどうかをチェックするには以下のような機能テストを使用します。

```ruby
require 'test_helper'

class UserControllerTest < ActionController::TestCase
  test "invite friend" do
    assert_difference 'ActionMailer::Base.deliveries.size', +1 do
      post :invite_friend, email: 'friend@example.com'
    end
    invite_email = ActionMailer::Base.deliveries.last

    assert_equal "You have been invited by me@example.com", invite_email.subject
    assert_equal 'friend@example.com', invite_email.to[0]
    assert_match(/Hi friend@example.com/, invite_email.body.to_s)
  end
end
```

ヘルパーをテストする
---------------

ヘルパーのテストについては、ヘルパーメソッドの出力が期待どおりであるかどうかをチェックするだけで十分です。ヘルパー関連のテストは`test/helpers`ディレクトリに置かれます。

ヘルパーテストの外枠は以下のような感じです。

```ruby
require 'test_helper'

class UserHelperTest < ActionView::TestCase
end
```

ヘルパー自体は単なるモジュールであり、ビューから利用するヘルパーメソッドをこの中に定義します。ヘルパーメソッドの出力をテストするには、以下のようなミックスインを使用する必要があるでしょう。

```ruby
class UserHelperTest < ActionView::TestCase
  include UserHelper

  test "should return the user name" do
    # ...
  end
end
```

さらに、テストクラスは`ActionView::TestCase`をextendしたものなので、`link_to`や`pluralize`などのRailsヘルパーメソッドにアクセスできます。

Testing Jobs
------------

Since your custom jobs can be queued at different levels inside your application,
you'll need to test both jobs themselves (their behavior when they get enqueued)
and that other entities correctly enqueue them.

### A Basic Test Case

By default, when you generate a job, an associated test will be generated as well
under the `test/jobs` directory. Here's an example test with a billing job:

```ruby
require 'test_helper'

class BillingJobTest < ActiveJob::TestCase
  test 'that account is charged' do
    BillingJob.perform_now(account, product)
    assert account.reload.charged_for?(product)
  end
end
```

This test is pretty simple and only asserts that the job get the work done
as expected.

By default, `ActiveJob::TestCase` will set the queue adapter to `:test` so that
your jobs are performed inline. It will also ensure that all previously performed
and enqueued jobs are cleared before any test run so you can safely assume that
no jobs have already been executed in the scope of each test.

### Custom Assertions And Testing Jobs Inside Other Components

Active Job ships with a bunch of custom assertions that can be used to lessen
the verbosity of tests:

| Assertion                              | Purpose |
| -------------------------------------- | ------- |
| `assert_enqueued_jobs(number)`         | Asserts that the number of enqueued jobs matches the given number. |
| `assert_performed_jobs(number)`        | Asserts that the number of performed jobs matches the given number. |
| `assert_no_enqueued_jobs { ... }`      | Asserts that no jobs have been enqueued. |
| `assert_no_performed_jobs { ... }`     | Asserts that no jobs have been performed. |
| `assert_enqueued_with([args]) { ... }` | Asserts that the job passed in the block has been enqueued with the given arguments. |
| `assert_performed_with([args]) { ... }`| Asserts that the job passed in the block has been performed with the given arguments. |

It's a good practice to ensure that your jobs correctly get enqueued or performed
wherever you invoke them (e.g. inside your controllers). This is precisely where
the custom assertions provided by Active Job are pretty useful. For instance,
within a model:

```ruby
require 'test_helper'

class ProductTest < ActiveJob::TestCase
  test 'billing job scheduling' do
    assert_enqueued_with(job: BillingJob) do
      product.charge(account)
    end
  end
end
```

その他のテスティングアプローチ
------------------------

Railsアプリケーションではビルトインの`minitest`ベースのテスティング以外のテストしか利用できないわけではありません。Rails開発者は以下のような実にさまざまなアプローチでテストの実行や支援を行っています。

* [NullDB](http://avdi.org/projects/nulldb/)はデータベースの利用を避けてテスティングを高速化する方法です。
* [Factory Girl](https://github.com/thoughtbot/factory_girl/tree/master)はフィクスチャーに代わるテストデータ提供/生成ツールです。
* [Fixture Builder](https://github.com/rdy/fixture_builder)はテスト実行直前にRubyのファクトリーをコンパイルしてフィクスチャーに変換するツールです。
* [MiniTest::Spec Rails](https://github.com/metaskills/minitest-spec-rails)、RailsのテストではMiniTest::Spec DSLを利用します。
* [Shoulda](http://www.thoughtbot.com/projects/shoulda)`test/unit`を拡張してさまざまなヘルパー/マクロ/アサーションを追加します。
* [RSpec](http://relishapp.com/rspec)はビヘイビア駆動開発用のフレームワークです。
* [Capybara](http://jnicklas.github.com/capybara/), Acceptance test framework for web applications