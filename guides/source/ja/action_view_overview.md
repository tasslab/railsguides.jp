Action View の概要
====================

このガイドの内容:

* Action Viewの概要とRailsでの利用法
* テンプレート、パーシャル(部分テンプレート)、レイアウトの最適な利用法
* Action Viewで提供されるヘルパーの紹介と、カスタムヘルパーの作成法
* ビューのローカライズ方法

--------------------------------------------------------------------------------

Action Viewについて
--------------------

Action ViewおよびAction Controllerは、Action Packを構成する2大要素です。Railsでは、WebリクエストはAction Packで取り扱われます。この動作はコントローラ寄りの部分 (ロジックの実行) とビュー寄りの部分(テンプレートの描画) に分かれます。Action Controllerは、データベースとのやりとりや、必要に応じたCRUD (Create/Read/Update/Delete) アクションの実行にかかわります。Action View はその後レスポンスを実際のWebページにまとめる役割を担います。

Action Viewのテンプレートは、HTMLタグの合間にERB (Embedded Ruby) を含む形式で書かれます。ビューテンプレートがコードの繰り返しでうずまって乱雑になるのを避けるために、フォーム・日付・文字列に対して共通の動作を提供するヘルパークラスが多数用意されています。アプリケーションの機能向上に応じて独自のヘルパーを追加することも簡単にできます。

NOTE: Action Viewの一部の機能はActive Recordと結びついていますが、Action ViewがActive Recordに依存しているわけではありません。Action Viewは独立したパッケージであり、どのようなRubyライブラリとでも組み合わせて使用できます。

Action ViewをRailsで使用する
----------------------------

アプリケーションの`app/views`ディレクトリには、1つのコントローラごとに1つのディレクトリが作成され、そこにビューテンプレートファイルが置かれます。このビューテンプレートはそのコントローラと関連付けられています。これらのファイルは、コントローラ内にあるアクションごとに出力された結果をビューで表示するために使用されます。

scaffoldを使用してリソースを生成するときに、Railsがデフォルトでどんなことを行なうのか見てみましょう。

```bash
$ bin/rails generate scaffold article
      [...]
      invoke  scaffold_controller
　create  app/controllers/articles_controller.rb
      invoke    erb
　create  app/views/articles
　create  app/views/articles/index.html.erb
　create  app/views/articles/edit.html.erb
　create  app/views/articles/show.html.erb
　create  app/views/articles/new.html.erb
　create  app/views/articles/_form.html.erb
      [...]
```

Railsのビューには命名規則があります。上で生成されたファイルを見るとわかるように、ビューテンプレートファイルは基本的にコントローラのアクションと関連付けられています。
以下に例を示します。 the index controller action of the `articles_controller.rb` will use the `index.html.erb` view file in the `app/views/articles` directory.
これらのERBファイルに、それらを内包するレイアウトテンプレートや、ビューから参照されるあらゆるパーシャル (部分テンプレート) が組み合わさって完全なHTMLが生成され、クライアントに送信されます。Within this guide you will find more detailed documentation about each of these three components.


テンプレート、パーシャル、レイアウト
-------------------------------

As mentioned, the final HTML output is a composition of three Rails elements: `Templates`, `Partials` and `Layouts`.
Below is a brief overview of each of them.

### テンプレート

Action Viewのテンプレートはさまざまな方法で記述することができます。If the template file has a `.erb` extension then it uses a mixture of ERB (Embedded Ruby) and HTML. If the template file has a `.builder` extension then the `Builder::XmlMarkup` library is used.

Railsでは複数のテンプレートシステムがサポートされており、テンプレートファイルの拡張子で区別されます。たとえば、ERBテンプレートシステムを使用するHTMLファイルの拡張子は`.html.erb`になります。

#### ERB

ERBテンプレートの内部では、`<% %>`タグや`<%= %>`タグにRubyコードを含めることができます。最初の`<% %>`タグはその中に書かれたRubyコードを実行しますが、実行結果は出力されません。条件文やループ、ブロックなど出力の不要な行はこのタグの中に書くとよいでしょう。次の`<%= %>`タグでは実行結果がWebページに出力されます。

以下は、名前を出力するためのループです。

```html+erb
<h1>Names of all the people</h1>
<% @people.each do |person| %>
  Name: <%= person.name %><br>
<% end %>
```

The loop is set up using regular embedding tags (`<% %>`) and the name is inserted using the output embedding tags (`<%= %>`). Note that this is not just a usage suggestion: regular output functions such as `print` and `puts` won't be rendered to the view with ERB templates. 以下のコードは誤りです。

```html+erb
<%# 間違い %>
Hi, Mr. <% puts "Frodo" %>
```

なお、Webページへの出力結果の最初と最後からホワイトスペースを取り除きたい場合は`<%-` および `-%>`を通常の`<%` および `%>`と交互にご使用ください (訳注: これは英語のようなスペース分かち書きを行なう言語向けのノウハウです)。

#### Builderてんぷれー

BuilderテンプレートはERBの代わりに使用できる、よりプログラミング向きな記法です。. They are especially useful for generating XML content. テンプレートの拡張子を`.builder`にすると、`xml`という名前のXmlMarkupオブジェクトが自動で使用できるようになります。

基本的な例を以下にいくつか示します。

```ruby
xml.em("emphasized")
xml.em { xml.b("emph & bold") }
xml.a("A Link", "href" => "http://rubyonrails.org")
xml.target("name" => "compile", "option" => "fast")
```

上のコードから以下が生成されます。

```html
<em>emphasized</em>
<em><b>emph &amp; bold</b></em>
<a href="http://rubyonrails.org">A link</a>
<target option="fast" name="compile" />
```

ブロックを後ろに伴うメソッドはすべて、ブロックの中にネストしたマークアップを含むXMLマークアップタグとして扱われます。以下の例で示します。

```ruby
xml.div {
  xml.h1(@person.name)
  xml.p(@person.bio)
}
```

上のコードの出力は以下のようなものになります。

```html
<div>
  <h1>David Heinemeier Hansson</h1>
  <p>A product of Danish Design during the Winter of '79...</p>
</div>
```

以下はBasecampで実際に使用されているRSS出力コードをそのまま引用したものです。

```ruby
xml.rss("version" => "2.0", "xmlns:dc" => "http://purl.org/dc/elements/1.1/") do
  xml.channel do
    xml.title(@feed_title)
    xml.link(@url)
    xml.description "Basecamp: Recent items"
    xml.language "en-us"
    xml.ttl "40"

    for item in @recent_items
      xml.item do
        xml.title(item_title(item))
        xml.description(item_description(item)) if item_description(item)
        xml.pubDate(item_pubDate(item))
        xml.guid(@person.firm.account.url + @recent_items.url(item))
        xml.link(@person.firm.account.url + @recent_items.url(item))
        xml.tag!("dc:creator", item.author_name) if item_has_creator?(item)
      end
    end
  end
end
```

#### テンプレートをキャッシュする

Railsは、デフォルトですべてのビューテンプレートをコンパイルしてメソッド化し、出力に備えます。developmentモードの場合、ビューテンプレートが変更されるとファイルの日付で変更が検出され、再度コンパイルされます。

### パーシャル

部分テンプレートは通常単にパーシャルと呼ばれます。パーシャルは、上とは異なる方法でレンダリング処理を扱いやすい単位に分割するためのしくみです。パーシャルを使用することで、ビュー内のコードをいくつものファイルに分割して書き出し、他のテンプレートでも使いまわすことができます。

#### パーシャルに名前を与える

パーシャルをビューの一部に含めて出力するには、ビュー内で`render`メソッドを使用します。

```erb
<%= render "menu" %>
```

上の呼び出しにより、`_menu.html.erb`という名前のファイルの内容が、renderメソッドを書いたその場所でレンダリングされます。パーシャルファイル名の冒頭にはアンダースコアが付いていることにご注意ください。これは通常のビューと区別するために付けられています。ただしrenderで呼び出す際にはこのアンダースコアは不要です。以下のように、他のフォルダの下にあるパーシャルを呼び出す際にもアンダースコアは不要です。

```erb
<%= render "shared/menu" %>
```

上のコードは、`app/views/shared/_menu.html.erb`パーシャルの内容をその場所でレンダリングします。

#### パーシャルを使用する to simplify Views

すぐに思い付くパーシャルの使い方といえば、パーシャルをサブルーチンと同等のものとみなすというのがあります。ビューの詳細部分をパーシャルに移動し、コードの見通しを良くするために、パーシャルを使うのです。以下に例を示します。 you might have a view that looks like this:

```html+erb
<%= render "shared/ad_banner" %>

<h1>Products</h1>

<p>Here are a few of our fine products:</p>
<% @products.each do |product| %>
  <%= render partial: "product", locals: {product: product} %>
<% end %>

<%= render "shared/footer" %>
```

上のコードの`_ad_banner.html.erb`パーシャルと`_footer.html.erb`パーシャルに含まれるコンテンツは、アプリケーションの多くのページと共有できます。あるページを開発中、パーシャルの部分については詳細を気にせずに済みます。

#### `as`と`object`オプション

`ActionView::Partials::PartialRenderer`は、デフォルトでテンプレートと同じ名前を持つローカル変数の中に自身のオブジェクトを持ちます。従って、以下のコードは

```erb
<%= render partial: "product" %>
```

within product we'll get `@product` in the local variable `product`, as if we had written:

```erb
<%= render partial: "product", locals: {product: @product} %>
```

With the `as` option we can specify a different name for the local variable. たとえば、ローカル変数名を`product`ではなく`item`にしたいのであれば、以下のようにします。

```erb
<%= render partial: "product", as: "item" %>
```

The `object` option can be used to directly specify which object is rendered into the partial; useful when the template's object is elsewhere (eg. in a different instance variable or in a local variable).

以下の例文ではyouが3度も使用されている。:

```erb
<%= render partial: "product", locals: {product: @item} %>
```

we would do:

```erb
<%= render partial: "product", object: @item %>
```

`object`オプションと`as`オプションは同時に使用することもできます。

```erb
<%= render partial: "product", object: @item, as: "item" %>
```

#### コレクションをレンダリングする

It is very common that a template will need to iterate over a collection and render a sub-template for each of the elements. This pattern has been implemented as a single method that accepts an array and renders a partial for each one of the elements in the array.

So this example for rendering all the products:

```erb
<% @products.each do |product| %>
  <%= render partial: "product", locals: { product: product } %>
<% end %>
```

can be rewritten in a single line:

```erb
<%= render partial: "product", collection: @products %>
```

When a partial is called with a collection, the individual instances of the partial have access to the member of the collection being rendered via a variable named after the partial. In this case, the partial is `_product`, and within it you can refer to `product` to get the collection member that is being rendered.

コレクション出力には短縮記法があります。`@products`が`Product`インスタンスのコレクションであれば、以下のコードでも同じ結果を得られます。

```erb
<%= render @products %>
```

Rails determines the name of the partial to use by looking at the model name in the collection, `Product` in this case. In fact, you can even render a collection made up of instances of different models using this shorthand, and Rails will choose the proper partial for each member of the collection.

#### スペーサーテンプレート

`:spacer_template`オプションを使用することで、メインパーシャルのインスタンスと交互にレンダリングされるセカンドパーシャルを指定することもできます。

```erb
<%= render partial: @products, spacer_template: "product_ruler" %>
```

Rails will render the `_product_ruler` partial (with no data passed to it) between each pair of `_product` partials.

### Layouts

Layouts can be used to render a common view template around the results of Rails controller actions. Typically, a Rails application will have a couple of layouts that pages will be rendered within. 以下に例を示します。 a site might have one layout for a logged in user and another for the marketing or sales side of the site. The logged in user layout might include top-level navigation that should be present across many controller actions. The sales layout for a SaaS app might include top-level navigation for things like "Pricing" and "Contact Us" pages. You would expect each layout to have a different look and feel. You can read about layouts in more detail in the [レイアウトとレンダリング](layouts_and_rendering.html) guide.

Partial Layouts
---------------

Partials can have their own layouts applied to them. These layouts are different from those applied to a controller action, but they work in a similar fashion.

Let's say we're displaying an article on a page which should be wrapped in a `div` for display purposes. Firstly, we'll create a new `Article`:

```ruby
Article.create(body: 'Partial Layouts are cool!')
```

In the `show` template, we'll render the `_article` partial wrapped in the `box` layout:

**articles/show.html.erb**

```erb
<%= render partial: 'article', layout: 'box', locals: {article: @article} %>
```

The `box` layout simply wraps the `_article` partial in a `div`:

**articles/_box.html.erb**

```html+erb
<div class='box'>
  <%= yield %>
</div>
```

The `_article` partial wraps the article's `body` in a `div` with the `id` of the article using the `div_for` helper:

**articles/_article.html.erb**

```html+erb
<%= div_for(article) do %>
  <p><%= article.body %></p>
<% end %>
```

this would output the following:

```html
<div class='box'>
  <div id='article_1'>
      <p>Partial Layouts are cool!</p>
  </div>
</div>
```

Note that the partial layout has access to the local `article` variable that was passed into the `render` call. However, unlike application-wide layouts, partial layouts still have the underscore prefix.

`yield`を呼び出す代わりに、パーシャルレイアウト内にあるコードのブロックを出力することもできます。以下に例を示します。 if we didn't have the `_article` partial, we could do this instead:

**articles/show.html.erb**

```html+erb
<% render(layout: 'box', locals: {article: @article}) do %>
  <%= div_for(article) do %>
      <p><%= article.body %></p>
  <% end %> <% end %>  ```

ここでは、同じ`_box`パーシャルを使用する前提であり、先の例と同じ出力が得られます。

ビューのパス
----------

(執筆予定)

Action Viewが提供するヘルパーの概要
-------------------------------------------

WIP: このリストにまだ含まれていないヘルパーがあります。完全なリストについては[APIドキュメント](http://api.rubyonrails.org/classes/ActionView/Helpers.html)を参照してください。

Action Viewで利用できるヘルパーの概要を以下に示します。[APIドキュメント](http://api.rubyonrails.org/classes/ActionView/Helpers.html) も参照して調べ直すことをお勧めします。APIドキュメントにはすべてのヘルパーの詳細が記載されており、本ガイドは概要を把握するためのものです。

### RecordTagHelper

このモジュールは、`div`などのコンテナタグを生成するメソッドを提供します。Active Recordオブジェクトを出力するためのコンテナ作成方法にはこれを使うことをお勧めします。この方法であれば、適切なクラスとid属性がコンテナに追加されるからです。これにより、これらのコンテナを通常の方法で簡単に参照でき、どのクラスやどのid属性を使用すべきかどうかを考えずに済みます。

#### content_tag_for

Active Recordオブジェクトに関連付けられるコンテナタグを出力します。

以下の例で考察してみましょう。 `@article` is the object of `Article` class, you can do:

```html+erb
<%= content_tag_for(:tr, @article) do %>
  <td><%= @article.title %></td>
<% end %>
```

上のコードによって以下のHTMLが生成されます。

```html
<tr id="article_1234" class="article">
  <td>Hello World!</td>
</tr>
```

オプションのハッシュを追加することで、HTML属性を指定することもできます。以下に例を示します。

```html+erb
<%= content_tag_for(:tr, @article, class: "frontpage") do %>
  <td><%= @article.title %></td>
<% end %>
```

上のコードによって以下のHTMLが生成されます。

```html
<tr id="article_1234" class="article frontpage">
  <td>Hello World!</td>
</tr>
```

Active Recordオブジェクトのコレクションを渡すこともできます。このメソッドはオブジェクトをループで回してそれぞれについてコンテナを作成しm以下の例で考察してみましょう。 `@articles` is an array of two `Article` objects:

```html+erb
<%= content_tag_for(:tr, @articles) do |article| %>
  <td><%= article.title %></td>
<% end %>
```

上のコードによって以下のHTMLが生成されます。

```html
<tr id="article_1234" class="article">
  <td>Hello World!</td>
</tr>
<tr id="article_1235" class="article">
  <td>Ruby on Rails Rocks!</td>
</tr>
```

#### div_for

このメソッドは内部で`content_tag_for`を呼び出して`:div`をタグ名にしてくれる、便利なメソッドです。Active Recordオブジェクトを単体またはコレクションとして渡すことができます。以下に例を示します。

```html+erb
<%= div_for(@article, class: "frontpage") do %>
  <td><%= @article.title %></td>
<% end %>
```

上のコードによって以下のHTMLが生成されます。

```html
<div id="article_1234" class="article frontpage">
  <td>Hello World!</td>
</div>
```

### AssetTagHelper

このモジュールは、画像・JavaScriptファイル・スタイルシート・フィードなどのアセットにビューをリンクするHTMLを生成するメソッドを提供します。

デフォルトでは、現在ホストされているpublicフォルダ内のアセットに対してリンクしますが、アプリケーション設定 (通常は`config/environments/production.rb`) の`config.action_controller.asset_host`で設定されているアセット用サーバーにリンクすることもできます。たとえば、`assets.example.com`というアセット専用ホストを使用したいとします。

```ruby
config.action_controller.asset_host = "assets.example.com"
image_tag("rails.png") # => <img src="http://assets.example.com/images/rails.png" alt="Rails" />
```

#### auto_discovery_link_tag

ブラウザやフィードリーダーが検出可能なRSSフィードやAtomフィードのリンクタグを返します。

```ruby
auto_discovery_link_tag(:rss, "http://www.example.com/feed.rss", { title: "RSS Feed" }) # =>
  <link rel="alternate" type="application/rss+xml" title="RSS Feed" href="http://www.example.com/feed.rss" />
```

#### image_path

`app/assets/images`に置かれている画像アセットへのパスを算出します。ドキュメントルート・ディレクトリからの完全なパスが返されます。このメソッドの内部では画像へのパス作成に`image_tag`が使用されています。

```ruby
image_path("edit.png") # => /assets/edit.png
```

config.assets.digestがtrueに設定されている場合、ファイル名にフィンガープリントが追加されます。

```ruby
image_path("edit.png") # => /assets/edit-2d1a2db63fc738690021fedb5a65b68e.png
```

#### image_url

`app/assets/images`に置かれている画像アセットへのURLを算出します。このメソッドは内部で`image_path`を呼び出しており、現在のホストまたはアセット用のホストとマージしてURLを生成します。

```ruby
image_url("edit.png") # => http://www.example.com/assets/edit.png
```

#### image_tag

Returns an HTML image tag for the source. 画像へのフルパス、または`app/assets/images`ディレクトリ内にあるファイルを引数として与えられます。

```ruby
image_tag("icon.png") # => <img src="/assets/icon.png" alt="Icon" />
```

#### javascript_include_tag

Returns an HTML script tag for each of the sources provided. `app/assets/javascripts`ディレクトリにあるJavaScriptファイル名 (拡張子`.js`はあってもなくても構いません) を引数として渡すことができます。この結果は現在のページにインクルードされます。ドキュメントルートからの相対完全パスを渡すこともできます。

```ruby
javascript_include_tag "common" # => <script src="/assets/common.js"></script>
```

アプリケーションでアセットパイプラインを使用せずにjQuery JavaScriptライブラリをインクルードする場合は、ソースとして`:defaults`を渡してください。`:defaults`を指定した場合、`app/assets/javascripts`ディレクトリに`application.js`というファイルがあればこれもインクルードされます。

```ruby
javascript_include_tag :defaults
```

ソースに`:all`を指定すると、`app/assets/javascripts`ディレクトリ以下にあるJavaScriptファイルをすべてインクルードできます。

```ruby
javascript_include_tag :all
```

複数のJavaScriptファイルをキャッシュして1つのファイルにすることができます。こうすることでJavaScriptファイルのダウンロードに必要なHTTP接続数を減らすことができ、速度が向上します。gzip圧縮すればさらに転送が速くなります。キャッシュが有効になるのは、`ActionController::Base.perform_caching`をtrueに設定した場合のみです。production環境ではデフォルトでtrueになりますが、development環境ではデフォルトではtrueになりません。

```ruby
javascript_include_tag :all, cache: true # =>
  <script src="/javascripts/all.js"></script>
```

#### javascript_path

`app/assets/javascripts`に置かれているJavaScriptアセットへのパスを算出します。ソースのファイル名に拡張子`.js`がない場合は自動的に補われます。ドキュメントルート・ディレクトリからの完全なパスが返されます。スクリプトパス作成のために内部で`javascript_include_tag`が使用されています。

```ruby
javascript_path "common" # => /assets/common.js
```

#### javascript_url

`app/assets/javascripts`に置かれているJavaScriptアセットへのURLを算出します。このメソッドは内部で`javascript_path`を呼び出しており、現在のホストまたはアセット用のホストとマージしてURLを生成します。

```ruby
javascript_url "common" # => http://www.example.com/assets/common.js
```

#### stylesheet_link_tag

引数として指定されたソースにあるスタイルシートへのリンクタグを返します。拡張子が指定されていない場合は、`.css`が自動的に補われます。

```ruby
stylesheet_link_tag "application" # => <link href="/assets/application.css" media="screen" rel="stylesheet" />
```

ソースに`:all`を指定すると、stylesheetディレクトリにあるすべてのスタイルシートを含めることができます。

```ruby
stylesheet_link_tag :all
```

複数のスタイルシートファイルをキャッシュして1つのファイルにすることができます。こうすることでスタイルシートファイルのダウンロードに必要なHTTP接続数を減らすことができ、速度が向上します。gzip圧縮すればさらに転送が速くなります。キャッシュが有効になるのは、`ActionController::Base.perform_caching`をtrueに設定した場合のみです。production環境ではデフォルトでtrueになりますが、development環境ではデフォルトではtrueになりません。

```ruby
stylesheet_link_tag :all, cache: true
# => <link href="/assets/all.css" media="screen" rel="stylesheet" />
```

#### stylesheet_path

`app/assets/stylesheets`に置かれているスタイルシートアセットへのパスを算出します。ソースのファイル名に拡張子`.css`がない場合は自動的に補われます。ドキュメントルート・ディレクトリからの完全なパスが返されます。このメソッドの内部ではスタイルシートへのパス作成に`stylesheet_link_tag`が使用されています。

```ruby
stylesheet_path "application" # => /assets/application.css
```

#### stylesheet_url

`app/assets/stylesheets`に置かれているスタイルシートアセットへのURLを算出します。このメソッドは内部で`stylesheet_path`を呼び出しており、現在のホストまたはアセット用のホストとマージしてURLを生成します。

```ruby
stylesheet_url "application" # => http://www.example.com/assets/application.css
```

### AtomFeedHelper

#### atom_feed

このヘルパーを使用して、Atomフィードを簡単に生成できます。以下にすべての使用例を示します。

**config/routes.rb**

```ruby
resources :articles
```

**app/controllers/articles_controller.rb**

```ruby
def index
  @articles = Article.all

  respond_to do |format|
    format.html
    format.atom
  end
end
```

**app/views/articles/index.atom.builder**

```ruby
atom_feed do |feed|
  feed.title("Articles Index")
  feed.updated((@articles.first.created_at))

  @articles.each do |article|
    feed.entry(article) do |entry|
      entry.title(article.title)
      entry.content(article.body, type: 'html')

      entry.author do |author|
        author.name(article.author_name)
      end
    end
  end
end
```

### BenchmarkHelper

#### benchmark

テンプレート内の1つのブロックの実行時間測定と、結果のログ出力に使用します。実行に時間のかかる行や、ボトルネックになる可能性のある行をこのブロックで囲み、実行にかかった時間を読み取ります。

```html+erb
<% benchmark "Process data files" do %>
  <%= expensive_files_operation %>
<% end %>
```

上のコードは、"Process data files (0.34523)"のようなログを出力します。このログは、コード最適化のためにタイミングを比較する際に役立てることができます。

### CacheHelper

#### cache

`cache`メソッドは、(アクション全体やページ全体ではなく) ビューの断片をキャッシュするメソッドです。この手法は、メニュー・ニュース記事・静的HTMLの断片などをキャッシュするのに便利です。このメソッドには、キャッシュしたいコンテンツを1つのブロックに含めて引数として渡します。詳細については、`ActionController::Caching::Fragments`を参照してください。

```erb
<% cache do %>
  <%= render "shared/footer" %>
<% end %>
```

### CaptureHelper

#### capture

`capture`メソッドを使用することで、テンプレートの一部を変数に保存することができます。保存された変数は、テンプレートやレイアウトのどんな場所でも自由に使用できます。

```html+erb
<% @greeting = capture do %>
  <p>Welcome! The date and time is <%= Time.now %></p>
<% end %>
```

上でキャプチャした変数は以下のように他の場所で自由に使用できます。

```html+erb
<html>
  <head>
    <title>Welcome!</title>
  </head>
  <body>
    <%= @greeting %>
  </body>
</html>
```

#### content_for

[REVIEW]`content_for`を呼び出すと、後の利用に備えて、idに対応するマークアップのブロックが保存されます。[REVIEW]以後、保存されたコンテンツを他のテンプレートやレイアウトで呼び出すことができます。呼び出しの際には、`yield`の引数となるidを渡します。

たとえば、あるRailsアプリケーション全体にわたって標準のアプリケーションレイアウトを使用しているが、特定のページでのみ特定のJavaScriptコードが必要となり、他のページではこのJavaScriptはまったく不要であるとします。このようなときには`content_for`を使用します。これにより、そのJavaScriptコードを特定のページにだけインクルードし、サイトの他の部分でインクルードされることのないようにできます。

**app/views/layouts/application.html.erb**

```html+erb
<html>
  <head>
    <title>Welcome!</title>
    <%= yield :special_script %>
  </head>
  <body>
    <p>Welcome! The date and time is <%= Time.now %></p>
  </body>
</html>
```

**app/views/articles/special.html.erb**

```html+erb
<p>This is a special page.</p>

<% content_for :special_script do %>
  <script>alert('Hello!')</script>
<% end %>
```

### DateHelper

#### date_select

日付用のselectタグのセットを返します。タグは年・月・日用にそれぞれあり、日付に関する特定の属性にアクセスして年月日を選択済みの状態にします。

```ruby
date_select("article", "published_on")
```

#### datetime_select

日付・時刻用のselectタグのセットを返します。タグは年・月・日・時・分用にそれぞれあり、日付・時刻に関する特定の属性にアクセスして日時が選択済みになります。

```ruby
datetime_select("article", "published_on")
```

#### distance_of_time_in_words

TimeオブジェクトやDateオブジェクト、秒を表す整数同士を比較して近似表現を返します。`include_seconds`をtrueにすると、より詳細な差を得られます。

```ruby
distance_of_time_in_words(Time.now, Time.now + 15.seconds)        # => less than a minute 
distance_of_time_in_words(Time.now, Time.now + 15.seconds, include_seconds: true)  # => less than 20 seconds
```

#### select_date

Returns a set of HTML select-tags (one for year, month, and day) pre-selected with the `date` provided.

```ruby
# 指定された日付 (ここでは本日から6日後) をデフォルト値とする日付セレクトボックスを生成する
select_date(Time.today + 6.days)

# 日付の指定がない場合、本日をデフォルト値とする日付セレクトボックスを生成する
select_date()
```

#### select_datetime

Returns a set of HTML select-tags (one for year, month, day, hour, and minute) pre-selected with the `datetime` provided.

```ruby
# 指定された日時 (ここでは本日から4日後) をデフォルト値とする日時セレクトボックスを生成する
select_datetime(Time.now + 4.days)

# 日時の指定がない場合、本日をデフォルト値とする日時セレクトボックスを生成する
select_datetime()
```

#### select_day

Returns a select tag with options for each of the days 1 through 31 with the current day selected.

```ruby
# Generates a select field for days that defaults to the day for the date provided
select_day(Time.today + 2.days)

# Generates a select field for days that defaults to the number given
select_day(5)
```

#### select_hour

Returns a select tag with options for each of the hours 0 through 23 with the current hour selected.

```ruby
# Generates a select field for hours that defaults to the hours for the time provided
select_hour(Time.now + 6.hours)
```

#### select_minute

Returns a select tag with options for each of the minutes 0 through 59 with the current minute selected.

```ruby
# 指定された分をデフォルト値として持つセレクトボックスを生成する
select_minute(Time.now + 6.hours)
```

#### select_month

Returns a select tag with options for each of the months January through December with the current month selected.

```ruby
# Generates a select field for months that defaults to the current month
select_month(Date.today)
```

#### select_second

Returns a select tag with options for each of the seconds 0 through 59 with the current second selected.

```ruby
# Generates a select field for seconds that defaults to the seconds for the time provided
select_second(Time.now + 16.minutes)
```

#### select_time

Returns a set of HTML select-tags (one for hour and minute).

```ruby
# Generates a time select that defaults to the time provided
select_time(Time.now)
```

#### select_year

Returns a select tag with options for each of the five years on each side of the current, which is selected. The five year radius can be changed using the `:start_year` and `:end_year` keys in the `options`.

```ruby
# Generates a select field for five years on either side of Date.today that defaults to the current year
select_year(Date.today)

# Generates a select field from 1900 to 2009 that defaults to the current year
select_year(Date.today, start_year: 1900, end_year: 2009)
```

#### time_ago_in_words

Like `distance_of_time_in_words`, but where `to_time` is fixed to `Time.now`.

```ruby
time_ago_in_words(3.minutes.from_now)  # => 3 minutes
```

#### time_select

Returns a set of select tags (one for hour, minute and optionally second) pre-selected for accessing a specified time-based attribute. The selects are prepared for multi-parameter assignment to an Active Record object.

```ruby
# Creates a time select tag that, when POSTed, will be stored in the order variable in the submitted attribute
time_select("order", "submitted")
```

### DebugHelper

YAMLからダンプしたオブジェクトを含む`pre`タグを返します。

```ruby
my_hash = {'first' => 1, 'second' => 'two', 'third' => [1,2,3]}
debug(my_hash)
```

```html
<pre class='debug_dump'>---
first: 1
second: two
third:
- 1
- 2
- 3
</pre>
```

### FormHelper

フォームヘルパーを使用すると、標準のHTML要素だけを使用するよりもはるかに容易に、モデルと連携動作するフォームを作成することができます。Formヘルパーはフォーム用のHTMLを生成し、テキストやパスワードといった入力の種類に応じたメソッドを提供します。(送信ボタンがクリックされたり、JavaScriptでform.submitを呼び出すなどして) フォームが送信されると、フォームの入力内容はparamsオブジェクトにまとめて保存され、コントローラに渡されます。

フォームヘルパーは、モデル属性の操作に特化したものと、より一般的なものの2種類に分類できます。ここではモデル属性の扱いに特化したものについて説明します。モデル属性に特化していない一般的なフォームヘルパーについては、ActionView::Helpers::FormTagHelperのドキュメントを参照してください。

ここで扱うフォームヘルパーの中心となるメソッドはform_forです。このメソッドはモデルのインスタンスからフォームを作成することができます。たとえば、以下のようにPersonというモデルがあり、このモデルをもとにしてインスタンスを1つ作成するとします。

```html+erb
# メモ: a @person変数はコントローラ側で設定済みであるとする (@person = Person.newなど)
<%= form_for @person, url: {action: "create"} do |f| %>
  <%= f.text_field :first_name %>
  <%= f.text_field :last_name %>
  <%= submit_tag 'Create' %>
<% end %>
```

上のコードによって生成されるHTMLは以下のようになります

```html
<form action="/people/create" method="post">
  <input id="person_first_name" name="person[first_name]" type="text" />
  <input id="person_last_name" name="person[last_name]" type="text" />
  <input name="commit" type="submit" value="Create" />
</form>
```

上のフォームが送信される時に作成されるparamsオブジェクトは以下のようになります。

```ruby
{"action" => "create", "controller" => "people", "person" => {"first_name" => "William", "last_name" => "Smith"}}
```

上のparamsハッシュには、Personモデル用の値がネストした形で含まれているので、コントローラで`params[:person]`と書くことで内容にアクセスできます。

#### check_box

指定された属性にアクセスするためのチェックボックスタグを生成します。

```ruby
# Let's say that @article.validated? is 1:
check_box("article", "validated")
# => <input type="checkbox" id="article_validated" name="article[validated]" value="1" />
#    <input name="article[validated]" type="hidden" value="0" />
```

#### fields_for

form_forのような特定のモデルオブジェクトの外側にスコープを作成しますが、フォームタグ自体は作成しません。このため、fields_forは同じフォームに別のモデルオブジェクトを追加するのに向いています。

```html+erb
<%= form_for @person, url: {action: "update"} do |person_form| %>
  First name: <%= person_form.text_field :first_name %> 
  Last name : <%= person_form.text_field :last_name %>

  <%= fields_for @person.permission do |permission_fields| %>
    Admin?  : <%= permission_fields.check_box :admin %>
  <% end %> <% end %>  ```

#### file_field

特定の属性にアクセスするための、ファイルアップロード用inputタグを返します。

```ruby
file_field(:user, :avatar)
# => <input type="file" id="user_avatar" name="user[avatar]" />
```

#### form_for

フィールドにどのような値があるかを問い合わせるのに使用される、特定のモデルオブジェクトの外側にフォームを1つとスコープを1つ作成します。

```html+erb
<%= form_for @article do |f| %>
  <%= f.label :title, 'Title' %>:
  <%= f.text_field :title %><br>
  <%= f.label :body, 'Body' %>:
  <%= f.text_area :body %><br>
<% end %>
```

#### hidden_field

特定の属性にアクセスするための、隠されたinputタグを返します。

```ruby
hidden_field(:user, :token)
# => <input type="hidden" id="user_token" name="user[token]" value="#{@user.token}" />
```

#### label

特定の属性用のinputフィールドに与えるラベルを返します。

```ruby
label(:article, :title)
# => <label for="article_title">Title</label>
```

#### password_field

特定の属性にアクセスするための、種類が"password"のinputタグを返します。

```ruby
password_field(:login, :pass)
# => <input type="text" id="login_pass" name="login[pass]" value="#{@login.pass}" />
```

#### radio_button

特定の属性にアクセスするためのラジオボタンタグを返します。

```ruby
# Let's say that @article.category returns "rails":
radio_button("article", "category", "rails")
radio_button("article", "category", "java")
# => <input type="radio" id="article_category_rails" name="article[category]" value="rails" checked="checked" />
#    <input type="radio" id="article_category_java" name="article[category]" value="java" />
```

#### text_area

特定の属性にアクセスするための、テキストエリア用開始タグと終了タグを返します。

```ruby
text_area(:comment, :text, size: "20x30")
# => <textarea cols="20" rows="30" id="comment_text" name="comment[text]">
#      #{@comment.text}
#    </textarea>
```

#### text_field

特定の属性にアクセスするための、種類が"text"のinputタグを返します。

```ruby
text_field(:article, :title)
# => <input type="text" id="article_title" name="article[title]" value="#{@article.title}" />
```

#### email_field

特定の属性にアクセスするための、種類が"emai"のinputタグを返します。

```ruby
email_field(:user, :email)
# => <input type="email" id="user_email" name="user[email]" value="#{@user.email}" />
```

#### url_field

特定の属性にアクセスするための、種類が"url"のinputタグを返します。

```ruby
url_field(:user, :url)
# => <input type="url" id="user_url" name="user[url]" value="#{@user.url}" />
```

### FormOptionsHelper

さまざまな種類のコンテナを1つのオプションタグのセットにまとめるためのメソッドを多数提供します。

#### collection_select

`select`タグと、`object`が属するクラスのメソッド値の既存の戻り値をコレクションにした`option`タグを返します。

例として、このメソッドを適用するオブジェクトの構造が以下のようになっているとします。

```ruby
class Article < ActiveRecord::Base
  belongs_to :author
end

class Author < ActiveRecord::Base
  has_many :articles
  def name_with_initial
    "#{first_name.first}. #{last_name}"
  end
end
```

Sample usage (selecting the associated Author for an instance of Article, `@article`):

```ruby
collection_select(:article, :author_id, Author.all, :id, :name_with_initial, {prompt: true})
```

If `@article.author_id` is 1, this would return:

```html
<select name="article[author_id]">
  <option value="">Please select</option>
  <option value="1" selected="selected">D. Heinemeier Hansson</option>
  <option value="2">D. Thomas</option>
  <option value="3">M. Clark</option>
</select>
```

#### collection_radio_buttons

`object`が属するクラスのメソッド値の既存の戻り値をコレクションにした`radio_button`タグを返します。

例として、このメソッドを適用するオブジェクトの構造が以下のようになっているとします。

```ruby
class Article < ActiveRecord::Base
  belongs_to :author
end

class Author < ActiveRecord::Base
  has_many :articles
  def name_with_initial
    "#{first_name.first}. #{last_name}"
  end
end
```

Sample usage (selecting the associated Author for an instance of Article, `@article`):

```ruby
collection_radio_buttons(:article, :author_id, Author.all, :id, :name_with_initial)
```

If `@article.author_id` is 1, this would return:

```html
<input id="article_author_id_1" name="article[author_id]" type="radio" value="1" checked="checked" />
<label for="article_author_id_1">D. Heinemeier Hansson</label>
<input id="article_author_id_2" name="article[author_id]" type="radio" value="2" />
<label for="article_author_id_2">D. Thomas</label>
<input id="article_author_id_3" name="article[author_id]" type="radio" value="3" />
<label for="article_author_id_3">M. Clark</label>
```

#### collection_check_boxes

`object`が属するクラスのメソッド値の既存の戻り値をコレクションにした`check_box`タグを返します。

例として、このメソッドを適用するオブジェクトの構造が以下のようになっているとします。

```ruby
class Article < ActiveRecord::Base
  has_and_belongs_to_many :authors
end

class Author < ActiveRecord::Base
  has_and_belongs_to_many :articles
  def name_with_initial
    "#{first_name.first}. #{last_name}"
  end
end
```

Sample usage (selecting the associated 作者 for an instance of Article, `@article`):

```ruby
collection_check_boxes(:article, :author_ids, Author.all, :id, :name_with_initial)
```

If `@article.author_ids` is [1], this would return:

```html
<input id="article_author_ids_1" name="article[author_ids][]" type="checkbox" value="1" checked="checked" />
<label for="article_author_ids_1">D. Heinemeier Hansson</label>
<input id="article_author_ids_2" name="article[author_ids][]" type="checkbox" value="2" />
<label for="article_author_ids_2">D. Thomas</label>
<input id="article_author_ids_3" name="article[author_ids][]" type="checkbox" value="3" />
<label for="article_author_ids_3">M. Clark</label>
<input name="article[author_ids][]" type="hidden" value="" />
```

#### option_groups_from_collection_for_select

`option`タグの文字列を返します。後述の`options_from_collection_for_select`と似ていますが、引数のオブジェクトリレーションに基いて`optgroup`タグを使用する点が異なります。

例として、このメソッドを適用するオブジェクトの構造が以下のようになっているとします。

```ruby
class Continent < ActiveRecord::Base
  has_many :countries
  # attribs: id, name
end

class Country < ActiveRecord::Base
  belongs_to :continent
  # attribs: id, name, continent_id
end
```

使用例は以下のようになります。

```ruby
option_groups_from_collection_for_select(@continents, :countries, :name, :id, :name, 3)
```

出力結果は以下のようになります。

```html
<optgroup label="Africa">
  <option value="1">Egypt</option>
  <option value="4">Rwanda</option>
  ...
</optgroup>
<optgroup label="Asia">
  <option value="3" selected="selected">China</option>
  <option value="12">India</option>
  <option value="5">Japan</option>
  ...
</optgroup>
```

NOTE: 返されるのは`optgroup`タグと`option`だけです。従って、出力結果の外側を適切な`select`タグで囲む必要があります。

#### options_for_select

コンテナ (ハッシュ、配列、enumerable、独自の型) を引数として受け付け、オプションタグの文字列を返します。

```ruby
options_for_select([ "VISA", "MasterCard" ])
# => <option>VISA</option> <option>MasterCard</option>
```

NOTE: 返されるのは`option`だけです。従って、出力結果の外側を適切なHTML `select`タグで囲む必要があります。

#### options_from_collection_for_select

`collection`を列挙した結果をoptionタグ化した文字列を返し、呼び出しの結果を`value_method`にオプション値として割り当て、`text_method`にオプションテキストとして割り当てます。

```ruby
options_from_collection_for_select(collection, value_method, text_method, selected = nil)
```

たとえば、@project.peopleに入っているpersonをループですべて列挙してinputタグを作成するのであれば、以下のようになります。

```ruby
options_from_collection_for_select(@project.people, "id", "name")
# => <option value="#{person.id}">#{person.name}</option>
```

NOTE: 返されるのは`option`だけです。従って、出力結果の外側を適切なHTML `select`タグで囲む必要があります。

#### select

指定されたオブジェクトとメソッドに従って、selectタグの中に一連のoptionタグを含んだものを作成します。

Example:

```ruby
select("article", "person_id", Person.all.collect {|p| [ p.name, p.id ] }, {include_blank: true})
```

If `@article.person_id` is 1, this would become:

```html
<select name="article[person_id]">
  <option value=""></option>
  <option value="1" selected="selected">David</option>
  <option value="2">Sam</option>
  <option value="3">Tobias</option>
</select>
```

#### time_zone_options_for_select

世界のほぼすべてのタイムゾーンを含むオプションタグの文字列を返します。

#### time_zone_select

time_zone_options_for_selectを使用してオプションタグを生成し、指定されたオブジェクトとメソッド用のselectタグとoptionタグを返します。

```ruby
time_zone_select( "user", "time_zone")
```

#### date_field

特定の属性にアクセスするための、種類が"date"のinputタグを返します。

```ruby
date_field("user", "dob")
```

### FormTagHelper

フォームタグを作成するためのメソッドを多数提供します。これらのメソッドは、テンプレートに割り当てられているActive Recordオブジェクトに依存しない点がFormHelperと異なります。その代わり、FormTagHelperのメソッドでは名前と値を個別に指定します。

#### check_box_tag

チェックボックス用のフォームinputタグを作成します。

```ruby
check_box_tag 'accept'
# => <input id="accept" name="accept" type="checkbox" value="1" />
```

#### field_set_tag

HTMLフォーム要素をグループ化するためのfieldsetタグを作成します。

```html+erb
<%= field_set_tag do %>
  <p><%= text_field_tag 'name' %></p>
<% end %>
# => <fieldset><p><input id="name" name="name" type="text" /></p></fieldset>
```

#### file_field_tag

ファイルアップロード用のフィールドを作成します。

```html+erb
<%= form_tag({action:"post"}, multipart: true) do %>
  <label for="file">File to Upload</label> <%= file_field_tag "file" %>
  <%= submit_tag %>
<% end %>
```

出力例:

```ruby
file_field_tag 'attachment'
# => <input id="attachment" name="attachment" type="file" />
```

#### form_tag

`url_for_options`で設定されたURLへのアクションに送信されるフォームタグを作成します。これは`ActionController::Base#url_for`と似ています。

```html+erb
<%= form_tag '/articles' do %>
  <div><%= submit_tag 'Save' %></div>
<% end %>
# => <form action="/articles" method="post"><div><input type="submit" name="submit" value="Save" /></div></form>
```

#### hidden_field_tag

フォームinputの「隠しフィールド」を作成します。この隠しフィールドは、通常であればHTTPがステートレスであることによって失われる可能性のあるデータを送信したり、ユーザーから見えないようにしておきたいデータを送信するのに使用されます。

```ruby
hidden_field_tag 'token', 'VUBJKB23UIVI1UU1VOBVI@'
# => <input id="token" name="token" type="hidden" value="VUBJKB23UIVI1UU1VOBVI@" />
```

#### image_submit_tag

送信画像を表示します。この画像をクリックするとフォームが送信されます。

```ruby
image_submit_tag("login.png")
# => <input src="/images/login.png" type="image" />
```

#### label_tag

フィールドのラベルを作成します。

```ruby
label_tag 'name'
# => <label for="name">Name</label>
```

#### password_field_tag

パスワード用のフィールドを作成します。このフィールドへの入力はマスク用文字で隠されます。

```ruby
password_field_tag 'pass'
# => <input id="pass" name="pass" type="password" />
```

#### radio_button_tag

ラジオボタンを作成します。ユーザーが同じオプショングループ内から選択できるよう、同じname属性でラジオボタンをグループ化してください。

```ruby
radio_button_tag 'gender', 'male'
# => <input id="gender_male" name="gender" type="radio" value="male" />
```

#### select_tag

ドロップダウン選択ボックスを作成します。

```ruby
select_tag "people", "<option>David</option>"
# => <select id="people" name="people"><option>David</option></select>
```

#### submit_tag

キャプションとして指定されたテキストを使用して送信ボタンを作成します。

```ruby
submit_tag "Publish this article"
# => <input name="commit" type="submit" value="Publish this article" />
```

#### text_area_tag

textareaタグでテキスト入力エリアを作成します。ブログへの投稿や説明文などの長いテキストを入力するにはtextareaをご使用ください。

```ruby
text_area_tag 'article'
# => <textarea id="article" name="article"></textarea>
```

#### text_field_tag

通常のテキストフィールドを作成します。ユーザー名や検索キーワード入力用のフィールドにはこの通常のテキストフィールドをご使用ください。

```ruby
text_field_tag 'name'
# => <input id="name" name="name" type="text" />
```

#### email_field_tag

種類が`email`の標準入力フィールドを作成します。

```ruby
email_field_tag 'email'
# => <input id="email" name="email" type="email" />
```

#### url_field_tag

種類が`url`の標準入力フィールドを作成します。

```ruby
url_field_tag 'url'
# => <input id="url" name="url" type="url" />
```

#### date_field_tag

種類が`date`の標準入力フィールドを作成します。

```ruby
date_field_tag "dob"
# => <input id="dob" name="dob" type="date" />
```

### JavaScriptHelper

ビューでJavaScriptを使用するための機能を提供します。

#### escape_javascript

JavaScriptセグメントから改行 (CR) と一重引用符と二重引用符をエスケープします。

#### javascript_tag

渡されたコードをJavaScript用タグにラップして返します。

```ruby
javascript_tag "alert('All is good')"
```

```html
<script>
//<![CDATA[
alert('All is good')
//]]>
</script>
```

### NumberHelper

数値をフォーマット済み文字列に変換するメソッド群を提供します。サポートされているフォーマットは電話番号、通貨、パーセント、精度、座標、ファイルサイズなどです。

#### number_to_currency

数値を通貨表示に変換します ($13.65など)。

```ruby
number_to_currency(1234567890.50) # => $1,234,567,890.50
```

#### number_to_human_size

バイト数を読みやすい形式にフォーマットします。ファイルサイズをユーザーに表示する場合に便利です。

```ruby
number_to_human_size(1234)          # => 1.2 KB
number_to_human_size(1234567)       # => 1.2 MB
```

#### number_to_percentage

数値をパーセント文字列に変換します。

```ruby
number_to_percentage(100, precision: 0)        # => 100%
```

#### number_to_phone

数値を米国式の電話番号に変換します。

```ruby
number_to_phone(1235551234) # => 123-555-1234
```

#### number_with_delimiter

数値に3桁ごとの桁区切り文字を追加します。

```ruby
number_with_delimiter(12345678) # => 12,345,678
```

#### number_with_precision

数値を指定された精度(`precision`)に変換します。デフォルトの精度は3です。

```ruby
number_with_precision(111.2345)     # => 111.235
number_with_precision(111.2345, 2)  # => 111.23
```

### SanitizeHelper

SanitizeHelperモジュールは、望ましくないHTML要素を除去するためのメソッド群を提供します。

#### sanitize

This sanitize helper will HTML encode all tags and strip all attributes that aren't specifically allowed.

```ruby
sanitize @article.body
```

[REVIEW]:attributesオプションまたは:tagsオプションが渡されると、そこで指定されたタグおよび属性のみが処理の対象外となります。

```ruby
sanitize @article.body, tags: %w(table tr td), attributes: %w(id class style)
```

さまざまな用途に合わせてデフォルト設定を変更できます。たとえば以下のようにデフォルトのタグにtableタグを追加するとします。

```ruby
class Application < Rails::Application
  config.action_view.sanitized_allowed_tags = 'table', 'tr', 'td'
end
```

#### sanitize_css(style)

CSSコードをサニタイズします。

#### strip_links(html)
リンクテキストを残してリンクタグをすべて削除します。

```ruby
strip_links("<a href="http://rubyonrails.org">Ruby on Rails</a>")
# => Ruby on Rails
```

```ruby
strip_links("emails to <a href="mailto:me@email.com">me@email.com</a>.")
# => emails to me@email.com.
```

```ruby
strip_links('Blog: <a href="http://myblog.com/">Visit</a>.')
# => Blog: Visit.
```

#### strip_tags(html)

HTMLからHTMLタグをすべて削除します。HTMLコメントも削除されます。
このメソッドではHTMLスキャナとHTMLトークナイザ (tokenizer) を使用しており、HTMLの解析能力はスキャナの能力に依存しています。

```ruby
strip_tags("Strip <i>these</i> tags!")
# => Strip these tags!
```

```ruby
strip_tags("<b>Bold</b> no more!  <a href='more.html'>See more</a>")
# => Bold no more!  See more
```

CAUTION: この出力にはエスケープされていない'<'、'>'、'&'文字が残ることがあり、それによってブラウザが期待どおりに動作しなくなることがあります。

### CsrfHelper

"csrf-param"メタタグと"csrf-token"メタタグを返します。これらの名称はそれぞれ、クロスサイトリクエストフォージェリ (CSRF: cross-site request foregery) のパラメータとトークンが元になっています。

```html
<%= csrf_meta_tags %>
```

NOTE: 通常のフォームではそのための隠しフィールドが生成されるので、これらのタグは使用されません。詳細については[セキュリティガイド](security.html#クロスサイトリクエストフォージェリ-(csrf))を参照してください。

ローカライズされたビュー
---------------

Action Viewは、現在のロケールに応じてさまざまなテンプレートを出力することができます。

以下に例を示します。 suppose you have a `ArticlesController` with a show action. By default, calling this action will render `app/views/articles/show.html.erb`. But if you set `I18n.locale = :de`, then `app/views/articles/show.de.html.erb` will be rendered instead. ローカライズ版のテンプレートが見当たらない場合は、装飾なしのバージョンが使用されます。つまり、ローカライズ版ビューがなくても動作しますが、ローカライズ版ビューがあればそれが使用されます。

同じ要領で、publicディレクトリのレスキューファイル (いわゆるエラーページ) もローカライズできます。たとえば、`I18n.locale = :de`と設定し、`public/500.de.html`と`public/404.de.html`を作成することで、ローカライズ版のレスキューページを作成できます。

RailsはI18n.localeに設定できるシンボルを制限していないので、ローカライズにかぎらず、あらゆる状況に合わせて異なるコンテンツを表示し分けるようにすることができます。たとえば、エキスパートユーザーには、通常ユーザーと異なる画面を表示したいとします。これを行なうには、`app/controllers/application.rb`に以下のように追記します。

```ruby
before_action :set_expert_locale

def set_expert_locale
  I18n.locale = :expert if current_user.expert?
end
```

Then you could create special views like `app/views/articles/show.expert.html.erb` that would only be displayed to expert users.

詳細については、[Rails国際化 (I18n) API](i18n.html) を参照してください。