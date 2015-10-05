Action View フォームヘルパー
============

Webアプリケーションにおけるフォームは、ユーザー入力を扱うのに不可欠なインターフェイスです。However, form markup can quickly become tedious to write and maintain because of the need to handle form control naming and its numerous attributes. Rails does away with this complexity by providing view helpers for generating form markup. However, since these helpers have different use cases, developers need to know the differences between the helper methods before putting them to use.

このガイドの内容:

* 検索フォーム、および特定のモデルを表さない一般的なフォームの作成法
* How to make model-centric forms for creating and editing specific database records.
* 複数の種類のデータからセレクトボックスを生成する方法
* What date and time helpers Rails provides.
* ファイルアップロード用フォームの動作変更方法
* How to post forms to external resources and specify setting an `authenticity_token`.
* 複雑なフォームの作成方法

--------------------------------------------------------------------------------

NOTE: このガイドはフォームヘルパーとその引数について網羅的に説明するものではありません。完全なリファレンスについては[Rails APIドキュメント](http://api.rubyonrails.org/)を参照してください。

基本的なフォームを作成する
------------------------

最も基本的なフォームヘルパは`form_tag`です。

```erb
<%= form_tag do %>
  Form contents
<% end %>
```

上のように引数なしで呼び出されると`<form>`タグを生成します。このフォームを現在のページに送信するときにはHTTPのPOSTメソッドが使用されます。たとえば現在のページが`/home/index`の場合、以下のようなHTMLが生成されます (読みやすくするため改行を追加してあります)。

```html
<form accept-charset="UTF-8" action="/" method="post">
  <input name="utf8" type="hidden" value="&#x2713;" />
  <input name="authenticity_token" type="hidden" value="J7CBxfHalt49OSHp27hblqK20c9PgwJ108nDHX/8Cts=" />
  Form contents
</form>
```

You'll notice that the HTML contains `input` element with type `hidden`. This `input` is important, because the form cannot be successfully submitted without it. The hidden input element has name attribute of `utf8` enforces browsers to properly respect your form's character encoding and is generated for all forms whether their actions are "GET" or "POST". 2番目隠しinput要素である`authenticity_token`要素は **クロスサイトリクエストフォージェリへの保護** のためのセキュリティ機能です。この要素はGET以外のすべてのフォームで生成されます (セキュリティ機能が有効になっている場合)。You can read more about this in the [Security Guide](security.html#cross-site-request-forgery-csrf).

### 一般的な検索フォーム

検索フォームはWebでよく使われています。このフォームには以下のものが含まれています。

* "GET"メソッドを対象としたフォーム要素
* 入力するものを示すラベル
* テキスト入力要素
* [送信]ボタン要素

このフォームを作成するには、`form_tag`、`label_tag`、`text_field_tag`、`submit_tag`が必要です。以下に例を示します。

```erb
<%= form_tag("/search", method: "get") do %>
  <%= label_tag(:q, "Search for:") %>
  <%= text_field_tag(:q) %>
  <%= submit_tag("Search") %
<% end %>
```

上のコードから以下のHTMLが生成されます。

```html
<form accept-charset="UTF-8" action="/search" method="get">
  <input name="utf8" type="hidden" value="&#x2713;" />
  <label for="q">Search for:</label>
  <input id="q" name="q" type="text" />
  <input name="commit" type="submit" value="Search" />
</form>
```

TIP: For every form input, an ID attribute is generated from its name (`"q"` in above example). これらのidは、cssでのスタイル追加やJavaScriptによるフォーム制御で使用するのに便利です。

HTMLの __すべての__ フォームコントロールには、`text_field_tag`や`submit_tag`と同様の便利なヘルパーが用意されています。

IMPORTANT: フォームを検索に使用する場合は必ず"GET"メソッドを使用してください。こうすることで、検索クエリがURLの一部となるので、ユーザーが検索結果をブックマークすると同じ検索を後でブックマークから実行することができます。Railsでは基本的に、アクションに対応する適切なHTTP verbを常に選ぶようにしてください (訳注: セキュリティガイドにも記載されていますが、更新フォームでGETメソッドを使用すると重大なセキュリティホールが生じます)。

### フォームヘルパーの呼び出しで複数のハッシュを使用する

`form_tag`ヘルパーは2つの引数を取ります。1つはアクションへのパスで、もう1つはオプションのハッシュです。このハッシュには、フォーム送信のメソッドと、HTMLオプション()が含まれます。

`link_to`ヘルパーのときと同様、文字列以外の引数も受け取れます。たとえば、Railsのルーティングメカニズムで認識可能なURLパラメータのハッシュを受け取り、このハッシュを正しいURLに変換することができます。ただし、`form_tag`の引数を両方ともハッシュにするとたちまち問題が生じるでしょう。たとえば次のようなコードを書いたとします。

```ruby
form_tag(controller: "people", action: "search", method: "get", class: "nifty_form")
# => '<form accept-charset="UTF-8" action="/people/search?method=get&class=nifty_form" method="post">'
```

上のコードでは、生成されたURLに`method`と`class`が追加されてしまっています。たとえ2つのハッシュを書いたつもりでも、実際にはそれらが1つのものとして扱われてしまっています。従って、波かっこ { } を使用して1つ目のハッシュを (あるいはどちらのハッシュも) 区別してあげる必要があります。今度は期待どおりのHTMLが生成されます。

```ruby
form_tag({controller: "people", action: "search"}, method: "get", class: "nifty_form")
# => '<form accept-charset="UTF-8" action="/people/search" method="get" class="nifty_form">'
```

### フォーム要素生成に使用するヘルパー

Railsには、チェックボックス/テキストフィールド/ラジオボタンなどのフォーム要素を生成するためのヘルパーが多数用意されています。These basic helpers, with names
ending in `_tag` (such as `text_field_tag` and `check_box_tag`), generate just a
single `<input>` element. これらのヘルパーの1番目のパラメータは、inputの名前と決まっています。フォームが送信されると、この名前がフォームデータに含まれて渡され、ユーザーが入力した値とともに、コントローラ内で`params`ハッシュとなってアクセス可能になります。たとえば、フォームに`<%= text_field_tag(:query) %>`というコードが含まれていたとすると、コントローラで`params[:query]`と指定することによってこのフィールドの値にアクセスできます。

Railsは、inputに名前を与えるときに一定のルールに従っています。これにより、配列やハッシュのような「非スカラー値」のパラメータをフォームから送信できるようになり、その結果`params`としてコントローラでアクセスできるようになるのです。詳細については[本ガイドの7章](#パラメータの命名ルールを理解する)を参照してください。これらのヘルパーの正確な使用法については[APIドキュメント](http://api.rubyonrails.org/classes/ActionView/Helpers/FormTagHelper.html)を参照してください。

#### チェックボックス

チェックボックスはフォームコントロールの一種で、ユーザーがオプションをオンまたはオフにできるようにするためのものです。

```erb
<%= check_box_tag(:pet_dog) %>
<%= label_tag(:pet_dog, "I own a dog") %>
<%= check_box_tag(:pet_cat) %>
<%= label_tag(:pet_cat, "I own a cat") %>
```

上のコードによって以下が生成されます。

```html
<input id="pet_dog" name="pet_dog" type="checkbox" value="1" />
<label for="pet_dog">I own a dog</label>
<input id="pet_cat" name="pet_cat" type="checkbox" value="1" />
<label for="pet_cat">I own a cat</label>
```

`check_box_tag`の最初のパラメータは、言うまでもなくinputの名前です。2番目のパラメータは、input(タグ)のvalue属性になります。チェックボックスをオンにすると、この値はフォームデータに含まれ、最終的に`params`に渡されます。

#### ラジオボタン

ラジオボタンも、チェックボックスと同様に一連のオプションをユーザーが選択できるようにするものですが、一度に1つの項目しか選択できない排他的な動作が特徴です。Radio buttons, while similar to checkboxes, are controls that specify a set of options in which they are mutually exclusive (i.e., the user can only pick one):

```erb
<%= radio_button_tag(:age, "child") %>
<%= label_tag(:age_child, "I am younger than 21") %>
<%= radio_button_tag(:age, "adult") %>
<%= label_tag(:age_adult, "I'm over 21") %>
```

出力は以下のようになります。

```html
<input id="age_child" name="age" type="radio" value="child" />
<label for="age_child">I am younger than 21</label>
<input id="age_adult" name="age" type="radio" value="adult" />
<label for="age_adult">I'm over 21</label>
```

`check_box_tag`ヘルパーのときと同様、`radio_button_tag`の2番目のパラメータがinput(タグ)のvalue属性になります。Because these two radio buttons share the same name (`age`), the user will only be able to select one of them, and `params[:age]` will contain either `"child"` or `"adult"`.

NOTE: チェックボックスとラジオボタンには必ずラベルを表示してください。ラベルを表示することで、そのオプションとラベルの名前が関連付けられるだけでなく、ラベルの部分までクリック可能になるのでユーザーにとってクリックしやすくなります。

### その他のヘルパー

これまで紹介した他にも、次のようなフィールドがあります: テキストエリア、パスワード、隠しフィールド、検索フィールド、電話番号フィールド、日付フィールド、時刻フィールド、色フィールド、日時フィールド、ローカル日時フィールド、月フィールド、週フィールド、URLフィールド、メールアドレスフィールド、数値フィールド、範囲フィールド。

```erb
<%= text_area_tag(:message, "Hi, nice site", size: "24x6") %>
<%= password_field_tag(:password) %>
<%= hidden_field_tag(:parent_id, "5") %>
<%= search_field(:user, :name) %>
<%= telephone_field(:user, :phone) %>
<%= date_field(:user, :born_on) %>
<%= datetime_field(:user, :meeting_time) %>
<%= datetime_local_field(:user, :graduation_day) %>
<%= month_field(:user, :birthday_month) %>
<%= week_field(:user, :birthday_week) %>
<%= url_field(:user, :homepage) %>
<%= email_field(:user, :address) %>
<%= color_field(:user, :favorite_color) %>
<%= time_field(:task, :started_at) %>
<%= number_field(:product, :price, in: 1.0..20.0, step: 0.5) %>
<%= range_field(:product, :discount, in: 1..100) %>
```

出力は以下のようになります。

```html
<textarea id="message" name="message" cols="24" rows="6">Hi, nice site</textarea>
<input id="password" name="password" type="password" />
<input id="parent_id" name="parent_id" type="hidden" value="5" />
<input id="user_name" name="user[name]" type="search" />
<input id="user_phone" name="user[phone]" type="tel" />
<input id="user_born_on" name="user[born_on]" type="date" />
<input id="user_meeting_time" name="user[meeting_time]" type="datetime" />
<input id="user_graduation_day" name="user[graduation_day]" type="datetime-local" />
<input id="user_birthday_month" name="user[birthday_month]" type="month" />
<input id="user_birthday_week" name="user[birthday_week]" type="week" />
<input id="user_homepage" name="user[homepage]" type="url" />
<input id="user_address" name="user[address]" type="email" />
<input id="user_favorite_color" name="user[favorite_color]" type="color" value="#000000" />
<input id="task_started_at" name="task[started_at]" type="time" />
<input id="product_price" max="20.0" min="1.0" name="product[price]" step="0.5" type="number" />
<input id="product_discount" max="100" min="1" name="product[discount]" type="range" />
```

隠しフィールドはユーザーには表示されず、事前に与えられた値を種類を問わず保持します。隠しフィールドに含まれている値はJavaScriptを使用して変更できます。

IMPORTANT: 「検索、電話、日付、時刻、色、日時、ローカル日時、月、週、URL、メールアドレス、数値、範囲」フィールドはHTML5から利用できるようになったコントロールです。
これらのフィールドを古いブラウザでも同じように扱いたいのであれば、CSSやJavaScriptを使用したHTML5ポリフィルが必要になるでしょう。
古いブラウザでHTML5に対応する方法は[山ほどあります](https://github.com/Modernizr/Modernizr/wiki/HTML5-Cross-Browser-Polyfills)が、現時点で代表的なものは[Modernizr](http://www.modernizr.com/)と[yepnope](http://yepnopejs.com/)でしょう(訳注: yepnopeについては[非推奨宣言](https://github.com/SlexAxton/yepnope.js#deprecation-notice)が出されています)。これらは、HTML5の新機能が使用されていることが検出された場合に、機能を追加するためのシンプルな方法を提供します。

TIP: パスワード入力フィールドを使用しているのであれば、入力されたパスワードをRailsのログに残さないようにしたいと思うことでしょう。その方法については[セキュリティガイド](security.html#logging)を参照してください。

モデルオブジェクトの取り扱い
--------------------------

### モデルオブジェクトヘルパー

フォームの主な仕事といえば、モデルオブジェクトの作成および修正でしょう。A particularly common task for a form is editing or creating a model object. `*_tag`ヘルパーをモデルオブジェクトの作成/修正に用いることはもちろん可能ですが、1つ1つのタグについて正しいパラメータが使用されているか、入力のデフォルト値は適切に設定されているかなどをいちいちコーディングするのは何とも面倒です。Railsにはまさにこのような作業を軽減するのにうってつけのヘルパーがあります。These helpers lack the `_tag` suffix, for example `text_field`, `text_area`.

これらのヘルパーの最初の引数はインスタンス変数名、2番目の引数はオブジェクトを呼び出すためのメソッド名 (通常は属性名を使用します)です。Railsは、オブジェクトのそのメソッドから値が返され、かつ適切な入力名が設定されるように、入力コントロールの値を設定してくれます。たとえば、コントローラで`@person`が定義されており、その人物の名前がHenryだとします。

```erb
<%= text_field(:person, :name) %>
```

このとき、上のコードからは以下の出力が得られます。

```erb
<input id="person_name" name="person[name]" type="text" value="Henry"/>
```

このフォームを送信すると、ユーザーが入力した値は`params[:person][:name]`に保存されます。`params[:person]`ハッシュは`Person.new`に渡しやすくなっています。`@person`がPersonモデルのインスタンスであれば`@person.update`にも渡しやすくなっています。これらのヘルパーでは2番目のパラメータとして属性名を渡すことがほとんどですが、必ずしもそうでないヘルパーもあります。上の例で言うなら、personオブジェクトに`name`メソッドと`name=`メソッドがありさえすればRailsは余分な作業をせずに済みます。

WARNING: ヘルパーに渡すのはインスタンス変数の「名前」でなければなりません (シンボル`:person`や文字列`"person"`など)。渡すのはモデルオブジェクトのインスタンスそのものではありません。

Railsのヘルパーには、モデルオブジェクトに関連する検証 (バリデーション) エラーを自動的に表示する機能もあります。詳細については本ガイドの[Active Record検証 (バリデーション)](./active_record_validations.html#バリデーションエラーをビューで表示する)を参照してください。

### フォームとオブジェクトを結び付ける

上のやり方でだいぶコーディングが楽になりましたが、改善の余地はまだまだあります。If `Person` has many attributes to edit then we would be repeating the name of the edited object many times. もっと楽に、フォームとモデルオブジェクトを結び付けるだけで簡単に作れないものか。それがまさに`form_for`なのです。

記事を扱うArticlesコントローラ`app/controllers/articles_controller.rb`があるとします。

```ruby
def new
  @article = Article.new
end
```

上のコントローラに対応するビュー`app/views/articles/new.html.erb`で`form_for`を使うと、以下のような感じになります。

```erb
<%= form_for @article, url: {action: "create"}, html: {class: "nifty_form"} do |f| %>
  <%= f.text_field :title %>
  <%= f.text_area :body, size: "60x12" %>
  <%= f.submit "Create" %>
<% end %>
```

以下の点にご注目ください。

* `@article`は、実際に編集されるオブジェクトそのものです (名前ではありません)。
* 1つのオプションに1つのハッシュが使用されています。ルーティングオプションが`:url`ハッシュで渡され、HTMLオプションが`:html`ハッシュで渡されています。フォームで`:namespace`オプションを使用して、フォーム要素上のid属性同士が衝突しないようにすることもできます。この名前空間属性の値は、生成されたHTMLのid属性の先頭にアンダースコア付きで追加されます。
* `form_for`メソッドからは **フォームビルダー** オブジェクト(ここでは変数`f`)が生成されます。
* フォームコントロールを作成するメソッドは、 **フォームビルダーオブジェクト`f`に対して** 呼び出されます。.

これにより、以下のHTMLが生成されます。

```html
<form accept-charset="UTF-8" action="/articles/create" method="post" class="nifty_form">
  <input id="article_title" name="article[title]" type="text" />
  <textarea id="article_body" name="article[body]" cols="60" rows="12"></textarea>
  <input name="commit" type="submit" value="Create" />
</form>
```

`form_for`に渡される名前は、`params`を使用してフォームの値にアクセスするときのキーに影響します。たとえば、この名前が`article`だとすると、すべての入力は`article[属性名]`というフォーム名を持ちます。従って`create`アクションでは、`:title`キーと`:body`キーを持つ1つのハッシュが`params[:article]`に含まれることになります。input名の重要性については、[パラメータの命名ルールを理解する](#パラメータの命名ルールを理解する)を参照してください。

フォームビルダー変数に対して呼び出されるヘルパーメソッドは、モデルオブジェクトのヘルパーメソッドと同一です。ただし、フォームの場合は編集の対象となるオブジェクトが既にフォームビルダーで管理されているので、どのオブジェクトに対して編集を行うかを指定する必要がない点が異なります。

`fields_for`メソッドを使用すれば、`<form>`タグを実際に作成することなく同様の結び付きを設定することができます。これは、同じフォームで別のモデルオブジェクトも編集できるようにしたい場合などに便利です。以下に例を示します。 if you had a `Person` model with an associated `ContactDetail` model, you could create a form for creating both like so:

```erb
<%= form_for @person, url: {action: "create"} do |person_form| %>
  <%= person_form.text_field :name %>
  <%= fields_for @person.contact_detail do |contact_details_form| %>
    <%= contact_details_form.text_field :phone_number %>
  <% end %> <% end %>  ```

上のコードから以下の出力が得られます。

```html
<form accept-charset="UTF-8" action="/people/create" class="new_person" id="new_person" method="post">
  <input id="person_name" name="person[name]" type="text" />
  <input id="contact_detail_phone_number" name="contact_detail[phone_number]" type="text" />
</form>
```

`fields_for`によって生成されたオブジェクトはフォームビルダーであり、`form_for`で生成されたものと似ています(実は`form_for`の内部では`fields_for`が呼び出されています)。

### レコード識別を利用する

これでArticleモデルをユーザーが直接操作できるようになりました。Rails開発で次に行なうべき最善の方法は、これを **リソース** として宣言することです。

```ruby
resources :articles
```

TIP: リソースを宣言すると、自動的に他にも多くの設定が行われます。リソースの設定方法の詳細については、[Railsルーティングガイド](routing.html#リソースベースのルーティング:-railsのデフォルト)を参照してください。

RESTfulなリソースを扱っている場合、レコード識別(record identification)を使用すると`form_for`の呼び出しがはるかに簡単になります。これは、モデルのインスタンスを渡すだけで、後はRailsがそこからモデル名など必要なものを取り出して処理してくれるというものです。

```ruby
## 新しい記事の作成
# 長いバージョン
form_for(@article, url: articles_path)
# 短いバージョン(レコード識別を利用)
form_for(@article)

## 既存の記事の修正
# 長いバージョン
form_for(@article, url: article_path(@article), html: {method: "patch"})
# 短いバージョン
form_for(@article)
```

この短い`form_for`呼び出しは、レコードの作成・編集のどちらにおいてもまったく同じになっています。これがどれほど便利であるかおわかりいただけると思います。レコード識別は、レコードが新しい場合には`record.new_record?`が必要とされている、などの適切な推測を行ってくれます。さらに送信用の正しいパスを選択し、オブジェクトのクラスに基づいた名前も選択してくれます。

Railsはフォームの`class`と`id`を自動的に設定してくれます。この場合、記事を作成するフォームには`id`と、`new_article`という`class`が与えられます。もし仮にidが23の記事を編集しようとしているのであれば、`class`は`edit_article`に設定され、idは`edit_article_23`に設定されます。なお、煩雑さを避けるため、以後これらの属性の表記は割愛します。

WARNING: モデルで単一テーブル継承(STI: single-table inheritance)を使用している場合、親クラスでリソースが宣言されていてもサブクラスでレコード識別を利用することはできません。その場合は、モデル名、`:url`、`:method`を明示的に指定する必要があります。

#### 名前空間を扱う

名前空間付きのルーティングを作成してある場合、`form_for`でもこれを利用した簡潔な表記が利用できます。アプリケーションのルーティングでadmin名前空間が設定されているとします。

```ruby
form_for [:admin, @article]
```

上のコードはそれによって、admin名前空間内にある`ArticlesController`に送信を行なうフォームを作成します (たとえば更新の場合は`admin_article_path(@article)`に送信されます)名前空間が多段階層になっている場合にも同様の文法が使用できます。If you have several levels of namespacing then the syntax is similar:

```ruby
form_for [:admin, :management, @article]
```

Railsのルーティングシステムの詳細と、関連するルールについては[ルーティングガイド](routing.html)を参照してください。

### フォームにおけるPATCH・PUT・DELETEメソッドの動作

Railsのフレームワークは、開発者がアプリケーションをRESTfulなデザインで構築するように働きかけています。すなわち、開発者はGETやPOSTリクエストだけでなく、PATCHやDELETEリクエストをたくさん作成・送信することになります。しかしながら、現実には多くのブラウザはフォーム送信時にGETとPOST以外のHTTPメソッドを _サポートしていません_ 。

そこでRailsでは、POSTメソッド上でこれらのメソッドをエミュレートすることによってこの問題を解決しています。具体的には、`"_method"`という名前の隠し入力をフォームに用意し、使いたいメソッドをここで指定します。

```ruby
form_tag(search_path, method: "patch")
```

上のコードから以下の出力が得られます。

```html
<form accept-charset="UTF-8" action="/search" method="post">
  <input name="_method" type="hidden" value="patch" />
  <input name="utf8" type="hidden" value="&#x2713;" />
  <input name="authenticity_token" type="hidden" value="f755bb0ed134b76c432144748a6d4b7a7ddf2b71" />
  ...
</form>
```

Railsは、POSTされたデータを解析する際にこの特殊な`_method`パラメータをチェックし、ここで指定されているメソッド(この場合はPATCH)があたかも実際にHTTPメソッドとして指定されたかのように振る舞います。

セレクトボックスを簡単に作成する
-----------------------------

Select boxes in HTML require a significant amount of markup (one `OPTION` element for each option to choose from), therefore it makes the most sense for them to be dynamically generated.

Here is what the markup might look like:

```html
<select name="city_id" id="city_id">
  <option value="1">Lisbon</option>
  <option value="2">Madrid</option>
  ...
  <option value="12">Berlin</option>
</select>
```

Here you have a list of cities whose names are presented to the user. Internally the application only wants to handle their IDs so they are used as the options' value attribute. Let's see how Rails can help out here.

### The Select and Option Tags

The most generic helper is `select_tag`, which - as the name implies - simply 上を実行すると以下が生成されます。 the `SELECT` tag that encapsulates an options string:

```erb
<%= select_tag(:city_id, '<option value="1">Lisbon</option>...') %>
```

This is a start, but it doesn't dynamically create the option tags. You can generate option tags with the `options_for_select` helper:

```html+erb
<%= options_for_select([['Lisbon', 1], ['Madrid', 2], ...]) %>

上のコードから以下の出力が得られます。

<option value="1">Lisbon</option>
<option value="2">Madrid</option>
...
```

`options_for_select`の最初の引数は入れ子になった配列であり、各要素には「オプションテキスト(city name)」と「オプション値(city id)」があります。オプション値の部分がコントローラに送信されます。送信されるidは、対応するデータベースオブジェクトのidであるのが普通ですが、必ずしもそうする必要はありません。

ここを理解すれば、`select_tag`と`options_for_select`を組み合わせて望み通りの完全なマークアップを得ることができます。

```erb
<%= select_tag(:city_id, options_for_select(...)) %>
```

`options_for_select`では、デフォルトにしたいオプションを値を渡すことでデフォルト値を設定できます。

```html+erb
<%= options_for_select([['Lisbon', 1], ['Madrid', 2], ...], 2) %>

上のコードから以下の出力が得られます。

<option value="1">Lisbon</option>
<option value="2" selected="selected">Madrid</option>
...
```

生成されるオプション内部の値がこの値とマッチすると、Railsは`selected`属性を自動的にそのオプションに追加します。

TIP: `options_for_select`の2番目の引数は、必要となる内部の値と正確に一致しなければなりません。In particular if the value is the integer `2` you cannot pass `"2"` to `options_for_select` - you must pass `2`. `params`ハッシュから取り出される値はすべて文字列になるので、注意が必要です。

WARNING: [REVIEW]`:include_blank`や`:prompt`が指定されていなくても、選択属性`required`がtrue`になっていると、`:include_blank`は強制的にtrueに設定され、表示の`size`は`1になり、`multiple`はtrueになりません。

ハッシュを使用して任意の値を追加することができます。

```html+erb
<%= options_for_select(
  [
    ['Lisbon', 1, { 'data-size' => '2.8 million' }],
    ['Madrid', 2, { 'data-size' => '3.2 million' }]
  ], 2
) %>

上のコードから以下の出力が得られます。

<option value="1" data-size="2.8 million">Lisbon</option>
<option value="2" selected="selected" data-size="3.2 million">Madrid</option>
...
```

### モデルを扱うセレクトボックス

ほとんどの場合、フォームコントロールは特定のデータベースと結び付けられるものであり、Railsがそのためのヘルパーを提供してくれることを期待するのは当然です。他のフォームヘルパーのときと同じ要領で、モデルを扱う場合には`select_tag`から`_tag`という接尾語を取り除きます。

```ruby
# controller:
@person = Person.new(city_id: 2)
```

```erb
# view:
<%= select(:person, :city_id, [['Lisbon', 1], ['Madrid', 2], ...]) %>
```

第3のパラメータであるオプション配列は、`options_for_select`に渡した引数と同じ種類のものです。このヘルパーのメリットの1つは、正しい街名が既に選ばれている場合、正しい街名が事前選択されているかどうかを気にする必要がないという点です。[REVIEW]Railsは`@person.city_id`属性を読み出してこれらを肩代わりしてくれます。

他のヘルパーのときと同様、`@person`オブジェクトを対象としたフォームビルダーで`select`ヘルパーを使用するのであれば、以下のような文法になります。

```erb
# フォームビルダーに対して選択を行なう
<%= f.select(:city_id, ...) %>
```

`select`ヘルパーにブロックを渡すこともできます。

```erb
<%= f.select(:city_id) do %>
  <% [['Lisbon', 1], ['Madrid', 2]].each do |c| -%>
    <%= content_tag(:option, c.first, value: c.last) %>
  <% end %> <% end %>  ```

WARNING: `select`ヘルパー(および類似の`collection_select`ヘルパー、`select_tag`ヘルパーなど)を使用して`belongs_to`関連付けを設定する場合は、関連付けそのものの名前ではなく、外部キーの名前(上の例であれば`city_id`)を渡す必要があります。If you specify `city` instead of `city_id` Active Record will raise an error along the lines of `ActiveRecord::AssociationTypeMismatch: City(#17815740) expected, got String(#1138750)` when you pass the `params` hash to `Person.new` or `update`. [REVIEW]さらに、属性の編集のみを行なうフォームヘルパーについても注意が必要です。ユーザーが外部キーを直接操作できてしまうとセキュリティ上の問題が生じる可能性があるため、十分注意してください。

### 任意のオブジェクトのコレクションに対してオプションタグを使用する

`options_for_select`を使用してオプションタグを生成する場合、各オプションのテキストと値を含む配列が作成されている必要があります。But what if you had a `City` model (perhaps an Active Record one) and you wanted to generate option tags from a collection of those objects? ひとつの方法は、コレクションをイテレートしてネストした配列を作成することです。

```erb
<% cities_array = City.all.map { |city| [city.name, city.id] } %>
<%= options_for_select(cities_array) %>
```

これはこれでまったく正当な方法ですが、Railsにはもっと簡潔な`options_from_collection_for_select`ヘルパーがあります。このヘルパーは、任意のオブジェクトのコレクションの他に2つの引数 ( **value** オプションと **text** オプションをそれぞれ読み出すためのメソッド名) を取ります。

```erb
<%= options_from_collection_for_select(City.all, :id, :name) %>
```

その名前が示すとおり、このヘルパーが生成するのはオプションタグだけです。実際に動作するセレクトボックスを生成するには、このメソッドを`options_for_select`と併用したときと同様、このメソッドと`select_tag`を併用する必要があります。モデルオブジェクトを使用して作業する場合、`select`を`select_tag`および`options_for_select`と組み合わせた場合と同様、`collection_select`を`select_tag`および`options_from_collection_for_select`と組み合わせます。

```erb
<%= collection_select(:person, :city_id, City.all, :id, :name) %>
```

As with other helpers, if you were to use the `collection_select` helper on a form builder scoped to the `@person` object, the syntax would be:

```erb
<%= f.collection_select(:city_id, City.all, :id, :name) %>
```

要約すると、`options_from_collection_for_select`ヘルパーは「`options_for_select`が`select`するもの」を「`collection_select`する」ということです。

NOTE: `options_for_select`に渡されるペアでは、名前が1番目でidが2番目でしたが、`options_from_collection_for_select`の場合は1番目の引数はvalueメソッドで2番目の引数はtextメソッドです。

### タイムゾーンと国を選択する

Railsでタイムゾーンをサポートするために、ユーザーが今どのタイムゾーンにいるのかを何らかの形でユーザーに尋ねなければなりません。そのためには、`collection_select`ヘルパーを使用して、事前定義済みのTimeZoneオブジェクトのリストからセレクトボックスを作成する必要がありますが、実はこの機能を実現する`time_zone_select`というそれ専用のヘルパーが既に用意されています。

```erb
<%= time_zone_select(:person, :time_zone) %>
```

`time_zone_options_for_select`という類似のヘルパーもあり、こちらではより細かい設定を行なうことができます。これら2つのメソッドに渡せる引数の詳細については、APIドキュメントを参照してください。

以前のRailsでは、`country_select`ヘルパーを使用して国を選択して _いました_ が、この機能は[country_selectプラグイン](https://github.com/stefanpenner/country_select). When using this, be aware that the exclusion or inclusion of certain names from the list can be somewhat controversial (and was the reason this functionality was extracted from Rails).

日付時刻フォームヘルパーを使用する
--------------------------------

HTML5標準の日付/時刻入力フィールドを生成するヘルパーの代りに別の日付/時刻ヘルパーを使用することもできます。いずれにしろ、日付/時刻ヘルパーは以下の2つの点が他のヘルパーと異なっています。

* 日付と時刻を一度に表す入力要素はありません。そのため、年、月、日などの個別のコンポーネントをいくつも使用しなければならず、従って`params`ハッシュ内でも日付時刻は単一の値では表されません。
* 他のヘルパーでは、そのヘルパーが最小限の基本機能を持つ (ベアボーン) ものであるか、あるいはモデルオブジェクトを扱うものであるかを`_tag`接尾語の有無で表します。日付/時刻ヘルパーの場合は、`select_date`、`select_time`、`select_datetime`がベアボーンヘルパーで、`date_select`、`time_select`、`datetime_select`がモデルオブジェクトヘルパーに相当します。

どちらのヘルパーファミリーを使用しても、年・月・日などさまざまなコンポーネントのセレクトボックスを同じように作成できます。

### ベアボーンヘルパー

The `select_*` family of helpers take as their first argument an instance of `Date`, `Time` or `DateTime` that is used as the currently selected value. 現在の日付が使用される場合はこのパラメータを省略できます。以下に例を示します。

```erb
<%= select_date Date.today, prefix: :start_date %>
```

上のコードから以下の出力が得られます(煩雑さを避けるため実際のオプション値を省略しています)。

```html
<select id="start_date_year" name="start_date[year]"> ... </select>
<select id="start_date_month" name="start_date[month]"> ... </select>
<select id="start_date_day" name="start_date[day]"> ... </select>
```

上の入力の結果は`params[:start_date]`に反映され、キーは`:year`、`:month`、`:day`となります。To get an actual `Date`, `Time` or `DateTime` object you would have to extract these values and pass them to the appropriate constructor, for example:

```ruby
Date.civil(params[:start_date][:year].to_i, params[:start_date][:month].to_i, params[:start_date][:day].to_i)
```

`:prefix`オプションは、`params`ハッシュから日付コンポーネントのハッシュを取り出すのに使用されるキーです。これで`start_date`に設定されました。省略すると`date`に設定されます。

### モデルオブジェクトヘルパー

`select_date`ヘルパーはActive Recordオブジェクトの更新・作成を行なうフォームでは扱いにくくなっています。Active Recordは、`param`ハッシュに含まれる要素がそれぞれ1つの属性にのみ対応していることを前提としているからです。
日付/時刻用のモデルオブジェクトヘルパーは、特殊な名前を持つパラメータを送信します。Active Recordはこの特殊な名前を見つけると、それらが他のパラメータと結び付けられているとみなし、モデルのカラムの種類に合ったコンストラクタが与えられているとみなします。以下に例を示します。

```erb
<%= date_select :person, :birth_date %>
```

上のコードから以下の出力が得られます(煩雑さを避けるため実際のオプション値を省略しています)。

```html
<select id="person_birth_date_1i" name="person[birth_date(1i)]"> ... </select>
<select id="person_birth_date_2i" name="person[birth_date(2i)]"> ... </select>
<select id="person_birth_date_3i" name="person[birth_date(3i)]"> ... </select>
```

ここから以下のような`params`ハッシュを得られます。

```ruby
{'person' => {'birth_date(1i)' => '2008', 'birth_date(2i)' => '11', 'birth_date(3i)' => '22'}}
```

上が`Person.new` (または`Person.update`)に与えられると、Active Recordはこれらのパラメータが`birth_date`属性を構成するために使用されなければならないことを理解し、接尾語付きの情報を使用します。この情報は、`Date.civil`などの関数にどのような順序でこれらのパラメータを渡さなければならないかを決定するのに使われます。

### 共通のオプション

どちらのヘルパーファミリーでも、個別のセレクトタグを生成するためのコア機能は共通なので、多くのオプションが同じように使えます。特にRailsでは、どちらのファミリーでも年のオプションはデフォルトで現在の年の前後5年が使用されます。この範囲が適切でない場合は`:start_year`オプションと`:end_year`オプションを使用して上書きできます。利用できるすべてのオプションを知りたい場合は[APIドキュメント](http://api.rubyonrails.org/classes/ActionView/Helpers/DateHelper.html)を参照してください。

経験則から言うと、モデルオブジェクトを扱うのであれば`date_select`を使用するのがよく、その他の場合、たとえば日付でフィルタするなどの検索フォームで使用するのであれば`select_date`を使用するのがよいでしょう。

NOTE: ビルトインのデートピッカー (date picker) は日付と曜日が連動してくれないなど、あまりできがよくないことが多いようです。

### 個別のコンポーネント

日付のうち、たとえば年だけ、月だけのコンポーネントを表示したくなることがあります。Railsでは日付/時刻の個別の要素を扱うための`select_year`、`select_month`、`select_day`、`select_hour`、`select_minute`、`select_second`ヘルパーが用意されています。これらのヘルパーは比較的単純なつくりになっています。By default they 上を実行すると以下が生成されます。 an input field named after the time component (for example, "year" for `select_year`, "month" for `select_month` etc.) although this can be overridden with the `:field_name` option. `:prefix`オプションの動作は`select_date`や`select_time`のときと同じで、デフォルト値も同じです。

The first parameter specifies which value should be selected and can either be an instance of a `Date`, `Time` or `DateTime`, in which case the relevant component will be extracted, or a numerical value. 以下に例を示します。

```erb
<%= select_year(2009) %>
<%= select_year(Time.now) %>
```

現在の年が2009年であれば上のコードの出力結果は同じになり、値は取り出されて`params[:date][:year]`に保存されます。

ファイルのアップロード
---------------

ファイルのアップロードはアプリケーションでよく行われるタスクの1つです (プロフィール写真のアップロードや、処理したいCSVファイルのアップロードなど)。ファイルのアップロードでぜひとも気を付けなければならないのは、出力されるフォームのエンコードは **必ず** "multipart/form-data"でなければならないという点です。`form_for`ヘルパーを使用すれば、この点は自動的に処理されます。`form_tag`を使用してファイルアップロードを行なう場合は、以下の例に示したようにエンコードを明示的に指定しなければなりません。

以下の2つはどちらもファイルアップロードのフォームです。

```erb
<%= form_tag({action: :upload}, multipart: true) do %>
  <%= file_field_tag 'picture' %>
<% end %>

<%= form_for @person do |f| %>
  <%= f.file_field :picture %>
<% end %>
```

Railsでは他と同様、ベアボーンヘルパーの`file_field_tag`とモデル指向の`file_field`が両方提供されています。他のヘルパーと唯一異なる点は、ファイル入力のデフォルト値を設定できないことです(実際、設定する意味がありません)。そしてご想像のとおり、アップロードされたファイルはベアボーンヘルパーの方では`params[:picture]`に保存され、モデル指向のヘルパーの方では`params[:person][:picture]`に保存されます。

### アップロード可能なファイル

The object in the `params` hash is an instance of a subclass of `IO`. Depending on the size of the uploaded file it may in fact be a `StringIO` or an instance of `File` backed by a temporary file. どちらのヘルパーを使用した場合でも、オブジェクトには`original_filename`属性と`content_type`属性が含まれます。`original_filename`属性に含まれる名前は、ユーザーのコンピュータ上にあるファイルの名前です。`content_type`属性には、アップロードの終わったファイルのMIMEタイプが含まれます。以下のスニペットでは、`#{Rails.root}/public/uploads`でアップロードされたコンテンツを、元と同じ名前で保存します(フォームは上の例と同じものを使用したとします)。

```ruby
def upload
  uploaded_io = params[:person][:picture]
  File.open(Rails.root.join('public', 'uploads', uploaded_io.original_filename), 'wb') do |file|
    file.write(uploaded_io.read)
  end
end
```

ファイルがアップロードされた後にはやらなければならないことがたくさんあります。ファイルをどこに保存するのかの決定 (ローカルディスク上か、Amazon S3か、など)、モデルとの関連付けの他に、画像であればサイズの変更やサムネイルの生成が必要になることもあります。これらの事後処理は本ガイドの範疇を超えるのでここでは扱いませんが、これらの処理を助けるライブラリがいくつもあることは知っておいてよいと思います。Two of the better known ones are [CarrierWave](https://github.com/jnicklas/carrierwave) and [Paperclip](https://github.com/thoughtbot/paperclip).

NOTE: ユーザーがファイルを選択しないでアップロードを行なうと、対応するパラメータには空文字列が置かれます。

### Ajaxを扱う

非同期のファイルアップロードフォームの作成は、他のフォームのように`form_for`に`remote: true`を指定すれば済むというわけにはいきません。Ajaxフォームのシリアライズは、ブラウザ内で実行されるJavaScriptによって行われます。そしてブラウザのJavaScriptは(危険を避けるため)ローカルのファイルにアクセスできないようになっているので、JavaScriptからアップロードファイルを読み出すことができません。これを回避する方法として最も一般的なのは、非表示のiframeをフォーム送信の対象として使用することでしょう。

フォームビルダーをカスタマイズする
-------------------------

As mentioned previously the object yielded by `form_for` and `fields_for` is an instance of `FormBuilder` (or a subclass thereof). フォームビルダーは、ある1つのオブジェクトのフォーム要素を表示するために必要なものをカプセル化します。While you can of course write helpers for your forms in the usual way, you can also subclass `FormBuilder` and add the helpers there. 以下に例を示します。

```erb
<%= form_for @person do |f| %>
  <%= text_field_with_label f, :first_name %>
<% end %>
```

上のコードは以下のように置き換えることもできます。

```erb
<%= form_for @person, builder: LabellingFormBuilder do |f| %>
  <%= f.text_field :first_name %>
<% end %>
```

by defining a `LabellingFormBuilder` class similar to the following:

```ruby
class LabellingFormBuilder < ActionView::Helpers::FormBuilder
  def text_field(attribute, options={})
    label(attribute) + super
  end
end
```

このクラスを頻繁に再利用するのであれば、`labeled_form_for`ヘルパーを定義して`builder: LabellingFormBuilder`オプションを自動的に適用するようにしてもよいでしょう。

ここで使用されるフォームビルダーは、以下のコードが実行された時のどうも決定します。

```erb
<%= render partial: f %>
```

If `f` is an instance of `FormBuilder` then this will render the `form` partial, setting the partial's object to the form builder. If the form builder is of class `LabellingFormBuilder` then the `labelling_form` partial would be rendered instead.

パラメータの命名ルールを理解する
------------------------------------------

ここまで説明したように、フォームから受け取る値は`params`ハッシュのトップレベルに置かれるか、他のハッシュの中に入れ子になって含まれます。以下に例を示します。 in a standard `create`
action for a Person model, `params[:person]` would usually be a hash of all the attributes for the person to create. `params`ハッシュには配列やハッシュの配列などを含めることもできます。 hash can also contain arrays, arrays of hashes and so on.

原則として、HTMLフォームはいかなる構造化データについても関知しません。フォームが生成するのはすべて名前と値のペアであり、これらは単なる文字列です。これらのデータをアプリケーション側で参照した時に配列やハッシュになっているのは、Railsで使用されている命名ルールのパラメータのおかげです。

TIP: You may find you can try out examples in this section faster by using the console to directly invoke Rack's parameter parser. 以下に例を示します。

```ruby
Rack::Utils.parse_query "name=fred&phone=0123456789"
# => {"name"=>"fred", "phone"=>"0123456789"}
```

### 基本構造

配列とハッシュは、基本となる2大構造です。ハッシュは、`params`の値にアクセスする時に使用される文法に反映されています。以下に例を示します。 if a form contains:

```html
<input id="person_name" name="person[name]" type="text" value="Henry"/>
```

このとき、`params`ハッシュの内容は以下のようになります。

```erb
{'person' => {'name' => 'Henry'}}
```

従って、コントローラ内で`params[:person][:name]`でアクセスすると、送信された値を取り出すことができます。

ハッシュは、以下のように何階層でも好きなだけネストすることができます。:

```html
<input id="person_address_city" name="person[address][city]" type="text" value="New York"/>
```

上のコードによってできる`params`ハッシュは以下のようになります。

```ruby
{'person' => {'address' => {'city' => 'New York'}}}
```

パラメータ名が重複している場合は、Railsによって無視されます。If the parameter name contains an empty set of square brackets `[]` then they will be accumulated in an array. If you wanted users to be able to input multiple phone numbers, you could place this in the form:

```html
<input name="person[phone_number][]" type="text"/>
<input name="person[phone_number][]" type="text"/>
<input name="person[phone_number][]" type="text"/>
```

This would result in `params[:person][:phone_number]` being an array containing the inputted phone numbers.

### 組み合わせの技法

これらの2つの概念を混ぜ合わせることもできます。One element of a hash might be an array as in the previous example, or you can have an array of hashes. 以下に例を示します。 a form might let you create any number of addresses by repeating the following form fragment

```html
<input name="addresses[][line1]" type="text"/>
<input name="addresses[][line2]" type="text"/>
<input name="addresses[][city]" type="text"/>
```

上のフォームによって`params[:addresses]`ハッシュが作成されます。これは`line1`、`line2`、`city`をキーに持つハッシュとなります。入力された名前が現在のハッシュに既にある場合は、新しいハッシュに値が追加されるようになります。

ただしここで1つ制限があります。ハッシュはいくらでもネストできますが、配列は1階層しか使用できません。Arrays can usually be replaced by hashes; for example, instead of having an array of model objects, one can have a hash of model objects keyed by their id, an array index or some other parameter.

WARNING: 配列パラメータは、`check_box`ヘルパーとの相性がよくありません。HTMLの仕様では、オンになっていないチェックボックスからは値が送信されません。しかし、チェックボックスから常に値が送信される方が何かと便利です。そこで`check_box`ヘルパーは、同じ名前で予備の隠し入力を作成することで、本来送信されないはずのチェックボックス値が見かけ上送信されるようにしています。チェックボックスがオフになっていると隠し入力値だけが送信され、チェックボックスがオンになっていると本来のチェックボックス値と隠し入力値が両方送信されますが、このとき優先されるのは本来のチェックボックス値の方です。従って、このように重複した値送信に対して配列パラメータを使用するとRailsが混乱することがあります。その理由は、入力名が重複している場合はそこで新しい配列要素が作成されるからです。これを回避するためには、`check_box_tag`を使用するか、配列をやめてハッシュを使用してください。

### フォームヘルパーを使用する

前の節ではRailsのフォームヘルパーをまったく使用していませんでした。もちろん、このように入力名を自分でこしらえて`text_field_tag`などのヘルパーに渡してもよいのですが、Railsにはさらに高度なサポートがあります。そのための便利な道具は、`form_for`と`fields_for`の名前パラメータ、そしてヘルパーが引数に取る`:index`オプションの2つです。

複数の住所をそれぞれ編集できるフィールドを持つフォームを作ることもできます。以下に例を示します。

```erb
<%= form_for @person do |person_form| %>
  <%= person_form.text_field :name %>
  <% @person.addresses.each do |address| %>
    <%= person_form.fields_for address, index: address.id do |address_form|%>
      <%= address_form.text_field :city %>
    <% end %>
  <% end %>
<% end %>
```

ここでは1人の人物が2つの住所 (idは23と45) を持てるものとします。これによって得られる出力は以下のようなものになります。

```html
<form accept-charset="UTF-8" action="/people/1" class="edit_person" id="edit_person_1" method="post">
  <input id="person_name" name="person[name]" type="text" />
  <input id="person_address_23_city" name="person[address][23][city]" type="text" />
  <input id="person_address_45_city" name="person[address][45][city]" type="text" />
</form>
```

ここから得られる`params`ハッシュは以下のようになります。

```ruby
{'person' => {'name' => 'Bob', 'address' => {'23' => {'city' => 'Paris'}, '45' => {'city' => 'London'}}}}
```

Railsは、これらの入力がpersonハッシュの一部でなければならないことを認識してくれます。これが可能なのは、最初のフォームビルダーで`fields_for`を呼び出してあるからです。`:index`オプションを指定すると、入力は`person[address][city]`のような名前の代わりに、住所と都市名の間に [ ] で囲まれたインデックスが挿入された名前が使用されます。このようにしておくと、修正すべきAddressレコードを簡単に指定できるので何かと便利です。他の意味を持つ数字を渡したり、文字列や`nil`を渡すこともできます。これらは、作成される配列パラメータの中に置かれます。

入力名の最初の部分(先の例の`person[address]`など)を明示的に示すことで、より複雑なネスティングを作成することもできます。

```erb
<%= fields_for 'person[address][primary]', address, index: address do |address_form| %>
  <%= address_form.text_field :city %>
<% end %>
```

上のコードから以下のような入力が作成されます。

```html
<input id="person_address_primary_1_city" name="person[address][primary][1][city]" type="text" value="bologna" />
```

Railsの一般的なルールとして、最終的な入力名は、`fields_for`や`form_for`に与えられた名前、インデックス値、そして属性名を連結したものになります。, the index value and the name of the attribute. `text_field`などのヘルパーに`:index`オプションを直接渡してもよいのですが、入力コントロールの1つ1つで指定するより、フォームビルダーのレベルで一度指定する方が、たいていの場合繰り返しが少なくて済みます。

名前に[]を追加して`:index`オプションを省略する略記法もあります。以下は`index: address`と指定した場合と同じ結果になります。

```erb
<%= fields_for 'person[address][primary][]', address do |address_form| %>
  <%= address_form.text_field :city %>
<% end %>
```

従って、上の結果はその前の例とまったく同じになります。

Forms to External 参考資料
---------------------------

Rails' form helpers can also be used to build a form for posting data to an external resource. However, at times it can be necessary to set an `authenticity_token` for the resource; this can be done by passing an `authenticity_token: 'your_external_token'` parameter to the `form_tag` options:

```erb
<%= form_tag 'http://farfar.away/form', authenticity_token: 'external_token' do %>
  Form contents
<% end %>
```

Sometimes when submitting data to an external resource, like a payment gateway, the fields that can be used in the form are limited by an external API and it may be undesirable to generate an `authenticity_token`. To not send a token, simply pass `false` to the `:authenticity_token` option:

```erb
<%= form_tag 'http://farfar.away/form', authenticity_token: false do %>
  Form contents
<% end %>
```

`form_for`でも同じ方法が使用できます。

```erb
<%= form_for @invoice, url: external_url, authenticity_token: 'external_token' do |f| %>
  Form contents
<% end %>
```

あるいは、`authenticity_token`フィールドの生成を抑制することもできます。

```erb
<%= form_for @invoice, url: external_url, authenticity_token: false do |f| %>
  Form contents
<% end %>
```

複雑なフォームを作成する
----------------------

最初は単一のオブジェクトを編集していただけのシンプルなフォームも、やがて成長して複雑になるものです。以下に例を示します。 when creating a `Person` you might want to allow the user to (on the same form) create multiple address records (home, work, etc.). 後でPersonを編集するときに、必要に応じて住所の追加・削除・変更が行えるようにする必要もあります。

### モデルを構成する

Active Recordは`accepts_nested_attributes_for`メソッドによってモデルレベルのサポートを行っています。

```ruby
class Person < ActiveRecord::Base
  has_many :addresses
  accepts_nested_attributes_for :addresses
end

class Address < ActiveRecord::Base
  belongs_to :person
end
```

上のコードによって`addresses_attributes=`メソッドが`Person`モデル上に作成され、これを使用して住所の作成・更新・(必要であれば)削除を行なうことができます。

### ネストしたフォーム

ユーザーは以下のフォームを使用して`Person`とそれに関連する複数の住所を作成することができます。

```html+erb
<%= form_for @person do |f| %>
  Addresses:
  <ul>
    <%= f.fields_for :addresses do |addresses_form| %>
      <li>
        <%= addresses_form.label :kind %>
        <%= addresses_form.text_field :kind %>

        <%= addresses_form.label :street %>
        <%= addresses_form.text_field :street %>
      ...
      </li>
    <% end %>
  </ul>
<% end %>
```


ネストした属性が関連付けによって受け入れられると、`fields_for`ヘルパーはその関連付けのすべての要素を一度ずつ出力します。特に、Personに住所が登録されていない場合は何も出力しません。フィールドのセットが少なくとも1つはユーザーに表示されるように、コントローラで1つ以上の空白の子を作成しておくというのはよく行われるパターンです。以下の例では、Personフォームを新たに作成したときに2組の住所フィールドが表示されるようになっています。

```ruby
def new
  @person = Person.new
  2.times { @person.addresses.build}
end
```

`fields_for`ヘルパーはフォームフィールドを1つ生成します。`accepts_nested_attributes_for`ヘルパーが受け取るのはこのようなパラメータの名前です。以下に例を示します。 when creating a user with
2 addresses, the submitted parameters would look like:

```ruby
{
  'person' => {
    'name' => 'John Doe',
    'addresses_attributes' => {
      '0' => {
        'kind' => 'Home',
        'street' => '221b Baker Street'
      },
      '1' => {
        'kind' => 'Office',
        'street' => '31 Spooner Street'
      }
    }
  }
}
```

`:addresses_attributes`ハッシュのキーはここでは重要ではありません。各アドレスのキーが重複していなければそれでよいのです。

関連付けられたオブジェクトが既に保存されている場合、`fields_for`メソッドは、保存されたレコードの`id`を持つ隠し入力を自動的に作成します。`fields_for`に`include_id: false`を渡すことでこの自動生成をオフにできm自動生成をオフにすることがあるとすれば、HTMLが有効でなくなってしまうような場所にinputタグが生成されないようにする場合や、子が`id`を持たないORM (オブジェクトリレーショナルマッピング) を使用したい場合があります。

### コントローラ

コントローラ内でパラメータをモデルに渡す前に、定番の[パラメータのホワイトリストチェック](action_controller_overview.html#strong-parameters)を行いましょう。

```ruby
def create
  @person = Person.new(person_params)
  # ...
end

private
  def person_params
    params.require(:person).permit(:name, addresses_attributes: [:id, :kind, :street])
  end
```

### オブジェクトを削除する

`accepts_nested_attributes_for`に`allow_destroy: true`を渡すことで、関連付けられたオブジェクトをユーザーが削除することを許可できるようになります。

```ruby
class Person < ActiveRecord::Base
  has_many :addresses
  accepts_nested_attributes_for :addresses, allow_destroy: true
end
```

あるオブジェクトの属性のハッシュに、キーが`_destroy`で値が`1`または`true`の組み合わせがあると、そのオブジェクトは削除されます。以下のフォームではユーザーが住所を削除できるようになっています。

```erb
<%= form_for @person do |f| %>
  Addresses:
  <ul>
    <%= f.fields_for :addresses do |addresses_form| %>
      <li>
        <%= addresses_form.check_box :_destroy%>
        <%= addresses_form.label :kind %>
        <%= addresses_form.text_field :kind %>
      ...
      </li>
    <% end %>
  </ul>
<% end %>
```

コントローラ内の、ホワイトリストチェックの終わったパラメータを更新して、`_destroy`フィールドがパラメータに含まれるようにしておくことを忘れないで下さい。

```ruby
def person_params
  params.require(:person).
    permit(:name, addresses_attributes: [:id, :kind, :street, :_destroy])
end
```

### 空のレコードができないようにする

ユーザーが何も入力しなかったフィールドは無視するようにしておく方がやはり便利です。これは、`:reject_if` procを`accepts_nested_attributes_for`に渡すことで制御できます。このprocは、フォームから送信された属性のハッシュ1つ1つについて呼び出されます。このprocが`false`を返す場合、Active Recordはそのハッシュに関連付けられたオブジェクトを作成しません。以下の例では、`kind`属性が設定されている場合にのみ住所オブジェクトを生成します。

```ruby
class Person < ActiveRecord::Base
  has_many :addresses
  accepts_nested_attributes_for :addresses, reject_if: lambda {|attributes| attributes['kind'].blank?}
end
```

代りにシンボル`:all_blank`を渡すこともできます。このシンボルが渡されると、すべての値が空欄のレコードを受け付けなくなるprocが生成されます。ただし`_destroy`の場合はどんな値であっても受け付けます。

### その場でフィールドを追加する

多くのフィールドセットを事前に出力する代わりに、[新しい住所を追加] ボタンを押したときだけこれらのフィールドをその場で追加するようにしたいこともあるでしょう。Rails does not provide any built-in support for this. フィールドセットをその場で生成する場合に気を付けたいのは、関連する配列のキーが重複しないようにすることです。JavaScriptで現在の日時を取得して数ミリ秒の時差からユニークな値を得るのが定番の手法です。