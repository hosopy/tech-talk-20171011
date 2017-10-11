## O/Rマッパーでやらかしたこと

#### 3社合同反省会
##### 株式会社ガラパゴス 細羽啓司

---

## 自己紹介

* 細羽啓司 / Keishi Hosoba
* 株式会社ガラパゴス
* スマートフォンアプリとバックエンドの開発
* GitHub: https://github.com/hosopy

---

## 今日話すこと

* O/Rマッパー(というかRDB)に関連したやらかし話
* 注) 悪いのはO/Rマッパーではなく、使い方とプロセスです

---

## O/Rマッパー

---

## ActiveRecord (Ruby)

---

## やらかし事件簿

* integer桁あふれ事件
* unscoped事件

---

## 事件簿1
### integer桁あふれ事件

---

## 事象

* とあるスマホアプリのバックエンド
* PostgreSQLのinteger型の主キー(id)が桁あふれを起こす
* Sequenceは元気に値(bigint)を生成するが、INSERTがひたすらエラー
* 特定のWebAPIが500連発

---

```
PG::NumericValueOutOfRange
PG::NumericValueOutOfRange
PG::NumericValueOutOfRange
PG::NumericValueOutOfRange
PG::NumericValueOutOfRange
PG::NumericValueOutOfRange
PG::NumericValueOutOfRange
PG::NumericValueOutOfRange
...
```

---

## サービスへの影響

* iOS/Androidアプリで、一部データのアプリ⇔クラウド同期が失敗
* 同期失敗のインジケーターが表示され続ける
* 自動実行とリトライ機構をおかげで、バックエンド復旧後のデータ復旧は自動で解決できた

---

## 復旧

* 特定のWebAPIををnginxでシャットアウト
    * テーブルへの読み書きを排除
    * 仕様的にサービス全体は止めずに済んだ
* idの型をbigintに変更

---

## ALTER TABLE is 無理

```sql
ALTER TABLE items ALTER COLUMN id TYPE bigint
```

- 142,099,472 |
- INDEX x 1 |
- UNIQUE INDEX x 1 |
- FOREIGN KEY x 1 |
- RDS(m4.xlarge)のスナップショットで実験 |
- 終わる気配なし |

---

## コピーしてリネーム

```sql
-- idをbigintにキャストしながらテーブルをコピー
CREATE TABLE items_copy AS SELECT cast(id AS bigint) AS id,... FROM items;
-- 主キー,INDEXを付与
ALTER TABLE items_copy add CONSTRAINT items_copy_pkey PRIMARY KEY(id);
CREATE UNIQUE INDEX index_items_copy_on_... ON items_copy (...);
CREATE INDEX index_items_copy_on_... ON items_copy USING BTREE (...);
-- 各種制約を付与
ALTER TABLE items_copy ADD CONSTRAINT fk_items_copy_user_id FOREIGN KEY (user_id) REFERENCES users(id);
ALTER TABLE items_copy ALTER COLUMN ... set DEFAULT 0;
...
-- テーブルをリネーム
ALTER TABLE items RENAME to items_org;
ALTER TABLE items_copy RENAME TO items;
```

@[2](COPY : 約13分)
@[4](PRIMARY KEY : 約14分)
@[5](UNIQUE INDEX : 約2時間)
@[6](INDEX : 約8分)
@[8](FOREIGN KEY : 約3分)
@[9-10](DEFAULT, NOT NULL : 約3分)
@[12-13](RENAME : 秒殺)

---

## 反省

---

## とりあえずID

* 設計時に主キーや型について深く考慮せず
* O/Rマッパー任せであった

![SQL](https://images-na.ssl-images-amazon.com/images/I/41qHKrFZi0L._SX258_BO1,204,203,200_.jpg)

---

## ActiveRecord::Migration

```ruby:db/migrate/20171007041502_create_items.rb
class CreateItems < ActiveRecord::Migration
  def change
    create_table :items do |t|
      t.string :title, null: false
    end
  end
end
```

---

## テーブル

```
dev=# \d items;
                              Table "public.items"
 Column |       Type        |                     Modifiers
--------+-------------------+----------------------------------------------------
 id     | integer           | not null default nextval('items_id_seq'::regclass)
 title  | character varying | not null
Indexes:
    "items_pkey" PRIMARY KEY, btree (id)
```

@[5](integer)

---

## シーケンス

```
dev=# \d items_id_seq;
        Sequence "public.items_id_seq"
    Column     |  Type   |        Value
---------------+---------+---------------------
 sequence_name | name    | items_id_seq
 last_value    | bigint  | 1
 start_value   | bigint  | 1
 increment_by  | bigint  | 1
 max_value     | bigint  | 9223372036854775807
 min_value     | bigint  | 1
 cache_value   | bigint  | 1
 log_cnt       | bigint  | 0
 is_cycled     | boolean | f
 is_called     | boolean | f
Owned by: public.items.id
```

@[6-10](bigint)

---

## ActiveRecord 5.1 以前

```ruby:db/migrate/20171007041502_create_items.rb
class CreateItems < ActiveRecord::Migration
  def change
    create_table :items, id: :bigserial do |t|
      t.string :title, null: false
    end
  end
end
```

@[3](id: bigserialでidはbigintとして定義される)

---

## ActiveRecord 5.1 以降

* デフォルトでbigint
* Rails 5.1: Default Primary Keys Are Now BIGINT
    * http://www.mccartie.com/2016/12/05/rails-5.1.html

---

## 事件簿2
### unscoped事件

---

## 事象

* とあるスマホアプリのバックエンド
* 外部キー条件の抜けたSELECTが実行され、意図しないレコードを参照した意図しない集計処理(バッチ)が実行された
* Full Scan発生でRDSのCPU使用率が爆発

---

## サービスへの影響

* バッチ処理
* フロントで一部の集計データがおかしなことに
* RDSのCPU使用率が爆発したことで、APIレスポンスの遅延が増加

---

## 復旧

* バグ修正後にバッチ処理を再実行して復旧
* 幸いにしてクリティカルなデータではなかった(相対的に...)

---

* Relation
* Association
* Scope
* Default Scope
* unscoped

---

## Relation

```ruby
# Model
class User < ActiveRecord::Base
end

# Relation
active_users = User.where(status: 0)

active_users.class
=> Activity::ActiveRecord_Relation

active_users.to_sql
=> SELECT "users".* FROM "users" WHERE "users"."status" = 0
```

---

## Association

```ruby
class User < ActiveRecord::Base
  has_many :posts
end

class Post < ActiveRecord::Base
  belongs_to :user
end

user = User.find(1)

user.posts.to_sql
=> SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = 1
```

---

## Scope

```ruby
class User < ActiveRecord::Base
  scope :active, -> { where(status: 0) }
end

User.where(status: 0).to_sql
=> SELECT "users".* FROM "users" WHERE "users"."status" = 0

User.active.to_sql
=> SELECT "users".* FROM "users" WHERE "users"."status" = 0
```

---

## Chain

```ruby
class User < ActiveRecord::Base
  scope :active, -> { where(status: 0) }
  scope :male, -> { where(gender: 0) }
end

User.active.male.to_sql
=> SELECT "users".* FROM "users" WHERE "users"."status" = 0
       AND "users"."gender" = 0
```

---

## Merge

```ruby
class User < ActiveRecord::Base
  has_many :posts
  scope :active, -> { where(status: 0) }
end

class Post < ActiveRecord::Base
  belongs_to :user
  scope :published, -> { where(published: true) }
end

Post.published.joins(:user).merge(User.active).to_sql
=> SELECT "posts".* FROM "posts" INNER JOIN "users" ON ...
       WHERE "posts"."deleted" = 'f' AND "posts"."published" = 't'
       AND "users"."status" = 0 AND "users"."gender" = 0
```

---

## Default Scope

```ruby
class Post < ActiveRecord::Base
  belongs_to :user

  default_scope -> { where(deleted: false) }
  scope :published, -> { where(published: true) }
end

Post.published
=> SELECT "posts".* FROM "posts" WHERE "posts"."published" = 't'
       AND "posts"."deleted" = 'f'
```

個人的にはDefault Scope否定派

---

## unscoped

```ruby
class Post < ActiveRecord::Base
  belongs_to :user

  default_scope -> { where(deleted: false) }
  scope :published, -> { where(published: true) }
end

Post.published
=> SELECT "posts".* FROM "posts" WHERE "posts"."published" = 't'
       AND "posts"."deleted" = 'f'

Post.unscoped.published
=> SELECT "posts".* FROM "posts" WHERE "posts"."published" = 't'
```

---

## そして事件が発生

```ruby
user = User.find(1)

user.posts.published.to_sql
=> SELECT "posts".* FROM "posts" WHERE "posts"."deleted" = 'f'
       AND "posts"."user_id" = 1 AND "posts"."published" = 't'

user.posts.unscoped.published.to_sql
=> SELECT "posts".* FROM "posts" WHERE "posts"."published" = 't'

# 脳内では...
=> SELECT "posts".* FROM "posts" WHERE "posts"."user_id" = 1
       AND "posts"."published" = 't'
```

@[1-4](deletedを条件から外したい)
@[1-6](unscopedしてしまえ！)
@[1-7](全Relationが解除されてuser_idの条件が消える)
@[7-10](脳内ではこれを期待していた...)

---

## 反省

---

## デプロイされるまで

- コードレビューがパス |
    - なんとなく正しそうなんだもん... |
- 単体テストがパス |
    - beforeで別ユーザーのデータも作っとけば... |
- ステージング環境でのQAテストもパス |
- 本番デプロイ |
- フロントで一部の表示データがおかしい |
- RDSのCPU使用率が爆発 |

---

## 再現性のある防ぎ方？

---

## ステージング環境のDB

* 常に本番相当(規模)のデータを用意しておくべき
* 少なくともRDSのSlowQueryやCPU使用率の爆発で気づけたはず

---

## まとめ

* 主キーについて考えよう
* ステージング環境のDBは本番相当規模にしておこう

---

## バックエンドエンジニア募集中!

https://www.wantedly.com/projects/99060

---

## ご清聴ありがとうございました

## 本日のスライド
### https://gitpitch.com/hosopy/tech-talk-20171011