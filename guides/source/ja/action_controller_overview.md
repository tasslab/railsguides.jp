Action Controller の概要
==========================

本ガイドでは、コントローラの動作と、アプリケーションのリクエストサイクルでコントローラがどのように使われるかについて説明します。

このガイドの内容:

* コントローラを経由するリクエストの流れを理解する
* コントローラに渡されるパラメータを制限する方法
* セッションやcookieにデータを保存する理由とその方法
* リクエストの処理中にフィルタを使用してコードを実行する方法
* Action ControllerにビルトインされているHTTP認証
* ユーザーのブラウザにデータを直接ストリーミング送信する方法
* 機密性の高いパラメータをフィルタしてログに出力されないようにする方法
* リクエスト処理中に発生する可能性のある例外の取り扱い

--------------------------------------------------------------------------------

コントローラの役割
--------------------------

Action Controllerは、MVCモデルのCに相当します。リクエストを処理するコントローラがルーティング設定によって指名されると、コントローラはリクエストの意味を理解し、適切な出力を行なうための責任を持ちます。幸い、これらの処理はほとんどAction Controllerが行ってくれます。しかも吟味された規則を使用しているので、リクエストの処理は可能な限り素直な方法で行われます。

従来のいわゆる[RESTful](http://ja.wikipedia.org/wiki/REST) なアプリケーションでは、コントローラはリクエストを受け取り (この部分は開発者からは見えないようになっています)、データをモデルから取得したりモデルに保存するなどの作業を行い、最後にビューを使用してHTML出力を生成する、という役割を担います。自分のコントローラのつくりがこれと少し違っていたりするかもしれませんが、気にする必要はありません。ここではコントローラの一般的な使用法について説明しています。

コントローラは、モデルとビューの間に立って仲介を行っているという見方もできます。コントローラはモデルのデータをビューで使用できるようにすることで、データをビューで表示したり、入力されたデータでモデルを更新したりします。

NOTE: ルーティングの詳細については、本ガイドの[Railsのルーティング](routing.html)を参照してください。

コントローラの命名規則
----------------------------

Railsのコントローラ名(ここでは"Controller"という文字は除きます)は、基本的に名前の最後の部分に「複数形」を使用します。ただしこれは絶対的に守らなければならないというものではありません (実際 `ApplicationController`はApplicationが単数になっています)。たとえば、`ClientsController`の方が`ClientController`より好ましく、`SiteAdminsController`は`SiteAdminController`や`SitesAdminsController`よりも好ましい、といった具合です。

しかし、この規則には従っておくことをお勧めします。その理由は、`resources`などのデフォルトのルーティングジェネレータがそのまま利用できるのと、URLやパスヘルパーの用法がアプリケーション全体で一貫するからです。コントローラ名の最後が複数形になっていないと、たとえば`resources`で簡単に一括ルーティングできず、`:path`や`:controller`をいちいち指定しなければなりません。詳細については、[レイアウト・レンダリングガイド](layouts_and_rendering.html)を参照してください。

NOTE: モデルの命名規則は「単数形」であり、コントローラの命名規則とは異なります。


メソッドとアクション
-------------------

Railsのコントローラは、`ApplicationController`を継承したRubyのクラスであり、他のクラスと同様のメソッドが使用できます。アプリケーションがブラウザからのリクエストを受け取ると、ルーティングによってコントローラとアクションが指名され、Railsはそれに応じてコントローラのインスタンスを生成し、アクション名と同じ名前のメソッドを実行します。

```ruby
class ClientsController < ApplicationController
  def new
  end
end
```

例として、クライアントを1人追加するためにブラウザでアプリケーションの`/clients/new`にアクセスすると、Railsは`ClientsController`のインスタンスを作成して`new`メソッドを実行します。ここでご注目いただきたいのは、`new`メソッドの内容が空であるにもかかわらず正常に動作するという点です。これは、Railsでは`new`アクションで特に指定のない場合には`new.html.erb`ビューを描画するようになっているからです。ビューで`@client`インスタンス変数にアクセスできるようにするために、`new`メソッドで`Client`を新規作成し、@clientに保存してみましょう。

```ruby
def new
  @client = Client.new
end
```

この詳細については、[レイアウト・レンダリングガイド](layouts_and_rendering.html)で説明されています。

`ApplicationController`は、便利なメソッドが多数定義されている`ActionController::Base`を継承しています。本ガイドではそれらの一部について説明しますが、もっと知りたい場合はAPIドキュメントかRailsのソースコードを参照してください。

publicなメソッドでないとアクションとして呼び出すことはできません。補助メソッドやフィルタのような、アクションとして扱われたくないメソッドはprivateを指定するのが定石です。

パラメータ
----------

コントローラのアクションでは、ユーザーから送信されたデータやその他のパラメータにアクセスして何か作業を行なうのが普通です。Railsに限らず、一般にWebアプリケーションでは2種類のパラメータを扱うことができます。1番目は、URLの一部として送信されるパラメータで、「クエリ文字列パラメータ」と呼ばれます。クエリ文字列は、常にURLの"?"の後に置かれます。2番目のパラメータは、「POSTデータ」と呼ばれるものです。POSTデータは通常、ユーザーが記入したHTMLフォームから受け取ります。これがPOSTデータと呼ばれているのは、HTTP POSTリクエストの一部として送信されるからです。Railsでは、クエリ文字列パラメータの受け取り方とPOSTデータの受け取り方に違いはありません。どちらもコントローラ内では`params`という名前のハッシュでアクセスできます。

```ruby
class ClientsController < ApplicationController
  # このアクションではクエリ文字列パラメータが使用されています
  # 送信側でHTTP GETリクエストが使用されているためです
  # ただしパラメータにアクセスするうえでは下との違いは生じませんThe URL for
  # 有効な顧客リストを得るため、このアクションへのURLは以下のようになっています
  # clients: /clients?status=activated
  def index
    if params[:status] == "activated"
      @clients = Client.activated
    else
      @clients = Client.inactivated
    end
  end

  # このアクションではPOSTパラメータが使用されています。このパラメータは通常
  # ユーザーが送信したHTMLフォームが元になります。The URL for
  # これはRESTfulなアクセスであり、URLは"/clients"となります。
  # データはURLではなくリクエストのbodyの一部として送信されます。
  def create
  @client = Client.new(params[:client])
    if @client.save
      redirect_to @client
    else
      # 以下の行ではデフォルトのレンダリング動作を上書きします。
      # 本来は"create"ビューが描画されます。
      render "new"
    end
  end
end
```

### Hash and Array パラメータ

`params`ハッシュは、一次元のキー・値ペアしか格納できないということはありません。配列や、ネストしたハッシュを格納することもできます。値の配列をフォームから送信したい場合は、以下のように空の角かっこ[]をキー名に追加してください。

```
GET /clients?ids[]=1&ids[]=2&ids[]=3
```

NOTE: "["と"]"はURLで使用できない文字なので、この例の実際のURLは "/clients?ids%5b%5d=1&ids%5b%5d=2&ids%5b%5d=3"のようになります。これについては、ブラウザが自動的に面倒を見てくれ、さらにRailsはパラメータ受け取り時に自動的に復元してくれるので、通常は気にする必要はありません。ただし、何らかの理由でサーバーにリクエストを手動送信しなければならない場合には、このことを思い出す必要があります。

The value of `params[:ids]` will now be `["1", "2", "3"]`. もう一つ知っておいて欲しいのは、パラメータの値はすべて「文字列型」であることです。Railsはパラメータの型を推測することもしなければ、型変換も行いませんので、必要であれば自分で型を変換する必要があります。パラメータの数字を`to_i`で整数に変換するのはよく行われます。

NOTE: `params`の中に`[]`、`[nil]`、`[nil, nil, ...]`という値があると、すべて自動的に`nil`に置き換えられます。この動作は、デフォルトのセキュリティ上の理由にもとづいています。詳細については [セキュリティガイド](security.html#安全でないクエリ生成) を参照してください。

フォームから、中かっこ[]の中にキー名を含めたハッシュを送信するには以下のようにします。

```html
<form accept-charset="UTF-8" action="/clients" method="post">
  <input type="text" name="client[name]" value="Acme" />
  <input type="text" name="client[phone]" value="12345" />
  <input type="text" name="client[address][postcode]" value="12345" />
  <input type="text" name="client[address][city]" value="Carrot City" />
</form>
```

このフォームを送信すると、`params[:client]`の値は`{ "name" => "Acme", "phone" => "12345", "address" => { "postcode" => "12345", "city" => "Carrot City" } }`になります。`params[:client][:address]`のハッシュがネストしていることにご注目ください。

この`params`ハッシュは、実は`ActiveSupport::HashWithIndifferentAccess`のインスタンスです。これはハッシュのように振る舞いますが、キーとしてシンボルと文字列のどちらを指定してもよい (交換可能) 点が異なります。

### JSONパラメータ

Webサービスアプリケーションを開発していると、パラメータをJSONフォーマットで受け取れたら便利なのにと思うことがあります。Railsでは、リクエストの"Content-Type"に"application/json"が指定されていれば、自動的にパラメータを`params`ハッシュに変換してくれます。以後は通常の`params`ハッシュと同様に操作できます。

たとえば、以下のJSONコンテンツを送信したとします。

```json
{ "company": { "name": "acme", "address": "123 Carrot Street" } }
```

`params[:company]`で受け取る値は`{ "name" => "acme", "address" => "123 Carrot Street" }`になります。

同様に、初期化設定で`config.wrap_parameters`をオンにした場合や、コントローラで`wrap_parameters`を呼び出した場合、JSONパラメータのルート要素を安全に取り除くことができます。このパラメータは、デフォルトではコントローラ名に応じて複製およびラップされます。従って、上のパラメータは以下のように書けます。

```json
{ "name": "acme", "address": "123 Carrot Street" }
```

データの送信先が`CompaniesController`であるとすると、以下のように`:company`というキーでラップされます。

```ruby
{ name: "acme", address: "123 Carrot Street", company: { name: "acme", address: "123 Carrot Street" } }
```

キー名のカスタマイズや、ラップしたい特定のパラメータについては[APIドキュメント](http://api.rubyonrails.org/classes/ActionController/ParamsWrapper.html)を参照してください。

NOTE: 従来のXMLパラメータ解析のサポートは、`actionpack-xml_parser`というgemに書き出されました。

### Routing パラメータ

`params`ハッシュに必ず含まれるキーは`:controller`キーと`:action`キーです。ただしこれらの値には直接アクセスせず、`controller_name`と`action_name`という専用のメソッドを使用してください。ルーティングで定義されるその他の値パラメータ (`id`など) にもアクセスできます。例として、「有効」と「無効」のいずれかで表される顧客のリストについて考えてみましょう。「プリティな」URLに含まれる`:status`パラメータを捉えるためのルートを1つ追加してみましょう。

```ruby
get '/clients/:status' => 'clients#index', foo: 'bar'
```

この場合、ブラウザで`/clients/active`というURLを開くと、`params[:status]`が"active" (有効) に設定されます。このルーティングを使用すると、あたかもクエリ文字列で渡したかのように`params[:foo]`にも"bar"が設定されます。同様に、`params[:action]`には"index"が含まれます。

### `default_url_options`

You can set global default parameters for URL generation by defining a method called `default_url_options` in your controller. このようなメソッドは、必要なデフォルト値を持つハッシュを返さねばならず、キーはシンボルでなければなりません。

```ruby
class ApplicationController < ActionController::Base
  def default_url_options
    { locale: I18n.locale }
  end
end
```

These options will be used as a starting point when generating URLs, so it's possible they'll be overridden by the options passed in `url_for` calls.

If you define `default_url_options` in `ApplicationController`, as in the example above, it would be used for all URL generation. このメソッドを特定のコントローラで定義すれば、そのコントローラで生成されるURLにだけ影響します。

### Strong パラメータ

With strong parameters, Action Controller parameters are forbidden to
be used in Active Model mass assignments until they have been
whitelisted. これは、多くの属性を一度に更新したいときに、どの属性の更新を許可し、どの属性の更新を禁止するかを明示的に決定しなければならないことを意味します。大雑把にすべての属性の更新をまとめて許可してしまうと、外部に公開する必要のない属性まで誤って公開されてしまう可能性が生じますので、そのような事態を防ぐために行います。

さらに、パラメータの属性には「必須 (required)」を指定することもでき、事前に定義したraise/rescueフローを実行して400 Bad Requestで終了するようにすることもできます。

```ruby
class PeopleController < ActionController::Base
  # 以下のコードはActiveModel::ForbiddenAttributes例外を発生します
  # 明示的な許可を行なわずに、パラメータを一括で渡してしまう
  # 危険な「マスアサインメント」が行われているからです。
  def create
    Person.create(params[:person])
  end

  # 以下のコードは、パラメータにpersonキーがあれば成功します。
  # personキーがない場合は
  # ActionController::ParameterMissing例外を発生します。
  # この例外はActionController::Baseにキャッチされ、
  # 400 Bad Requestを返します。
  def update
    person = current_account.people.find(params[:id])
    person.update!(person_params)
    redirect_to person
  end

  private
    # Using a private method to encapsulate the permissible parameters
    # これは非常によい手法であり、createとupdateの両方で使いまわすことで
    # 同じ許可を与えることができます。また、許可する属性をユーザーごとにチェックするよう
    # このメソッドを特殊化することもできます。
    def person_params
  params.require(:person).permit(:name, :age)
    end
end
```

#### 許可されたスカラー値

Given

```ruby
params.permit(:id)
```

`:id`キーが`params`にあり、それの値の型が許可済みスカラーがあれば、ホワイトリストチェックをパスします。でなければ、このキーはフィルタによって除外されます。従って、ハッシュやその他のオブジェクトを外部から注入することはできなくなります。

許可されるスカラー型は`String`、`Symbol`、`NilClass`、`Numeric`、`TrueClass`、`FalseClass`、`Date`、`Time`、`DateTime`、`StringIO`、`IO`、`ActionDispatch::Http::UploadedFile`、`Rack::Test::UploadedFile`です。

`params`の値が、許可されたスカラー値の配列でなければならないことを宣言するには、以下のようにキーを空の配列にマップします。

```ruby
params.permit(id: [])
```

パラメータのハッシュ全体をホワイトリスト化したい場合は、`permit!`メソッドを使用できます。

```ruby
params.require(:log_entry).permit!
```

This will mark the `:log_entry` parameters hash and any sub-hash of it
permitted. ただし、`permit!`は属性を一括で許可してしまうものなので、くれぐれも慎重に使用してください。現在のモデルはもちろんのこと、将来属性が追加されたときにそこにマスアサインメントの脆弱性が生じる可能性があるからです。

#### Nested パラメータ

ネストしたパラメータに対しても以下のように許可を与えることができます。

```ruby
params.permit(:name, { emails: [] },
              friends: [ :name,
                         { family: [ :name ], hobbies: [] }])
```

この宣言では、`name`、`emails`、`friends`属性がホワイトリスト化されます。ここでは、`emails`は許可を受けたスカラー値の配列であることが期待され、`friends`は特定の属性を持つリソースの配列であることが期待され、どちらも`name`属性(許可を受けたあらゆるスカラー値を受け付ける)を持ちます。また、`hobbies`属性(許可を受けたスカラー値の配列)を持ち、`family`属性(同じく、許可を受けたあらゆるスカラー値を受け付ける)を持つことも期待されます。

#### その他の事例

今度は`new`アクションでも許可済み属性を使用したいところです。しかし今度は`new`を呼び出す時点ではルートキーがないので、ルートキーに対して`require`を指定することができないという問題があります。

```ruby
# `fetch`を使用することでデフォルト値を提供し、
# the Strong パラメータ API from there.
params.fetch(:blog, {}).permit(:title, :author)
```

`accepts_nested_attributes_for`メソッドを使用すると、関連付けられたレコードの更新や削除を行えます。この動作は、`id`と`_destroy`パラメータに基づきます。

```ruby
# :id と :_destroyを許可します。
params.require(:author).permit(:name, books_attributes: [:title, :id, :_destroy])
```

整数のキーを持つハッシュは異なる方法で処理されます。これらは、あたかも直接の子オブジェクトであるかのように属性を宣言することができます。`has_many`関連付けとともに`accepts_nested_attributes_for`メソッドを使用すると、これらの種類のパラメータを取得できます。

```ruby
# 以下のデータをホワイトリスト化
# {"book" => {"title" => "Some Book",
#             "chapters_attributes" => { "1" => {"title" => "First Chapter"},
#                                        "2" => {"title" => "Second Chapter"}}}}

params.require(:book).permit(:title, chapters_attributes: [:title])
```

#### Outside the Scope of Strong パラメータ

strong parameter APIは、最も一般的な使用状況を念頭に置いて設計されています。つまり、あらゆるホワイトリスト化問題を扱えるような万能性があるわけではないということです。しかし、このAPIを自分のコードに混在させることで、状況に対応しやすくすることはできます。

次のような状況を想像してみましょう。製品名と、その製品名に関連する任意のデータを表すパラメータがあるとします。そして、この製品名もデータハッシュ全体もまとめてホワイトリスト化したいとします。strong parameters APIは、任意のキーを持つネストしたハッシュ全体を直接ホワイトリスト化することはしませんが、ネストしたハッシュのキーを使用して、ホワイトリスト化する対象を宣言することはできます。

```ruby
def product_params
  params.require(:product).permit(:name, data: params[:product][:data].try(:keys))
end
```

セッション
-------

Railsアプリケーションにはユーザーごとにセッションが設定されます。前のリクエストの情報を次のリクエストでも利用するためにセッションに少量のデータが保存されます。セッションはコントローラとビューでのみ使用可能です。また、以下のように複数のストレージから選ぶことができます。

* `ActionDispatch::セッション::CookieStore` - Stores everything on the client.
* `ActionDispatch::セッション::CacheStore` - Stores the data in the Rails cache.
* `ActionDispatch::セッション::ActiveRecordStore` - Stores the data in a database using Active Record. (require `activerecord-session_store` gem).
* `ActionDispatch::セッション::MemCacheStore` - Stores the data in a memcached cluster (this is a legacy implementation; consider using CacheStore instead).

すべてのセッションにおいて、セッションごとに独自IDをcookieに保存します (注意: RailsではセッションIDをURLで渡すことはセキュリティ上の危険があるため許可されません。セッションIDはcookieで渡さなくてはなりません)。

多くのセッションストアでは、このIDは単にサーバー上のセッションデータ (データベーステーブルなど) を検索するために使用されています。ただしCookieStoreは例外的にcookie自身にすべてのセッションデータを保存します(セッションIDも必要であれば利用可能です)。そしてRailsではCookieStoreがデフォルトで使用され、かつRailsでの推奨セッションストアでもあります。CookieStoreの利点は、非常に軽量であることと、新規Webアプリケーションでセッションを利用するための準備がまったく不要である点です。cookieデータは改竄防止のために暗号署名が与えられています。さらにcookie自身も暗号化されているので、内容を他人に読まれないようになっています。(改ざんされたcookieはRailsが拒否します)

CookieStoreには約4KBのデータを保存できます。他のセッションストアに比べて少量ですが、通常はこれで十分です。セッションに大量のデータを保存することは、利用するセッションストアの種類にかかわらずお勧めできません。特に、セッションに複雑なオブジェクト (モデルインスタンスなどの基本的なRubyオブジェクトでないもの) を保存することはお勧めできません。このようなことをすると、サーバーがリクエスト間でセッションを再編成できずにエラーになることがあります。

If your user sessions don't store critical data or don't need to be around for long periods (for instance if you just use the flash for messaging), you can consider using `ActionDispatch::セッション::CacheStore`. この方式では、Webアプリケーションに設定されているキャッシュ実装を利用してセッションを保存します。この方法のよい点は、既存のキャッシュインフラをそのまま利用してセッションを保存できることと、管理用の設定を追加する必要がないことです。この方法の欠点はセッションが短命になることです。セッションはいつでも消える可能性があります。

セッションストレージの詳細については[セキュリティガイド](security.html)を参照してください。

別のセッションメカニズムが必要な場合は、`config/initializers/session_store.rb`ファイルを変更することで切り替えられます。

```ruby
# デフォルトのcookieベースのセッションに代えてデータベースセッションを
# 使用する場合は、機密情報を保存しないこと。
# (セッションテーブルの作成は"rails g active_record:session_migration"で行なう)
# Rails.application.config.session_store :active_record_store
```

Railsは、セッションデータに署名するときにセッションキー(=cookieの名前)を設定します。この動作も`config/initializers/session_store.rb`で変更できます。

```ruby
# このファイルを変更後サーバーを必ず再起動してください。
Rails.application.config.session_store :cookie_store, key: '_your_app_session'
```

`:domain`キーを渡して、cookieで使用するドメイン名を指定することもできます。

```ruby
# このファイルを変更後サーバーを必ず再起動してください。
Rails.application.config.session_store :cookie_store, key: '_your_app_session', domain: ".example.com"
```

Railsはセッションデータの署名に使用する秘密鍵を (CookieStore用に) 設定します。この秘密鍵は`config/secrets.yml`で変更できます。

```ruby
# このファイルを変更後サーバーを必ず再起動してください。

# Your secret key is used for verifying the integrity of signed cookies.
# If you change this key, all old signed cookies will become invalid!

# Make sure the secret is at least 30 characters and all random,
# no regular words or you'll be exposed to dictionary attacks.
# You can use `rake secret` to generate a secure secret key.

# Make sure the secrets in this file are kept private
# if you're sharing your code publicly.

development:
  secret_key_base: a75d...

test:
  secret_key_base: 492f...

# Do not keep production secrets in the repository,
# instead read values from the environment.
production:
  secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
```

NOTE: `CookieStore`を使用中に秘密鍵を変更すると、既存のセッションがすべて無効になります。

### Accessing the セッション

コントローラ内では`session`インスタンスメソッドを使用してセッションにアクセスできます。

NOTE: セッションs are lazily loaded. アクションのコードでセッションにアクセスしなかった場合、セッションは読込されません。従って、アクセスしていないのであればセッションを無効にする必要はまったくありません。アクセスしないことで既にセッションは無効になっています。

セッション values are stored using key/value pairs like a hash:

```ruby
class ApplicationController < ActionController::Base

  private

  # キー付きのセッションに保存されたidでユーザーを検索する
  # :current_user_id はRailsアプリケーションでユーザーログインを扱う際の定番の方法。
  # ログインするとセッション値が設定され、
  # ログアウトするとセッション値が削除される。
  def current_user
    @_current_user ||= session[:current_user_id] &&
      User.find_by(id: session[:current_user_id])
  end
end
```

セッションを使用して何かを行なうのであれば、ハッシュに似たキーにセッションを割り当てます。

```ruby
class LoginsController < ApplicationController
  # "Create" a login, aka "log the user in"
  def create
    if user = User.authenticate(params[:username], params[:password])
      # セッションのuser idを保存し、
      # 今後のリクエストで使用できるようにする
      session[:current_user_id] = user.id
      redirect_to root_url
    end
  end
end
```

セッションからデータの一部を削除したい場合は、キーに`nil`を割り当てます。

```ruby
class LoginsController < ApplicationController
  # ログインを削除する (=ログアウト)
  def destroy
    # セッションidからuser idを削除する
    @_current_user = session[:current_user_id] = nil
    redirect_to root_url
  end
end
```

To reset the entire session, use `reset_session`.

### Flash

flashはセッションの中の特殊な部分であり、リクエストごとにクリアされます。この特徴から、flashは「直後のリクエスト」でのみ参照可能になります。これはエラーメッセージを渡したりするのに便利です。

flashのアクセス方法はセッションとほとんど同じで、ハッシュとしてアクセスします (これを[FlashHash](http://api.rubyonrails.org/classes/ActionDispatch/Flash/FlashHash.html) インスタンスと呼びます)。

例として、ログアウトする動作を扱ってみましょう。コントローラでflashを使用することで、次のリクエストで表示するメッセージを送信できます。

```ruby
class LoginsController < ApplicationController
  def destroy
    session[:current_user_id] = nil
    flash[:notice] = "You have successfully logged out."
    redirect_to root_url
  end
end
```

flashメッセージはリダイレクトの一部として割り当てることもできることにご注目ください。オプションとして`:notice`、`:alert`の他に、一般的な`:flash`を使用することもできます。

```ruby
redirect_to root_url, notice: "You have successfully logged out."
redirect_to root_url, alert: "You're stuck here!"
redirect_to root_url, flash: { referral_code: 1234 }
```

この`destroy`アクションでは、アプリケーションの`root_url`にリダイレクトし、そこでメッセージを表示します。flashメッセージは、直前のアクションでflashメッセージにどのようなメッセージを設定していたかにかかわらず、次に行われるアクションだけですべて決まりますのでご注意ください。Railsアプリケーションのレイアウトでは、flashを使用して警告や通知を表示するのが通例です。

```erb
<html>
  <!-- <head/> -->
  <body>
    <% flash.each do |name, msg| -%>
      <%= content_tag :div, msg, class: name %>
    <% end -%>

    <!-- 以下略 -->
  </body>
</html>
```

このように、アクションで通知(notice)や警告(alert)メッセージを設置すると、レイアウト側で自動的にそのメッセージが表示されます。

flashには、通知や警告に限らず、セッションに保存可能なものであれば何でも保存できます。

```erb
<% if flash[:just_signed_up] %>
  <p class="welcome">Welcome to our site!</p>
<% end %>
```

flashの値を別のリクエストにも引き継ぎたい場合は、`keep`メソッドを使用します。

```ruby
class MainController < ApplicationController
  # このアクションはroot_urlに対応しており、このアクションに対する
  # すべてのリクエストをUsersController#indexにリダイレクトしたいとします。
  # あるアクションでflashを設定してこのindexアクションにリダイレクトすると、
  # 別のリダイレクトが発生した場合にはflashは消えてしまいます。
  # ここで'keep'を使用すると別のリクエストでflashが消えないようになります。
  def index
    # すべてのflash値を保持する
    flash.keep

    # キーを指定して値を保持することもできる
    # flash.keep(:notice)
    redirect_to users_url
  end
end
```

#### `flash.now`

デフォルトでは、flashに値を追加すると直後のリクエストでその値を利用できますが、場合によっては、次のリクエストを待たずに同じリクエスト内でこれらのflash値にアクセスしたくなることがあります。たとえば、`create`アクションに失敗してリソースが保存されなかった場合に、`new`テンプレートを直接描画するとします。このとき新しいリクエストは行われませんが、この状態でもflashを使用してメッセージを表示したいこともあります。このような場合、`flash.now`を使用すれば通常の`flash`と同じ要領でメッセージを表示できます。

```ruby
class ClientsController < ApplicationController
  def create
  @client = Client.new(params[:client])
    if @client.save
      # ...
    else
      flash.now[:error] = "Could not save client"
      render action: "new"
    end
  end
end
```

Cookies
-------

Webアプリケーションでは、cookieと呼ばれる少量のデータをクライアントのブラウザに保存することができます。HTTPでは基本的にリクエストとリクエストの間には何の関連もありませんが、cookieを使用することでリクエスト同士の間で (あるいはセッション同士の間であっても) このデータが保持されるようになります。Railsでは`cookies`メソッドを使用してcookieに簡単にアクセスできます。アクセス方法はセッションの場合とよく似ていて、ハッシュのように動作します。

```ruby
class CommentsController < ApplicationController
  def new
    # cookieにコメント作者名が残っていたらフィールドに自動入力する
    @comment = Comment.new(author: cookies[:commenter_name])
  end

  def create
    @comment = Comment.new(params[:comment])
    if @comment.save
      flash[:notice] = "Thanks for your comment!"
      if params[:remember_name]
        # コメント作者名を保存する
        cookies[:commenter_name] = @comment.author
      else
        # コメント作者名がcookieに残っていたら削除する
        cookies.delete(:commenter_name)
      end
      redirect_to @comment.article
    else
      render action: "new"
    end
  end
end
```

セッションを削除する場合はキーに`nil`を指定することで削除しましたが、cookieを削除する場合はこの方法ではなく、`cookies.delete(:key)`を使用してください。

Railsでは、機密データ保存のために署名済みcookie jarと暗号化cookie jarを利用することもできます。署名済みcookie jarでは、暗号化した署名をcookie値に追加することで、cookieの改竄を防ぎます。暗号化cookie jarでは、署名の追加と共に、値自体を暗号化してエンドユーザーに読まれることのないようにします。
Refer to the [API documentation](http://api.rubyonrails.org/classes/ActionDispatch/Cookies.html)
for more details.

これらの特殊なcookieではシリアライザを使用して値を文字列に変換して保存し、読み込み時に逆変換(deserialize)を行ってRubyオブジェクトに戻しています。

使用するシリアライザを指定することもできます。

```ruby
Rails.application.config.action_dispatch.cookies_serializer = :json
```

新規アプリケーションのデフォルトシリアライザは`:json`です。既存のcookieが残っている旧アプリケーションとの互換性のため、`serializer`オプションに何も指定されていない場合は`:marshal`が使用されます。

シリアライザのオプションに`:hybrid`を指定することもできます。これを指定すると、`Marshal`でシリアライズされた既存のcookieも透過的にデシリアライズし、保存時に`JSON`フォーマットで保存し直します。これは、既存のアプリケーションを`:json`シリアライザに移行するときに便利です。

`load`メソッドと`dump`メソッドに応答するカスタムのシリアライザを指定することもできます。

```ruby
Rails.application.config.action_dispatch.cookies_serializer = MyCustomSerializer
```

`:json`または`:hybrid`シリアライザを使用する場合、一部のRubyオブジェクトがJSONとしてシリアライズされない可能性があることにご注意ください。たとえば、`Date`オブジェクトや`Time`オブジェクトはstringsとしてシリアライズされ、`Hash`のキーはstringに変換されます。

```ruby
class CookiesController < ApplicationController
  def set_cookie
    cookies.encrypted[:expiration_date] = Date.tomorrow # => Thu, 20 Mar 2014
    redirect_to action: 'read_cookie'
  end

  def read_cookie
    cookies.encrypted[:expiration_date] # => "2014-03-20"
  end
end
```

cookieには文字列や数字などの単純なデータだけを保存することをお勧めします。
If you have to store complex objects, you would need to handle the conversion
manually when reading the values on subsequent requests.

cookieセッションストアを使用する場合、`session`や`flash`ハッシュについてもこのことは該当します。

XMLとJSONデータを描画する
---------------------------

ActionControllerのおかげで、`XML`データや`JSON`データの出力 (レンダリング) は非常に簡単に行えます。scaffoldを使用して生成したコントローラは以下のようになっているはずです。

```ruby
class UsersController < ApplicationController
  def index
    @users = User.all
    respond_to do |format|
      format.html # index.html.erb
      format.xml  { render xml: @users}
      format.json { render json: @users}
    end
  end
end
```

上のコードでは、`render xml: @users.to_xml`ではなく`render xml: @users`となっていることにご注目ください。Railsは、オブジェクトが文字列型でない場合は自動的に`to_xml`を呼んでくれます。

フィルタ
-------

フィルタ are methods that are run before, after or "around" a controller action.

フィルタ are inherited, so if you set a filter on `ApplicationController`, it will be run on every controller in your application.

"Before"フィルタは、リクエストサイクルを止めてしまう可能性があることに留意してください。"before"フィルタのよくある使われ方として、ユーザーがアクションを実行する前にログインを要求するというのがあります。このフィルタメソッドは以下のような感じになるでしょう。

```ruby
class ApplicationController < ActionController::Base
  before_action :require_login

  private

  def require_login
    unless logged_in?
      flash[:error] = "You must be logged in to access this section"
      redirect_to new_login_url # halts request cycle
    end
  end
end
```

このメソッドはエラーメッセージをflashに保存し、ユーザーがログインしていない場合にはログインフォームにリダイレクトするというシンプルなものです。"before"フィルタによって出力またはリダイレクトが行われると、このアクションは実行されません。フィルタの実行後に実行されるようスケジュールされた追加のフィルタがある場合、これらもキャンセルされます。

この例ではフィルタを`ApplicationController`に追加したので、これを継承するすべてのコントローラが影響を受けます。つまり、アプリケーションのあらゆる機能についてログインが要求されることになります。当然ですが、アプリケーションのあらゆる画面で認証を要求してしまうと、認証に必要なログイン画面まで表示できなくなるという困った事態になってしまいます。従って、このようにすべてのコントローラやアクションでログイン要求を設定すべきではありません。`skip_before_action`メソッドを使用すれば、特定のアクションでフィルタを止めることができます。

```ruby
class LoginsController < ApplicationController
  skip_before_action :require_login, only: [:new, :create]
end
```

上のようにすることで、`LoginsController`の`new`アクションと`create`アクションをこれまでどおり認証不要にすることができました。特定のアクションについてだけフィルタをスキップしたい場合には`:only`オプションを使用します。逆に特定のアクションについてだけフィルタをスキップしたくない場合は`:except`オプションを使用します。These options can be used when adding filters too, so you can add a filter which only runs for selected actions in the first place.

### After フィルタ and Around フィルタ

"before"フィルタ以外に、アクションの実行後に実行されるフィルタや、実行前実行後の両方で実行されるフィルタを使用することもできます。

"after"フィルタは"before"フィルタと似ていますが、"after"フィルタの場合はアクションは既に実行済みであり、クライアントに送信されようとしている応答データにアクセスできる点が"before"フィルタとは異なります。当然ながら、"after"フィルタをどのように書いても、アクションの実行が中断するようなことはありません。

"around"フィルタを使用する場合は、フィルタ内のどこかで必ず`yield`を実行して、関連付けられたアクションを実行してやる義務が生じます。これはRackミドルウェアの動作と似ています。

たとえば、何らかの変更に際して承認ワークフローがあるWebサイトを考えてみましょう。管理者はこれらの変更内容を簡単にプレビューし、トランザクション内で承認できるとします。

```ruby
class ChangesController < ApplicationController
  around_action :wrap_in_transaction, only: :show

  private

  def wrap_in_transaction
    ActiveRecord::Base.transaction do
      begin
        yield
      ensure
        raise ActiveRecord::Rollback
      end
    end
  end
end
```

"around"フィルタの場合、画面出力 (レンダリング) もその作業に含まれていることにご注意ください。特に、上の例で言うとビュー自身がデータベースから (スコープなどを使用して) 読み出しを行うとすると、その読み出しはトランザクション内で行われ、データはプレビューに表示されます。

あえてyieldを実行せず、自分で応答を作成するという方法もあります。この場合、アクションは実行されません。

### Other Ways to Use フィルタ

While the most common way to use filters is by creating private methods and using *_action to add them, there are two other ways to do the same thing.

The first is to use a block directly with the *\_action methods. このブロックはコントローラを引数として受け取ります。さきほどの`require_login`フィルタを書き換えて、ブロックを使用するようにします。

```ruby
class ApplicationController < ActionController::Base
  before_action do |controller|
    unless controller.send(:logged_in?)
      flash[:error] = "You must be logged in to access this section"
      redirect_to new_login_url
    end
  end
end
```

Note that the filter in this case uses `send` because the `logged_in?` method is private and the filter is not run in the scope of the controller. この方法は、特定のフィルタを実装する方法としては推奨されませんが、もっとシンプルな場合には役に立つことがあるかもしれません。

2番目の方法はクラスを使用してフィルタを扱うものです (実際には、正しいメソッドに応答するオブジェクトであれば何でも構いません)。他の2つの方法で実装すると読みやすくならず、再利用も困難になるような複雑な場合に有用です。例として、ログインフィルタをクラスで書き換えてみましょう。

```ruby
class ApplicationController < ActionController::Base
  before_action LoginFilter
end

class LoginFilter
  def self.before(controller)
    unless controller.send(:logged_in?)
      controller.flash[:error] = "You must be logged in to access this section"
      controller.redirect_to controller.new_login_url
    end
  end
end
```

繰り返しますが、この例はフィルタとして理想的なものではありません。その理由は、このフィルタがコントローラのスコープで動作せず、コントローラが引数として渡されるからです。このフィルタクラスには、フィルタと同じ名前のメソッドを実装する必要があります。従って、`before_action`フィルタの場合、クラスに`before`メソッドを実装しなければならない、という具合です。`around`メソッド内では`yield`を呼んでアクションを実行してやる必要があります。

リクエストフォージェリからの保護
--------------------------

クロスサイトリクエストフォージェリは攻撃手法の一種です。邪悪なWebサイトがユーザーをだまし、攻撃目標となるWebサイトへの危険なリクエストを知らないうちに作成させるというものです。攻撃者は標的ユーザーに関する知識や権限を持っていなくても、目標サイトに対してデータの追加/変更/削除を行わせることができてしまいます。

この攻撃を防ぐために必要な手段の第一歩は、「create/update/destroyのような破壊的な操作に対して絶対にGETリクエストでアクセスできない」ようにすることです。WebアプリケーションがRESTful規則に従っていれば、これは守られているはずです。しかし、邪悪なWebサイトはGET以外のリクエストを目標サイトに送信することなら簡単にできてしまいます。リクエストフォージェリの保護はまさにここを守るためのものです。文字どおり、リクエストの偽造(forgery)から保護するのです。

その具体的な保護方法は、推測不可能なトークンをすべてのリクエストに追加することです。このトークンはサーバーだけが知っています。これにより、リクエストに含まれているトークンが不正であればアクセスが拒否されます。

以下のようなフォームを試しに生成してみます。

```erb
<%= form_for @user do |f| %>
  <%= f.text_field :username %>
  <%= f.text_field :password %>
<% end %>
```

以下のようにトークンが隠しフィールドに追加されている様子がわかります。

```html
<form accept-charset="UTF-8" action="/users/1" method="post">
<input type="hidden"
       value="67250ab105eb5ad10851c00a5621854a23af5489"
       name="authenticity_token"/>
<!-- fields -->
</form>
```

Railsでは、[formヘルパー](form_helpers.html)を使用して生成されたあらゆるフォームにトークンを追加します。従って、この攻撃を心配する必要はほとんどありません。formヘルパーを使用せずに手作りした場合や、別の理由でトークンが必要な場合には、`form_authenticity_token`メソッドでトークンを生成できます。

`form_authenticity_token`メソッドは、有効な認証トークンを生成します。このメソッドは、カスタムAjax呼び出しなどのように、Railsが自動的にトークンを追加してくれない箇所で使用するのに便利です。

本ガイドの[セキュリティガイド](security.html)では、この話題を含む多くのセキュリティ問題について言及しており、いずれもWebアプリケーションを開発するうえで必読です。

リクエストオブジェクトとレスポンスオブジェクト
--------------------------------

すべてのコントローラには、現在実行中のリクエストサイクルに関連するリクエストオブジェクトとレスポンスオブジェクトを指す、2つのアクセサメソッドがあります。`request`メソッドは`AbstractRequest`クラスのインスタンスを含み、`response`メソッドは、今クライアントに戻されようとしている内容を表すレスポンスオブジェクトを返します。

### `request`オブジェクト

リクエストオブジェクトには、クライアントブラウザから返されるリクエストに関する有用な情報が多数含まれています。利用可能なメソッドをすべて知りたい場合は[APIドキュメント](http://api.rubyonrails.org/classes/ActionDispatch/Request.html)を参照してください。その中から、このオブジェクトでアクセス可能なメソッドを紹介します。

| `request`のプロパティ                     | 目的                                                                          |
| ----------------------------------------- | -------------------------------------------------------------------------------- |
| host                                      | リクエストで使用されているホスト名                                              |
| domain(n=2)                               | ホスト名の右 (TLD:トップレベルドメイン) から数えて`n`番目のセグメント            |
| format                                    | クライアントからリクエストされたContent-Type                                        |
| method                                    | リクエストで使用されたHTTPメソッド                                            |
| get?, post?, patch?, put?, delete?, head? | HTTPメソッドがGET/POST/PATCH/PUT/DELETE/HEADのいずれかの場合にtrueを返す               |
| headers                                   | リクエストに関連付けられたヘッダーを含むハッシュを返す               |
| port                                      | リクエストで使用されたポート番号 (整数)                                  |
| protocol                                  | プロトコル名に"://"を加えたものを返す ("http://"など) |
| query_string                              | URLの一部で使用されているクエリ文字 ("?"より後の部分)                    |
| remote_ip                                 | クライアントのipアドレス                                                    |
| url                                       | リクエストで使用されているURL全体                                             |

#### `path_parameters`、`query_parameters`、`request_parameters`

Rails collects all of the parameters sent along with the request in the `params` hash, whether they are sent as part of the query string or the post body. requestオブジェクトには3つのアクセサがあり、パラメータの由来に応じたアクセスを行なうこともできます。`query_parameters`ハッシュにはクエリ文字列として送信されたパラメータが含まれます。`request_parameters`ハッシュにはPOST bodyの一部として送信されたパラメータが含まれます。`path_parameters`には、ルーティング機構によって特定のコントローラとアクションへのパスの一部であると認識されたパラメータが含まれます。

### `response`オブジェクト

responseオブジェクトはアクションが実行されるときに構築され、クライアントに送り返されるデータを描画 (レンダリング) するものなので、responseオブジェクトを直接使用することは通常ありません。しかし、たとえばafter filter内などでresponseオブジェクトを直接操作できれば便利です。responseオブジェクトのアクセサメソッドがセッターを持っていれば、これを使用してresponseオブジェクトの値を直接変更できます。

| `response`のプロパティ | 目的                                                                                             |
| ---------------------- | --------------------------------------------------------------------------------------------------- |
| body                   | This is the string of data being sent back to the client. This is most often HTML.                  |
| status                 | The HTTP status code for the response, like 200 for a successful request or 404 for file not found. |
| location               | The URL the client is being redirected to, if any.                                                  |
| content_type           | The content type of the response.                                                                   |
| charset                | The character set being used for the response. Default is "utf-8".                                  |
| headers                | Headers used for the response.                                                                      |

#### Setting Custom Headers

If you want to set custom headers for a response then `response.headers` is the place to do it. The headers attribute is a hash which maps header names to their values, and Rails will set some of them automatically. If you want to add or change a header, just assign it to `response.headers` this way:

```ruby
response.headers["Content-Type"] = "application/pdf"
```

Note: in the above case it would make more sense to use the `content_type` setter directly.

HTTP Authentications
--------------------

Rails comes with two built-in HTTP authentication mechanisms:

* Basic Authentication
* Digest Authentication

### HTTP Basic Authentication

HTTP basic authentication is an authentication scheme that is supported by the majority of browsers and other HTTP clients. As an example, consider an administration section which will only be available by entering a username and a password into the browser's HTTP basic dialog window. Using the built-in authentication is quite easy and only requires you to use one method, `http_basic_authenticate_with`.

```ruby
class AdminsController < ApplicationController
  http_basic_authenticate_with name: "humbaba", password: "5baa61e4"
end
```

With this in place, you can create namespaced controllers that inherit from `AdminsController`. The filter will thus be run for all actions in those controllers, protecting them with HTTP basic authentication.

### HTTP Digest Authentication

HTTP digest authentication is superior to the basic authentication as it does not require the client to send an unencrypted password over the network (though HTTP basic authentication is safe over HTTPS). Using digest authentication with Rails is quite easy and only requires using one method, `authenticate_or_request_with_http_digest`.

```ruby
class AdminsController < ApplicationController
  USERS = { "lifo" => "world" }

  before_action :authenticate

  private

    def authenticate
      authenticate_or_request_with_http_digest do |username|
        USERS[username]
      end
    end
end
```

As seen in the example above, the `authenticate_or_request_with_http_digest` block takes only one argument - the username. And the block returns the password. Returning `false` or `nil` from the `authenticate_or_request_with_http_digest` will cause authentication failure.

Streaming and File Downloads
----------------------------

Sometimes you may want to send a file to the user instead of rendering an HTML page. All controllers in Rails have the `send_data` and the `send_file` methods, which will both stream data to the client. `send_file` is a convenience method that lets you provide the name of a file on the disk and it will stream the contents of that file for you.

To stream data to the client, use `send_data`:

```ruby
require "prawn"
class ClientsController < ApplicationController
  # Generates a PDF document with information on the client and
  # returns it. The user will get the PDF as a file download.
  def download_pdf
    client = Client.find(params[:id])
    send_data generate_pdf(client),
              filename: "#{client.name}.pdf",
              type: "application/pdf"
  end

  private

    def generate_pdf(client)
      Prawn::Document.new do
        text client.name, align: :center
        text "Address: #{client.address}"
        text "Email: #{client.email}"
      end.render
    end
end
```

The `download_pdf` action in the example above will call a private method which actually generates the PDF document and returns it as a string. This string will then be streamed to the client as a file download and a filename will be suggested to the user. Sometimes when streaming files to the user, you may not want them to download the file. Take images, for example, which can be embedded into HTML pages. To tell the browser a file is not meant to be downloaded, you can set the `:disposition` option to "inline". The opposite and default value for this option is "attachment".

### Sending Files

If you want to send a file that already exists on disk, use the `send_file` method.

```ruby
class ClientsController < ApplicationController
  # Stream a file that has already been generated and stored on disk.
  def download_pdf
    client = Client.find(params[:id])
    send_file("#{Rails.root}/files/clients/#{client.id}.pdf",
              filename: "#{client.name}.pdf",
              type: "application/pdf")
  end
end
```

This will read and stream the file 4kB at the time, avoiding loading the entire file into memory at once. You can turn off streaming with the `:stream` option or adjust the block size with the `:buffer_size` option.

If `:type` is not specified, it will be guessed from the file extension specified in `:filename`. If the content type is not registered for the extension, `application/octet-stream` will be used.

WARNING: Be careful when using data coming from the client (params, cookies, etc.) to locate the file on disk, as this is a security risk that might allow someone to gain access to files they are not meant to.

TIP: It is not recommended that you stream static files through Rails if you can instead keep them in a public folder on your web server. It is much more efficient to let the user download the file directly using Apache or another web server, keeping the request from unnecessarily going through the whole Rails stack.

### RESTful Downloads

While `send_data` works just fine, if you are creating a RESTful application having separate actions for file downloads is usually not necessary. In REST terminology, the PDF file from the example above can be considered just another representation of the client resource. Rails provides an easy and quite sleek way of doing "RESTful downloads". Here's how you can rewrite the example so that the PDF download is a part of the `show` action, without any streaming:

```ruby
class ClientsController < ApplicationController
  # The user can request to receive this resource as HTML or PDF.
  def show
    @client = Client.find(params[:id])

    respond_to do |format|
      format.html
      format.pdf { render pdf: generate_pdf(@client) }
    end
  end
end
```

In order for this example to work, you have to add the PDF MIME type to Rails. This can be done by adding the following line to the file `config/initializers/mime_types.rb`:

```ruby
Mime::Type.register "application/pdf", :pdf
```

NOTE: Configuration files are not reloaded on each request, so you have to restart the server in order for their changes to take effect.

Now the user can request to get a PDF version of a client just by adding ".pdf" to the URL:

```bash
GET /clients/1.pdf
```

### Live Streaming of Arbitrary Data

Rails allows you to stream more than just files. In fact, you can stream anything
you would like in a response object. The `ActionController::Live` module allows
you to create a persistent connection with a browser. Using this module, you will
be able to send arbitrary data to the browser at specific points in time.

#### Incorporating Live Streaming

Including `ActionController::Live` inside of your controller class will provide
all actions inside of the controller the ability to stream data. You can mix in
the module like so:

```ruby
class MyController < ActionController::Base
  include ActionController::Live

  def stream
    response.headers['Content-Type'] = 'text/event-stream'
    100.times {
      response.stream.write "hello world\n"
      sleep 1
    }
  ensure
    response.stream.close
  end
end
```

The above code will keep a persistent connection with the browser and send 100
messages of `"hello world\n"`, each one second apart.

There are a couple of things to notice in the above example. We need to make
sure to close the response stream. Forgetting to close the stream will leave
the socket open forever. We also have to set the content type to `text/event-stream`
before we write to the response stream. This is because headers cannot be written
after the response has been committed (when `response.committed` returns a truthy
value), which occurs when you `write` or `commit` the response stream.

#### Example Usage

Let's suppose that you were making a Karaoke machine and a user wants to get the
lyrics for a particular song. Each `Song` has a particular number of lines and
each line takes time `num_beats` to finish singing.

If we wanted to return the lyrics in Karaoke fashion (only sending the line when
the singer has finished the previous line), then we could use `ActionController::Live`
as follows:

```ruby
class LyricsController < ActionController::Base
  include ActionController::Live

  def show
    response.headers['Content-Type'] = 'text/event-stream'
    song = Song.find(params[:id])

    song.each do |line|
      response.stream.write line.lyrics
      sleep line.num_beats
    end
  ensure
    response.stream.close
  end
end
```

The above code sends the next line only after the singer has completed the previous
line.

#### Streaming Considerations

Streaming arbitrary data is an extremely powerful tool. As shown in the previous
examples, you can choose when and what to send across a response stream. However,
you should also note the following things:

* Each response stream creates a new thread and copies over the thread local
  variables from the original thread. Having too many thread local variables can
  negatively impact performance. Similarly, a large number of threads can also
  hinder performance.
* Failing to close the response stream will leave the corresponding socket open
  forever. Make sure to call `close` whenever you are using a response stream.
* WEBrick servers buffer all responses, and so including `ActionController::Live`
  will not work. You must use a web server which does not automatically buffer
  responses.

Log Filtering
-------------

Rails keeps a log file for each environment in the `log` folder. These are extremely useful when debugging what's actually going on in your application, but in a live application you may not want every bit of information to be stored in the log file.

### Parameters Filtering

You can filter out sensitive request parameters from your log files by appending them to `config.filter_parameters` in the application configuration. These parameters will be marked [FILTERED] in the log.

```ruby
config.filter_parameters << :password
```

### Redirects Filtering

Sometimes it's desirable to filter out from log files some sensitive locations your application is redirecting to.
You can do that by using the `config.filter_redirect` configuration option:

```ruby
config.filter_redirect << 's3.amazonaws.com'
```

You can set it to a String, a Regexp, or an array of both.

```ruby
config.filter_redirect.concat ['s3.amazonaws.com', /private_path/]
```

Matching URLs will be marked as '[FILTERED]'.

Rescue
------

Most likely your application is going to contain bugs or otherwise throw an exception that needs to be handled. For example, if the user follows a link to a resource that no longer exists in the database, Active Record will throw the `ActiveRecord::RecordNotFound` exception.

Rails' default exception handling displays a "500 Server Error" message for all exceptions. If the request was made locally, a nice traceback and some added information gets displayed so you can figure out what went wrong and deal with it. If the request was remote Rails will just display a simple "500 Server Error" message to the user, or a "404 Not Found" if there was a routing error or a record could not be found. Sometimes you might want to customize how these errors are caught and how they're displayed to the user. There are several levels of exception handling available in a Rails application:

### The Default 500 and 404 Templates

By default a production application will render either a 404 or a 500 error message. These messages are contained in static HTML files in the `public` folder, in `404.html` and `500.html` respectively. You can customize these files to add some extra information and layout, but remember that they are static; i.e. you can't use RHTML or layouts in them, just plain HTML.

### `rescue_from`

If you want to do something a bit more elaborate when catching errors, you can use `rescue_from`, which handles exceptions of a certain type (or multiple types) in an entire controller and its subclasses.

When an exception occurs which is caught by a `rescue_from` directive, the exception object is passed to the handler. The handler can be a method or a `Proc` object passed to the `:with` option. You can also use a block directly instead of an explicit `Proc` object.

Here's how you can use `rescue_from` to intercept all `ActiveRecord::RecordNotFound` errors and do something with them.

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :record_not_found

  private

    def record_not_found
      render plain: "404 Not Found", status: 404
    end
end
```

Of course, this example is anything but elaborate and doesn't improve on the default exception handling at all, but once you can catch all those exceptions you're free to do whatever you want with them. For example, you could create custom exception classes that will be thrown when a user doesn't have access to a certain section of your application:

```ruby
class ApplicationController < ActionController::Base
  rescue_from User::NotAuthorized, with: :user_not_authorized

  private

    def user_not_authorized
      flash[:error] = "You don't have access to this section."
      redirect_to :back
    end
end

class ClientsController < ApplicationController
  # Check that the user has the right authorization to access clients.
  before_action :check_authorization

  # Note how the actions don't have to worry about all the auth stuff.
  def edit
    @client = Client.find(params[:id])
  end

  private

    # If the user is not authorized, just throw the exception.
    def check_authorization
      raise User::NotAuthorized unless current_user.admin?
    end
end
```

WARNING: You shouldn't do `rescue_from Exception` or `rescue_from StandardError` unless you have a particular reason as it will cause serious side-effects (e.g. you won't be able to see exception details and tracebacks during development).

NOTE: Certain exceptions are only rescuable from the `ApplicationController` class, as they are raised before the controller gets initialized and the action gets executed. See Pratik Naik's [article](http://m.onkey.org/2008/7/20/rescue-from-dispatching) on the subject for more information.

Force HTTPS protocol
--------------------

Sometime you might want to force a particular controller to only be accessible via an HTTPS protocol for security reasons. You can use the `force_ssl` method in your controller to enforce that:

```ruby
class DinnerController
  force_ssl
end
```

Just like the filter, you could also pass `:only` and `:except` to enforce the secure connection only to specific actions:

```ruby
class DinnerController
  force_ssl only: :cheeseburger
  # or
  force_ssl except: :cheeseburger
end
```

Please note that if you find yourself adding `force_ssl` to many controllers, you may want to force the whole application to use HTTPS instead. In that case, you can set the `config.force_ssl` in your environment file.