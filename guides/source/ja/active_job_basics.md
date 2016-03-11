Active Job の基礎
=================

本ガイドでは、バックグラウンドで実行するジョブの作成やキュー登録 (エンキュー: enqueue) 、実行方法について解説します。

このガイドの内容:

* ジョブの作成方法
* ジョブの登録方法
* バックグラウンドでのジョブ実行方法
* アプリケーションから非同期にメールを送信する方法

--------------------------------------------------------------------------------


はじめに
------------

Active Jobは、ジョブを宣言し、それによってバックエンドでさまざまな方法によるキュー操作を実行するためのフレームワークです。これらのジョブでは、定期的なクリーンアップを始めとして、請求書発行やメール配信など、どんなことでも実行できます。これらのジョブをより細かな作業単位に分割して並列実行することもできます。


The Purpose of Active Job
-----------------------------
Active Jobの主要な目的は、Railsアプリを即席で作成した直後でも使用できる、自前のジョブ管理インフラを持つことです。これにより、Delayed JobとResqueなどのように、さまざまなジョブ実行機能のAPIの違いを気にせずにジョブフレームワーク機能やその他のgemを搭載することができるようになります。バックエンドでのキューイング作業では、操作方法以外のことを気にせずに済みます。さらに、ジョブ管理フレームワークを切り替える際にジョブを書き直さずに済みます。


ジョブを作成する
--------------

このセクションでは、ジョブの作成方法とジョブの登録 (enqueue) 方法を手順を追って説明します。

### ジョブを作成する

Active Jobは、ジョブ作成用のRailsジェネレータを提供しています。The following will create a
job in `app/jobs` (with an attached test case under `test/jobs`):

```bash
$ bin/rails generate job guests_cleanup
invoke  test_unit
create    test/jobs/guests_cleanup_job_test.rb
create  app/jobs/guests_cleanup_job.rb
```

以下のようにすると、特定のキューに対してジョブを1つ作成できます。

```bash
$ bin/rails generate job guests_cleanup --queue urgent
```

ジェネレータを使用したくないのであれば、`app/jobs`の下に自分でジョブファイルを作成することもできます。ジョブファイルでは必ず`ActiveJob::Base`を継承してください。

作成されたジョブは以下のようになります。

```ruby
class GuestsCleanupJob < ActiveJob::Base
  queue_as :default

  def perform(*guests)
    # 後で行なう
  end
end
```

Note that you can define `perform` with as many arguments as you want.

### ジョブをキューに登録する

キューへのジョブ登録は以下のように行います。

```ruby
# Enqueue a job to be performed as soon the queuing system is
# free.
GuestsCleanupJob.perform_later guest
```

```ruby
# Enqueue a job to be performed tomorrow at noon.
GuestsCleanupJob.set(wait_until: Date.tomorrow.noon).perform_later(guest)
```

```ruby
# Enqueue a job to be performed 1 week from now.
GuestsCleanupJob.set(wait: 1.week).perform_later(guest)
```

```ruby
# `perform_now` and `perform_later` will call `perform` under the hood so
# you can pass as many arguments as defined in the latter.
GuestsCleanupJob.perform_later(guest1, guest2, filter: 'some_filter')
```

以上で終わりです。

ジョブを実行する
-------------

アダプタが設定されていない場合、ジョブは直ちに実行されます。

### バックエンド

Active Jobには、Sidekiq、Resque、Delayed Jobなどさまざまなキューイングバックエンドに接続できるアダプタがビルトインで用意されています。利用可能な最新のアダプタのリストについては、APIドキュメントの[ActiveJob::QueueAdapters](http://api.rubyonrails.org/classes/ActiveJob/QueueAdapters.html) を参照してください。

### Setting the Backend

You can easily set your queueing backend:

```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
    # Be sure to have the adapter's gem in your Gemfile and follow
    # the adapter's specific installation and deployment instructions.
    config.active_job.queue_adapter = :sidekiq
  end
end
```


キュー
------

多くのアダプタでは複数のキューを扱うことができます。Active Jobを使用することで、特定のキューに入っているジョブをスケジューリングすることができます。

```ruby
class GuestsCleanupJob < ActiveJob::Base
  queue_as :low_priority
  #....
end
```

`application.rb`で以下のように`config.active_job.queue_name_prefix`を使用することで、すべてのジョブでキュー名の前に特定の文字列を追加することができます。

```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
    config.active_job.queue_name_prefix = Rails.env
  end
end

# app/jobs/guests_cleanup.rb
class GuestsCleanupJob < ActiveJob::Base
  queue_as :low_priority
  #....
end

# 以上で、production環境ではproduction_low_priorityというキューでジョブが
# production environment and on staging_low_priority on your staging
#
```

The default queue name prefix delimiter is '\_'.  This can be changed by setting
`config.active_job.queue_name_delimiter` in `application.rb`:

```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
    config.active_job.queue_name_prefix = Rails.env
    config.active_job.queue_name_delimiter = '.'
  end
end

# app/jobs/guests_cleanup.rb
class GuestsCleanupJob < ActiveJob::Base
  queue_as :low_priority
  #....
end

# Now your job will run on queue production.low_priority on your
# production environment and on staging.low_priority on your staging
#
```

If you want more control on what queue a job will be run you can pass a `:queue`
option to `#set`:

```ruby
MyJob.set(queue: :another_queue).perform_later(record)
```

To control the queue from the job level you can pass a block to `#queue_as`. The
block will be executed in the job context (so you can access `self.arguments`)
and you must return the queue name:

```ruby
class ProcessVideoJob < ActiveJob::Base
  queue_as do
    video = self.arguments.first
    if video.owner.premium?
      :premium_videojobs
    else
      :videojobs
    end
  end

  def perform(video)
    # do process video
  end
end

ProcessVideoJob.perform_later(Video.last)
```

NOTE: 設定したキュー名をキューイングバックエンドが「リッスンする」ようにしてください。一部のバックエンドでは、リッスンするキューを指定する必要があるものがあります。


コールバック
---------

Active Jobは、ジョブのライフサイクルでのフックを提供します。これによりコールバックが利用できるので、ジョブのライフサイクルの間に特定のロジックをトリガできます。

### 利用可能なコールバック

* `before_enqueue`
* `around_enqueue`
* `after_enqueue`
* `before_perform`
* `around_perform`
* `after_perform`

### 使用法

```ruby
class GuestsCleanupJob < ActiveJob::Base
  queue_as :default

  before_enqueue do |job|
    # ジョブインスタンスで行なう作業
  end

  around_perform do |job, block|
    # 実行前に行なう作業
    block.call
    # 実行後に行なう作業
  end

  def perform
    # 後で行なう
  end
end
```


Action Mailer
------------

最近のWebアプリケーションでよく実行されるジョブといえば、リクエスト-レスポンスのサイクルの外でメールを送信することでしょう。これにより、ユーザーが送信を待つ必要がなくなります。Active JobはAction Mailerと統合されているので、非同期メール送信を簡単に行えます。

```ruby
# すぐにメール送信したい場合は#deliver_nowを使用
UserMailer.welcome(@user).deliver_now

# Active Jobを使用して後でメール送信したい場合は#deliver_laterを使用
UserMailer.welcome(@user).deliver_later
```


Internationalization
--------------------

Each job uses the `I18n.locale` set when the job was created. Useful if you send
emails asynchronously:

```ruby
I18n.locale = :eo

UserMailer.welcome(@user).deliver_later # Email will be localized to Esparanto.
```


GlobalID
--------

Active JobではGlobalIDがパラメータとしてサポートされています。GlobalIDを使用すると、動作中のActive Recordオブジェクトをジョブに渡す際にクラスとidを指定する必要がありません。クラスとidを指定する従来の方法では、後で明示的にデシリアライズ (deserialize) する必要がありました。従来のジョブが以下のようなものだったとします。

```ruby
class TrashableCleanupJob < ActiveJob::Base
  def perform(trashable_class, trashable_id, depth)
    trashable = trashable_class.constantize.find(trashable_id)
    trashable.cleanup(depth)
  end
end
```

現在は以下のように簡潔に書くことができます。

```ruby
class TrashableCleanupJob < ActiveJob::Base
  def perform(trashable, depth)
    trashable.cleanup(depth)
  end
end
```

This works with any class that mixes in `GlobalID::Identification`, which
by default has been mixed into Active Model classes.


例外
----------

Active Jobでは、ジョブ実行時に発生する例外をキャッチする方法が1つ提供されています。

```ruby

class GuestsCleanupJob < ActiveJob::Base
  queue_as :default

  rescue_from(ActiveRecord::RecordNotFound) do |exception|
   # ここに例外処理を書く
  end

  def perform
    # 後で行なう
  end
end
```