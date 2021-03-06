---
title: "Railsは使いたくないけど，Active Record Migrationは使いたい"
emoji: "🛠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Ruby", "migration", "database", "SQL"]
published: true
---


# Railsは使いたくないけど，Active Record Migrationは使いたい

Active Record Migration は，`ALTER TABLE` などの毎回書き方を忘れる(人による) SQL文を書くことなく，マイグレーションを書くことができて便利ですよね．
また，`change` 関数で定義しておくと，大抵の変更は自動でrollbackもやってくれます．
そのため，Goでサーバー書いてるけど，Active Record Migrationを使いたい！という人は，一定数いると思っています．
（Railsをそのまま使うと，CIでRailsのビルドに時間がボトルネックになるので，，，）

そんな願望を叶えてくれるGemが以下です．

https://github.com/thuss/standalone-migrations

> Rails migrations in non-Rails (and non Ruby) projects.

Rails6 にも対応しています．


# 使ってみる

以下，サンプルの置き場所です．

https://github.com/tanimutomo/standalone-migrations-sample

## 設定

基本的に レポジトリの README.md をみていただければ，そのままなので，ここは簡単にセットアップできるよ，ということを知ってもらうために，雑に流れだけ描きます．
bundleを使います．

1. Gemfile に， `gem "standalone_migrations"` を追加
2. `bundle install`
3. Rakefileに以下を追加
```ruby
require 'standalone_migrations'
StandaloneMigrations::Tasks.load_tasks
```
4. `db/config.yml` を設定


## サンプルのマイグレーション

`users` o2m `articles` という設計のデータベースのサンプルです．
生成のコマンドは，`rake db:new_migration name={migration_name}` です．
（もちろん Rails ではないので， `rails generate {model, migration}` は使えません．）

例
```shell
$ rake db:new_migration name=create_articles
      create  db/migrate/20210306091002_create_articles.rb
```

```ruby
# db/migrate/{timestamp}_create_articles.rb

class CreateArticles < ActiveRecord::Migration[6.0]
  def change
    create_table :articles do |t|
      t.string :title
      t.text :body

      t.timestamps
    end
  end
end


# db/migrate/{timestamp}_create_users.rb

class CreateUsers < ActiveRecord::Migration[6.0]
  def change
    create_table :users do |t|
      t.string :name
      t.text :description

      t.timestamps
    end
  end
end


# db/migrate/{timestamp}_add_user_id_to_articles.rb

class AddUserIdToArticles < ActiveRecord::Migration[6.0]
  def change
    add_column :articles, :user_id, :bigint
    add_index :articles, :user_id
    add_foreign_key :articles, :users
  end
end
```

生成された `db/schema.rb`．
いい感じですね．

```ruby
# db/schema.rb

ActiveRecord::Schema.define(version: 2021_03_06_091500) do

  create_table "articles", options: "ENGINE=InnoDB DEFAULT CHARSET=utf8", force: :cascade do |t|
    t.string "title"
    t.text "body"
    t.datetime "created_at", precision: 6, null: false
    t.datetime "updated_at", precision: 6, null: false
    t.bigint "user_id"
    t.index ["user_id"], name: "index_articles_on_user_id"
  end

  create_table "users", options: "ENGINE=InnoDB DEFAULT CHARSET=utf8", force: :cascade do |t|
    t.string "name"
    t.text "description"
    t.datetime "created_at", precision: 6, null: false
    t.datetime "updated_at", precision: 6, null: false
  end

  add_foreign_key "articles", "users"
end
```

もちろん，Active Recordなので，マイグレーションコマンドや出力もRailsと同じです．

```ruby
$ rake db:migrate
== 20210306091500 AddUserIdToArticles: migrating ==============================
-- add_column(:articles, :user_id, :bigint)
   -> 0.0381s
-- add_index(:articles, :user_id)
   -> 0.0294s
-- add_foreign_key(:articles, :users)
   -> 0.0459s
== 20210306091500 AddUserIdToArticles: migrated (0.1136s) =====================
```

`change` 関数で定義してあるので，rollbackもよしなにやってくれます．

```ruby
$ rake db:rollback
== 20210306091500 AddUserIdToArticles: reverting ==============================
-- remove_foreign_key(:articles, :users)
   -> 0.0327s
-- remove_index(:articles, {:column=>:user_id})
   -> 0.0334s
-- remove_column(:articles, :user_id, :bigint)
   -> 0.0346s
== 20210306091500 AddUserIdToArticles: reverted (0.1037s) =====================
```

statusもいつも通り出せます．
```ruby
$ rake db:migrate:status

database: standalone_migrations_sample

 Status   Migration ID    Migration Name
--------------------------------------------------
   up     20210306091002  Create articles
   up     20210306091409  Create users
  down    20210306091500  Add user id to articles
```


# まとめ

Railsなしで，Active Record Migrationを利用するための，Gem [standalone-migrations](https://github.com/thuss/standalone-migrations) を紹介しました．
これでRailsは必要ないですね！！！（うそです）
簡単に使えるので，ぜひ使ってみてください．
