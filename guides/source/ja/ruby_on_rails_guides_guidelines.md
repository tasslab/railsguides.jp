Rails ガイドのガイドライン
===============================

本ガイドは、Ruby on Railsガイドを書くためのガイドラインです。本ガイド自身が本ガイドに従って書かれており、望ましいガイドラインの例であると同時に優美なループを形成しています。

このガイドの内容:

* Railsドキュメントの記法
* ガイドをローカルで生成する方法

--------------------------------------------------------------------------------

マークダウン (Markdown)
-------

Guides are written in [GitHub Flavored マークダウン (Markdown)](https://help.github.com/articles/github-flavored-markdown). There is comprehensive [documentation for マークダウン (Markdown)](http://daringfireball.net/projects/markdown/syntax), as well as a [cheatsheet](http://daringfireball.net/projects/markdown/basics).

Prologue
--------

ガイドの冒頭には、読者の開発意欲を高めるような文を置いてください。ガイドの青い部分がこれに該当します。プロローグでは、そのガイドの概要と、ガイドで学ぶ項目について記載してください。As an example, see the [Routing Guide](routing.html).

Headings
------

The title of every guide uses an `h1` heading; guide sections use `h2` headings; subsections use `h3` headings; etc. Note that the generated HTML output will use heading tags starting with `<h2>`.

```
ガイドのタイトル
===========

Section
-------

### サブセクション
```

When writing headings, capitalize all words except for prepositions, conjunctions, internal articles, and forms of the verb "to be":

```
#### Middlewareスタックは配列
#### オブジェクトが保存されるタイミング
```

Use the same inline formatting as regular text:

```
##### `:content_type`オプション
```

API ドキュメント作成ガイドライン
----------------------------

ガイドとAPIは、必要な箇所が互いに首尾一貫している必要があります。In particular, these sections of the [API ドキュメント作成ガイドライン](api_documentation_guidelines.html) also apply to the guides:

* [言葉遣い](api_documentation_guidelines.html#語調)
* [English](api_documentation_guidelines.html#english)
* [サンプルコード](api_documentation_guidelines.html#サンプルコード)
* [Filenames](api_documentation_guidelines.html#file-names)
* [フォント](api_documentation_guidelines.html#フォント)

HTMLガイド
-----------

ガイドを生成する前に、システムに最新のBundlerがインストールされていることを確認してください。現時点であれば、Bundler 1.3.5がインストールされている必要があります。

To install the latest version of Bundler, run `gem install bundler`.

### 生成

To generate all the guides, just `cd` into the `guides` directory, run `bundle install`, and execute:

```
bundle exec rake guides:generate
```

or

```
bundle exec rake guides:generate:html
```

`my_guide.md`ファイルだけを生成したい場合は環境変数`ONLY`に設定します。

```
touch my_guide.md
bundle exec rake guides:generate ONLY=my_guide
```

デフォルトでは、変更のないガイドは生成がスキップされるので、`ONLY`を使用する機会はそうないと思われます。

すべてのガイドを強制的に生成するには`ALL=1`を指定します。

生成の際には`WARNINGS=1`を指定しておくことをお勧めします。これにより、重複したIDが検出され、内部リンクが切れている場合に警告が出力されます。

英語以外の言語向けに生成を行いたい場合は、`source`ディレクトリの下にたとえば`source/es`のようにその言語用のディレクトリを作成し、`GUIDES_LANGUAGE`環境変数を設定します。

```
bundle exec rake guides:generate GUIDES_LANGUAGE=es
```

生成スクリプトの設定に使用できる環境変数をすべて知りたい場合は、単に以下を実行してください。

```
rake
```

### 検証

生成されたHTMLを検証するには以下を実行します。

```
bundle exec rake guides:validate
```

特に、タイトルを元にIDが生成される関係上、タイトルでの重複が生じやすくなっています。重複を検出するには、ガイド生成時に`WARNINGS=1`を指定してください。警告に解決方法が出力されます。

Kindleガイド
-------------

### 生成

Kindle向けにガイドを生成するには、以下のrakeタスクを実行します。To generate guides for the Kindle, use the following rake task:

```
bundle exec rake guides:generate:kindle
```