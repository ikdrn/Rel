# Rel — Relation Language
## 次世代データベース言語 設計仕様書 v0.2

**バージョン**: 0.2.0-draft
**設計日**: 2026-03-06
**ステータス**: RFC

---

## なぜSQLを捨てるのか

SQLの「本当の問題」はキーワードの順序ではない。

**問題の本質: SQLは「テーブル走査」という誤った世界観に基づいている。**

現実のデータはこういう構造をしている:

```
User ─── orders ──→ Order ─── items ──→ Product ─── category ──→ Category
  │
  └── address ──→ Address
  └── tags ──→ Tag (多対多)
```

しかしSQLはこのグラフを、まず全部バラバラの2次元テーブルに押し込み、
クエリのたびにJOINで「関係を手動で再構築する」という歪な設計になっている。

Relは**エンティティとその関係をそのまま記述する**。
テーブルもJOINも、SELECT/FROM/WHEREも存在しない。

---

## 目次

1. [言語の世界観](#1-言語の世界観)
2. [基本構文：エンティティパターン](#2-基本構文エンティティパターン)
3. [リレーション探索](#3-リレーション探索)
4. [形状の抽出](#4-形状の抽出)
5. [集計](#5-集計)
6. [ソートとページング](#6-ソートとページング)
7. [スキーマ定義](#7-スキーマ定義)
8. [リレーション定義](#8-リレーション定義)
9. [型システム](#9-型システム)
10. [名前付きパターン（再利用）](#10-名前付きパターン再利用)
11. [モジュール](#11-モジュール)
12. [書き込み操作](#12-書き込み操作)
13. [実行モデル](#13-実行モデル)
14. [分散・ストリーム](#14-分散ストリーム)
15. [SQLとの対比](#15-sqlとの対比)

---

## 1. 言語の世界観

### データはグラフである

```
┌────────────────────────────────────────────────────────────────┐
│  SQLの世界観                    Relの世界観                    │
│                                                                │
│  users テーブル                 User エンティティ              │
│  ┌──┬──────┬─────┐             ┌────────────────────┐         │
│  │id│name  │cntry│             │  User              │         │
│  ├──┼──────┼─────┤             │  ├── id: UUID      │         │
│  │1 │Alice │JP   │             │  ├── name: String  │         │
│  │2 │Bob   │US   │             │  ├── country: Str  │         │
│  └──┴──────┴─────┘             │  ├── .orders ──────┼──→ Order│
│                                │  └── .address ─────┼──→ Addr │
│  orders テーブル               └────────────────────┘         │
│  ┌──┬───────┬───────┐                                          │
│  │id│user_id│amount │          ※ エンティティが主役。          │
│  ├──┼───────┼───────┤            テーブルは実装の詳細。        │
│  │10│  1   │ 5000  │            JOINは不要。                   │
│  └──┴───────┴───────┘                                          │
└────────────────────────────────────────────────────────────────┘
```

### Relの基本構造

```
エンティティパターン { 制約 } -> .リレーション -> エンティティパターン { 制約 }
  => 抽出する形状
```

演算子は3つだけ覚えればよい:

```
{ }   エンティティのパターンマッチ（制約）
->    リレーション探索（グラフ走査）
=>    結果の形状を定義（抽出）
```

---

## 2. 基本構文：エンティティパターン

### 2.1 エンティティを直接指定する

SQLに`FROM users`はない。エンティティ名を書くだけ。

```rel
-- すべてのユーザー
User

-- 条件付き（WHEREもない。{}の中に書く）
User { age > 18 }

-- 複数条件（ANDはカンマ、ORは|）
User { age > 18, country: "Japan" }

-- OR条件
User { country: "Japan" | country: "Korea" }

-- ネストした条件
User { age > 18, status: Active, plan: "premium" | plan: "enterprise" }
```

### 2.2 パターンの記法

```rel
-- フィールド一致（== と同じ）
User { country: "Japan" }

-- 比較演算子
User { age > 18 }
User { price >= 1000, price <= 5000 }
User { name != "admin" }

-- Optionフィールドの存在チェック
User { phone: Some }          -- phoneが存在する
User { phone: None }          -- phoneが存在しない

-- リストの中に含まれる
User { country: in ["Japan", "Korea", "Singapore"] }

-- 正規表現マッチ
User { name: ~ /^田/ }

-- 存在チェック（リレーション経由）
User { has(.orders) }         -- 注文がある
User { lacks(.orders) }       -- 注文がない
User { .orders.count > 3 }    -- 注文が3件より多い
```

### 2.3 SQLとの対比

```sql
-- SQL
SELECT * FROM users WHERE age > 18 AND country = 'Japan'
```

```rel
-- Rel: FROM なし、WHERE なし、SELECT なし
User { age > 18, country: "Japan" }
```

---

## 3. リレーション探索

### 3.1 矢印でグラフを辿る

JOINもON句もない。リレーションは事前にスキーマで定義されているので、辿るだけ。

```rel
-- ユーザーとその注文
User { country: "Japan" } -> .orders -> Order { amount > 1000 }

-- 3段階の探索
User -> .orders -> .items -> Product { category: "Electronics" }

-- 4段階
User -> .orders -> .items -> Product -> .category -> Category { name: "Electronics" }
```

### 3.2 探索演算子の種類

```
演算子    意味                            SQLの相当
──────────────────────────────────────────────────────────
->        リレーションを辿る（必須）      INNER JOIN
-?>       リレーションを辿る（任意）      LEFT JOIN
<-        逆方向に辿る                    逆JOIN
<->       双方向                          FULL OUTER JOIN
~>        再帰探索                        WITH RECURSIVE
```

```rel
-- 必須リレーション（対応する相手が必ずいる場合のみ）
User -> .orders -> Order

-- 任意リレーション（クーポンがないユーザーも含む）
User -?> .coupon -> Coupon

-- 自己参照（組織ツリー）
Employee { id: $root_id } ~> .reports(depth: 1..10)
```

### 3.3 リレーション上の制約

矢印の後にも`{}`で制約を追加できる。

```rel
-- 「完了済みの注文」のみを辿る
User -> .orders { status: Delivered, created_at > 2026-01-01 } -> Order

-- 「高評価レビュー」のみ
Product -> .reviews { rating >= 4 } -> Review
```

### 3.4 ネームバインディング（同じ型が複数現れる場合）

```rel
-- フォロワーとフォロイー（どちらもUser型）
u: User { id: $user_id } -> .following -> v: User
=> { from: u.name, to: v.name }

-- 「作成者」と「承認者」が同じUser型
o: Order -> .creator -> creator: User
         -?> .approver -> approver: User?
=> { o.id, creator.name, approver: approver?.name ?? "未承認" }
```

### 3.5 多対多リレーション

```rel
-- 中間テーブルを意識せず書ける
Product -> .tags -> Tag { name: "sale" }

-- 特定タグを持つ商品と、そのタグ一覧
User -> .tags -> Tag
=> per(user) { user.name, tags: collect(tag.name) }
```

---

## 4. 形状の抽出

### 4.1 `=>` 演算子

SQLの`SELECT`に相当するが、パイプラインの**最後に一度だけ書く**。
処理フローの一部ではなく、「欲しい形状の宣言」である。

```rel
-- フィールドを列挙
User { age > 18 }
=> { .id, .name, .email }

-- 名前を変えて抽出
User { age > 18 }
=> { id: .id, full_name: .name, contact: .email }

-- 計算フィールド
User
=> { .name, label: .name ++ " (" ++ .country ++ ")" }

-- リレーション先を含む形状（ネスト）
User { country: "Japan" } -> .orders -> Order
=> {
  user_name:  user.name,
  order_id:   order.id,
  amount:     order.amount,
  product_names: order.items.product.name  -- ドットで深く辿れる
}
```

### 4.2 ネストした形状

```rel
-- ユーザーの注文一覧を、ネスト構造で取得
User { country: "Japan" }
=> {
  .id,
  .name,
  orders: .orders => [{
    .id,
    .amount,
    .status,
    items: .items => [{ product: .product.name, .qty }]
  }]
}
```

結果イメージ:
```json
{
  "id": "...",
  "name": "田中",
  "orders": [
    {
      "id": "...",
      "amount": 5000,
      "status": "Delivered",
      "items": [
        { "product": "スマートフォン", "qty": 1 }
      ]
    }
  ]
}
```

### 4.3 条件つき抽出

```rel
User
=> {
  .name,
  tier: .age >= 65 ? "senior" : .age >= 18 ? "adult" : "minor",
  phone_masked: .phone?.take(4) ++ "****"
}
```

### 4.4 全フィールドの抽出

```rel
-- 全フィールド
User { age > 18 } => .

-- 全フィールド + 追加
User { age > 18 } => { ., rank: row_number() }
```

---

## 5. 集計

### 5.1 `per()` — GROUP BY の再発明

SQLの`GROUP BY`は「どのフィールドでグループ化するか」を宣言する。
Relの`per()`は「どのエンティティ単位で集計するか」を宣言する。

```sql
-- SQL
SELECT country, COUNT(*) FROM users GROUP BY country
```

```rel
-- Rel: "ユーザーを国ごとに集計する"
User
=> per(.country) {
  country: .country,
  count:   count()
}
```

### 5.2 エンティティ単位の集計

```rel
-- SQL: SELECT u.id, u.name, COUNT(o.id), SUM(o.amount)
--      FROM users u JOIN orders o ON o.user_id = u.id GROUP BY u.id, u.name

-- Rel: "ユーザーごとに注文を集計する"
User { country: "Japan" } -> .orders -> Order
=> per(user) {
  user.name,
  order_count:   count(order),
  total_revenue: sum(order.amount),
  avg_order:     avg(order.amount),
  last_order:    max(order.created_at)
}
```

### 5.3 複数レベルの集計

```rel
-- ユーザーごと・ステータスごとの集計
User -> .orders -> Order
=> per(user, order.status) {
  user.name,
  status:  order.status,
  count:   count(order),
  revenue: sum(order.amount)
}
```

### 5.4 HAVING 相当（集計後フィルタ）

```rel
-- 売上1万円以上のユーザーのみ
User -> .orders -> Order
=> per(user) {
  user.name,
  total: sum(order.amount)
} where total > 10000
```

### 5.5 集計関数一覧

```rel
count()            -- 件数
count(entity)      -- nullでない件数
count(distinct .f) -- 重複除外件数
sum(.field)
avg(.field)
min(.field)
max(.field)
median(.field)
stddev(.field)
variance(.field)

-- コレクション系
collect(.field)           -- リストに収集
collect(distinct .field)  -- 重複除外リスト
first(.field)
last(.field)
any(.condition)           -- 1件でも条件を満たすか
all(.condition)           -- 全件条件を満たすか
```

### 5.6 ウィンドウ集計

```rel
-- 注文をユーザーごとにランキング（直近順）
User -> .orders -> Order
=> {
  user.name,
  order.amount,
  order.created_at,
  rank: rank() over user by order.created_at desc,
  running_total: sum(order.amount) over user by order.created_at
}
```

---

## 6. ソートとページング

### 6.1 ソート

```rel
-- sort はパターンの後、=> の前または後に書ける
User { age > 18 }
sort (.name asc)
=> { .name, .age }

-- 複数条件
User
sort (.country asc, .age desc)
=> { .name, .country, .age }

-- 集計後のソート
User -> .orders -> Order
=> per(user) { user.name, total: sum(order.amount) }
sort (total desc)
```

### 6.2 ページング

```rel
-- オフセットベース
User { age > 18 }
sort (.created_at desc)
take 20, skip 40

-- カーソルベース（分散環境推奨）
User
sort (.created_at desc, .id desc)
after "eyJpZCI6MTAwfQ=="
take 20
```

### 6.3 1件取得

```rel
-- 1件のみ（0件や複数件はエラー）
User { id: $id } one

-- 最初の1件（なければ None）
User { email: $email } first
```

---

## 7. スキーマ定義

SQLの`CREATE TABLE`に相当するが、「テーブル」ではなく「エンティティ」を定義する。
エンティティには型・制約・デフォルト値をすべて宣言できる。

### 7.1 エンティティ定義

```rel
entity User {
  id:            UUID           = gen_uuid_v7()
  email:         Email          unique indexed
  name:          String(200)    required
  display_name:  String(100)?
  age:           Int?           check(. >= 0 and . <= 150)
  status:        UserStatus     = Active
  country:       String?
  plan:          Plan           = Free
  metadata:      Json           = {}
  created_at:    Timestamp      = now()
  updated_at:    Timestamp      = now()  auto_update
  deleted_at:    Timestamp?     indexed

  index [country, status]
  index [created_at desc]
  check(name != "")
}
```

### 7.2 フィールド修飾子一覧

```
修飾子              意味
──────────────────────────────────────────────────────────
= <expr>           デフォルト値
required           NOT NULL（デフォルトはnullable）
unique             ユニーク制約
indexed            単一インデックス
check(expr)        チェック制約（. は自フィールドの値）
auto_update        更新時に自動でデフォルト式を再評価
immutable          作成後変更不可
encrypted          保存時に暗号化
deprecated         非推奨（警告が出る）
computed(expr)     計算フィールド（ストアしない）
```

### 7.3 カスタム型の定義

```rel
-- 型エイリアス（バリデーション付き）
type Email    = String  check(is_email(.))
type Url      = String  check(is_url(.))
type Price    = Decimal(10,2) check(. >= 0)
type Quantity = Int check(. >= 0)

-- 構造型
type Address = {
  street:  String,
  city:    String,
  country: String,
  zip:     String?
}

-- Enum
enum UserStatus { Active, Inactive, Suspended, Deleted }
enum Plan       { Free, Standard, Premium, Enterprise }
enum OrderStatus {
  Draft, Pending, Processing,
  Shipped { tracking: String, carrier: String },
  Delivered { at: Timestamp },
  Cancelled { reason: String }
}
```

### 7.4 エンティティの継承

```rel
-- 基底エンティティ
entity Timestamped {
  created_at: Timestamp = now()
  updated_at: Timestamp = now() auto_update
  deleted_at: Timestamp?
}

-- 継承
entity User extends Timestamped {
  id:    UUID   = gen_uuid_v7()
  email: Email  unique
  name:  String required
}
```

### 7.5 マイグレーション

```rel
migration "2026-03-06/001-initial" {
  up {
    create User, Order, OrderItem, Product, Category, Tag
  }
  down {
    drop Tag, Category, Product, OrderItem, Order, User
  }
}

migration "2026-03-06/002-add-phone" {
  up   { User += { phone: String? } }
  down { User -= phone }
}
```

---

## 8. リレーション定義

リレーションはスキーマの一部として宣言する。
クエリ時にJOINキーを書かなくてよいのはこのためである。

### 8.1 基本的なリレーション

```rel
-- 1対多: User は複数の Order を持つ
User -> orders -> Order[*] via order.user_id

-- 多対1: Order は1人の User に属する
Order -> user -> User via order.user_id

-- 1対1: User は1つの Profile を持つ（任意）
User -> profile -> Profile? via profile.user_id
```

### 8.2 多対多

```rel
-- 中間テーブルを自動管理
User -> tags <-> Tag[*] via UserTag {
  user_id: -> User.id
  tag_id:  -> Tag.id
}
```

### 8.3 自己参照

```rel
entity Employee {
  id:         UUID
  name:       String
  manager_id: UUID?
}

Employee -> manager  -> Employee?   via employee.manager_id
Employee -> reports  -> Employee[*] via employee.manager_id
Employee -> ancestors -> Employee[*] recursive via employee.manager_id
```

### 8.4 条件付きリレーション

```rel
-- アクティブな注文のみ
User -> active_orders -> Order[*]
  via order.user_id
  where order.status in [Pending, Processing, Shipped]
  sort order.created_at desc
```

### 8.5 ポリモーフィックリレーション

```rel
entity Comment {
  id:          UUID
  body:        String
  target_type: String
  target_id:   UUID
}

Comment -> target -> Post | Product | User via (target_type, target_id)
```

---

## 9. 型システム

### 9.1 プリミティブ型

```
数値:     Int, Int8, Int16, Int32, Int64, UInt, Float, Float32, Decimal(p,s)
文字列:   String, String(n), Text
真偽値:   Bool
時間:     Date, Time, Timestamp, TimestampTZ, Duration
バイナリ: Bytes
識別子:   UUID, ULID, Snowflake
ネット:   IpAddr, Url, Email
地理:     Point, Polygon, GeoHash
```

### 9.2 コンテナ型

```rel
String?          -- Option<String>: Some(value) | None
List<T>          -- 順序あり、重複可
Set<T>           -- 順序なし、重複不可
Map<K, V>        -- キーバリュー
(T1, T2, T3)     -- タプル
Stream<T>        -- 無限シーケンス（ストリーム処理用）
```

### 9.3 Option型（NULLの廃止）

```
SQLのNULL    三値論理。NULL == NULL → NULL（!）
RelのNone    Option型。None == None → false（直感的）
```

```rel
-- Option の操作
user.phone ?? "N/A"              -- None ならデフォルト値
user.phone?.to_upper()           -- None なら None のまま
user.phone!                      -- 強制アンラップ（Noneならパニック）

-- パターンマッチ
user.phone match {
  Some(p) => send_sms(p),
  None    => send_email(user.email)
}
```

### 9.4 代数的データ型

```rel
-- Enumのバリアントがデータを持てる
enum OrderStatus {
  Pending
  Processing { started_at: Timestamp }
  Shipped    { tracking: String, carrier: String }
  Delivered  { at: Timestamp }
  Cancelled  { reason: String, refund: Price? }
}

-- パターンマッチ
order.status match {
  Pending             => "受付中",
  Processing { started_at } => "処理中: " ++ started_at,
  Shipped { tracking, carrier } => carrier ++ "/" ++ tracking,
  Delivered { at }    => "配達済: " ++ at,
  Cancelled { reason } => "ＣＡ: " ++ reason
}
```

### 9.5 型推論

```rel
-- アノテーション不要
let count   = User { age > 18 } count     -- Int
let names   = User => .name               -- Stream<String>
let revenue = Order sum(.amount)          -- Decimal

-- 型エラーはコンパイル時に検出
User { age > "adult" }
-- Error[E001]: Type mismatch: age is Int, "adult" is String
```

---

## 10. 名前付きパターン（再利用）

### 10.1 パターンの定義

`def`でパターンに名前をつける。SQLのVIEWに相当するが、**合成可能**。

```rel
-- 基本パターン
def active_users = User { status: Active, deleted_at: None }

-- 既存パターンを拡張して合成
def adult_users   = active_users { age >= 18 }
def premium_users = active_users { plan: Premium | plan: Enterprise }
def japan_users   = active_users { country: "Japan" }

-- 合成
def japan_premium = japan_users { plan: Premium | plan: Enterprise }
```

### 10.2 パラメータ付きパターン

```rel
-- 引数を取るパターン
def users_in(country: String) = active_users { country }

def orders_since(days: Int) = Order {
  created_at > now() - days.d,
  status: Delivered
}

def top_products(n: Int, category: String?) =
  Product { category: category ?? any, stock > 0, is_active: true }
  sort (.sold_count desc)
  take n

-- 使用
users_in("Japan") -> .orders -> orders_since(30)
=> per(user) { user.name, revenue: sum(order.amount) }
```

### 10.3 パターンの合成（高階）

```rel
-- クエリを引数に取る
def with_pagination(pattern: Pattern, page: Int, per: Int) =
  pattern skip (page * per) take per

def with_soft_delete(pattern: Pattern) =
  pattern { deleted_at: None }

-- 使用
with_pagination(adult_users, page: 2, per: 50)
=> { .id, .name }
```

### 10.4 マテリアライズドパターン

```rel
-- 定期的に事前計算してキャッシュ
@materialized(refresh: 1h)
def daily_revenue =
  Order { status: Delivered }
  => per(.created_at.date) {
    date:    .created_at.date,
    revenue: sum(.amount),
    orders:  count()
  }
```

---

## 11. モジュール

### 11.1 モジュール定義

```rel
module analytics {
  import ecommerce.*
  import std.time.*

  export def active_users = User { status: Active }

  export def revenue_by_country =
    Order { status: Delivered }
    -> .user -> User
    => per(user.country) {
      country: user.country,
      revenue: sum(order.amount),
      orders:  count(order)
    }
    sort (revenue desc)

  export def top_customers(limit: Int = 10) =
    User -> .orders { status: Delivered } -> Order
    => per(user) {
      user.name,
      total:  sum(order.amount),
      count:  count(order)
    }
    sort (total desc)
    take limit

  -- プライベート（exportなし）
  def _base = Order { status: Delivered }
}
```

### 11.2 インポートとネームスペース

```rel
import analytics.revenue_by_country
import analytics { top_customers, active_users }
import analytics as an

-- 使用
an.top_customers(limit: 20) => { .name, .total }
```

---

## 12. 書き込み操作

### 12.1 作成（INSERT）

```rel
-- エンティティを作成
create User {
  email: "alice@example.com",
  name:  "Alice",
  age:   25
}
-- → User型を返す

-- 複数作成
create User[*] [
  { email: "alice@example.com", name: "Alice" },
  { email: "bob@example.com",   name: "Bob"   }
]
```

### 12.2 更新（UPDATE）

```rel
-- エンティティパターンにマッチするものを更新
User { id: $id } update {
  name:       "New Name",
  updated_at: now()
}

-- リレーション先も含めた更新
User { country: "Japan" } update { tax_rate: 0.1 }
```

### 12.3 削除（DELETE）

```rel
-- 物理削除
User { status: Deleted, deleted_at < now() - 90d } delete

-- ソフトデリート（updateで表現）
User { id: $id } update { deleted_at: now(), status: Deleted }
```

### 12.4 UPSERT

```rel
User { email: "alice@example.com" } upsert {
  name:       "Alice Updated",
  updated_at: now()
} on_create {
  -- 新規作成時のみ設定するフィールド
  created_at: now()
}
```

### 12.5 トランザクション

```rel
transaction place_order {
  isolation: Serializable
  timeout:   30s
  retry:     3

  let user = User { id: $user_id, status: Active } one

  let product = Product { id: $product_id } lock(write) one
  guard product.stock >= $qty else InsufficientStock { available: product.stock }

  product update { stock: product.stock - $qty }

  let order = create Order {
    user_id:      $user_id,
    total_amount: product.price * $qty,
    status:       Pending
  }

  create OrderItem {
    order_id:   order.id,
    product_id: $product_id,
    qty:        $qty,
    unit_price: product.price
  }

  return { order.id, total: order.total_amount }
}
```

---

## 13. 実行モデル

### 13.1 コンパイルパイプライン

```
┌─────────────────────────────────────────────────────────────────┐
│                   Rel コンパイルパイプライン                    │
│                                                                 │
│  .rel ソース                                                    │
│       │                                                         │
│       ▼                                                         │
│  [ Lexer ]  → Token列                                           │
│       │                                                         │
│       ▼                                                         │
│  [ Parser ] → AST                                               │
│               ├── EntityPattern                                 │
│               ├── TraversalExpr                                 │
│               ├── ExtractionExpr                                │
│               └── AggregationExpr                               │
│       │                                                         │
│       ▼                                                         │
│  [ Type Checker ]                                               │
│    - エンティティの型解決                                       │
│    - リレーションの検証                                         │
│    - 抽出形状の型付け                                           │
│    - 集計文脈のチェック                                         │
│       │                                                         │
│       ▼                                                         │
│  [ Logical Plan ]  グラフ探索計画                               │
│    EntityScan → TraversalJoin → Aggregate → Project             │
│       │                                                         │
│       ▼                                                         │
│  [ Optimizer ]                                                  │
│    - パターン条件のプッシュダウン                               │
│    - インデックス選択                                           │
│    - JOINの順序最適化                                           │
│    - マテリアライズドパターンの置換                             │
│       │                                                         │
│       ▼                                                         │
│  [ Physical Plan ]  実際の実行計画                              │
│    IndexLookup / SeqScan / HashJoin / MergeJoin                 │
│       │                                                         │
│       ▼                                                         │
│  [ Distributed Planner ]  シャード分割計画                      │
│       │                                                         │
│       ▼                                                         │
│  [ Executor ]  並列実行 → 結果ストリーム                        │
└─────────────────────────────────────────────────────────────────┘
```

### 13.2 ASTの構造例

```
クエリ: User { country: "Japan" } -> .orders -> Order { amount > 1000 }
        => per(user) { user.name, total: sum(order.amount) }

AST:
Query
├── Pattern: EntityPattern
│   ├── entity: "User"
│   ├── constraints: [Field("country") == Lit("Japan")]
│   └── traversal:
│       └── TraversalExpr
│           ├── relation: ".orders"
│           ├── target: EntityPattern
│           │   ├── entity: "Order"
│           │   └── constraints: [Field("amount") > Lit(1000)]
│           └── extraction:
│               └── AggregateExtract
│                   ├── group_by: [Binding("user")]
│                   └── shape:
│                       ├── Field("user.name")
│                       └── Alias("total", Agg(Sum, Field("order.amount")))
```

### 13.3 論理プランへの変換

```
論理プラン（S式表記）:

(Aggregate
  group_by: [user.id]
  aggs: [(sum order.amount) as total]
  (HashJoin inner
    key: (user.id = order.user_id)
    (Filter (= user.country "Japan")
      (EntityScan "User"))
    (Filter (> order.amount 1000)
      (EntityScan "Order"))))

最適化後:
(Aggregate
  group_by: [user.id]
  aggs: [(sum order.amount) as total]
  (HashJoin inner
    key: (user.id = order.user_id)
    (IndexScan "User" index:idx_country key:"Japan")     ← インデックス選択
    (IndexScan "Order" index:idx_amount range:(1000,∞)))) ← インデックス選択
```

---

## 14. 分散・ストリーム

### 14.1 透過的分散

```rel
-- 書き方は完全に同じ。エンジンが自動分散。
User -> .orders -> Order
=> per(user.country) { country: user.country, revenue: sum(order.amount) }

-- Scatter-Gather の流れ:
-- Coordinator → 各シャードに部分クエリ送信
-- 各シャード  → 部分集計して返す
-- Coordinator → マージして最終結果
```

### 14.2 シャーディング宣言

```rel
entity Order {
  ...
  @shard(key: .user_id, strategy: hash, count: 256, replicas: 3)
}
```

### 14.3 ストリーム処理

```rel
-- ストリーム定義
stream OrderEvent from kafka("orders") {
  schema: { order_id: UUID, user_id: UUID, amount: Decimal, ts: Timestamp }
}

-- ストリームクエリ（パターン言語そのまま使える）
OrderEvent { amount > 10000 }
  window tumbling(1m)
  => per(.country) {
    country: .country,
    count:   count(),
    revenue: sum(.amount)
  }
  -> sink kafka("revenue-metrics")

-- ストリームとバッチの結合
OrderEvent
  enrich User on .user_id == user.id
  => {
    .order_id,
    user.name,
    user.country,
    .amount
  }
```

---

## 15. SQLとの対比

### 15.1 発想の違い

```
┌─────────────────────────────────────────────────────────────────┐
│         SQL                       Rel                          │
├─────────────────────────────────────────────────────────────────┤
│  "usersテーブルを走査する"       "Userエンティティを探す"        │
│  FROM users                      User { ... }                  │
├─────────────────────────────────────────────────────────────────┤
│  "WHERE句でフィルタする"         "パターンにマッチさせる"        │
│  WHERE age > 18                  User { age > 18 }             │
├─────────────────────────────────────────────────────────────────┤
│  "ordersテーブルをJOINする"      "ordersリレーションを辿る"      │
│  JOIN orders ON ...              -> .orders -> Order            │
├─────────────────────────────────────────────────────────────────┤
│  "カラムをSELECTする"            "欲しい形状を宣言する"          │
│  SELECT name, amount             => { user.name, order.amount } │
├─────────────────────────────────────────────────────────────────┤
│  "GROUP BYで集約する"            "何単位で集計するか宣言する"    │
│  GROUP BY user_id                per(user) { ... }             │
└─────────────────────────────────────────────────────────────────┘
```

### 15.2 具体的なクエリ対比

#### シンプルなクエリ
```sql
-- SQL
SELECT id, name FROM users WHERE age > 18 ORDER BY name LIMIT 10;
```
```rel
-- Rel
User { age > 18 }
sort (.name)
take 10
=> { .id, .name }
```

#### JOIN
```sql
-- SQL
SELECT o.id, u.name, p.name
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE u.country = 'Japan';
```
```rel
-- Rel: JOINキーを一切書かない
User { country: "Japan" } -> .orders -> .items -> Product
=> { order.id, user.name, product.name }
```

#### GROUP BY / HAVING
```sql
-- SQL
SELECT country, COUNT(*), SUM(amount)
FROM orders
JOIN users ON orders.user_id = users.id
WHERE status = 'delivered'
GROUP BY country
HAVING COUNT(*) > 100
ORDER BY SUM(amount) DESC;
```
```rel
-- Rel
Order { status: Delivered } -> .user -> User
=> per(user.country) {
  country: user.country,
  count:   count(order),
  revenue: sum(order.amount)
} where count > 100
sort (revenue desc)
```

#### 再帰クエリ
```sql
-- SQL（複雑）
WITH RECURSIVE sub AS (
  SELECT id, name, manager_id, 0 AS depth FROM employees WHERE id = $id
  UNION ALL
  SELECT e.id, e.name, e.manager_id, s.depth+1
  FROM employees e JOIN sub s ON e.manager_id = s.id
  WHERE s.depth < 10
)
SELECT * FROM sub;
```
```rel
-- Rel（直感的）
Employee { id: $id } ~> .reports(depth: 0..10)
=> { .id, .name, .depth }
```

#### ウィンドウ関数
```sql
-- SQL
SELECT user_id, amount, created_at,
  RANK() OVER (PARTITION BY user_id ORDER BY created_at DESC) as rank
FROM orders;
```
```rel
-- Rel
Order
=> {
  .user_id, .amount, .created_at,
  rank: rank() over user by .created_at desc
}
```

### 15.3 機能比較表

| 機能 | SQL | Rel |
|------|-----|-----|
| データモデル | テーブル（2次元） | エンティティグラフ |
| フィルタ記述 | WHERE句（別記） | パターン内 `{}` |
| テーブル結合 | JOIN + ON句（毎回） | `->` 探索（定義済み） |
| NULL処理 | 三値論理 | Option型（二値） |
| 集計 | GROUP BY | `per()` |
| 集計後フィルタ | HAVING | `where` after `per()` |
| 再帰クエリ | WITH RECURSIVE | `~>` |
| クエリ再利用 | VIEW / プロシージャ | `def`（合成可能） |
| モジュール | なし | `module` |
| 型安全 | 弱い | 強い静的型 |
| コンパイル時検証 | ほぼなし | 型・リレーション全チェック |
| 分散処理 | 方言依存 | ネイティブ組み込み |
| ストリーム | なし | ネイティブ |
| スキーマと言語 | 分離 | 統合（型定義=エンティティ定義） |

---

## 付録A: 完全なクエリ例

### ECサイト — ダッシュボード

```rel
-- 直近30日のKPI
@report monthly_kpi(days: Int = 30) {
  let orders = Order { created_at > now() - days.d, status: Delivered }
  let users  = User { has(.orders { created_at > now() - days.d }) }

  return {
    revenue:        orders sum(.amount),
    order_count:    orders count(),
    new_users:      User { created_at > now() - days.d } count(),
    active_users:   users count(),
    avg_order:      orders avg(.amount),
    top_country:    (orders -> .user -> User
                     => per(user.country) { country: user.country, rev: sum(order.amount) }
                     sort (rev desc) take 1) first?.country
  }
}
```

### SNS — タイムライン

```rel
def timeline(user_id: UUID, cursor: String?, limit: Int = 50) =
  User { id: user_id }
  -> .following -> User
  -> .posts { is_public: true }
  -> Post
  -?> .media -> Media
  sort (post.created_at desc, post.id desc)
  after cursor
  take limit

timeline($user_id, cursor: $cursor)
=> {
  edges: [{
    id:         post.id,
    body:       post.body,
    created_at: post.created_at,
    author: {   id: user.id, name: user.name, avatar: user.avatar_url },
    media:  media?.map(m => { url: m.url, type: m.type })
  }],
  next_cursor: last(encode_cursor(post.created_at, post.id))
}
```

### 不正検知 — リアルタイム

```rel
-- バッチ側: ユーザーの通常パターンを事前計算
@materialized(refresh: 6h)
def user_baseline =
  Order { created_at > now() - 90d } -> .user -> User
  => per(user) {
    user_id:       user.id,
    avg_amount:    avg(order.amount),
    stddev_amount: stddev(order.amount),
    usual_country: mode(user.country)
  }

-- ストリーム側: リアルタイム検知
OrderEvent
  enrich user_baseline on .user_id == baseline.user_id
  let z = (.amount - baseline.avg_amount) / (baseline.stddev_amount + 0.01)
  let mismatch = .country != baseline.usual_country
  where z > 3.0 or mismatch
  => {
    .order_id, .user_id, .amount,
    z_score:          z,
    country_mismatch: mismatch,
    risk_level:       z > 5.0 ? Critical : High
  }
  -> sink fraud_alerts
```

---

## 付録B: エラーメッセージ例

```
Error[E001]: Type mismatch in entity pattern
  --> query.rel:3:12
  |
3 |  User { age > "adult" }
  |               ^^^^^^^
  |               expected Int (field 'age' is Int), found String
  |
  hint: Did you mean `age > 18`?

Error[E042]: Unknown relation
  --> query.rel:5:9
  |
5 |  User -> .payment -> Payment
  |           ^^^^^^^
  |           relation 'payment' not found on entity 'User'
  |
  hint: Available relations on User: orders, profile, tags, following
        Did you mean `Order -> .payment`?

Error[E050]: Non-aggregate field in per() context
  --> query.rel:9:5
  |
9 |    order.id,          <- ここが問題
  |    ^^^^^^^^
  |    'order.id' is not in per(user) group key and is not an aggregate
  |
  hint: Use `first(order.id)`, `collect(order.id)`, or add `order` to per()

Error[E070]: Option type used without unwrapping
  --> query.rel:12:18
  |
12|  => { phone: user.phone.to_upper() }
  |                    ^^^^^
  |                    'phone' is Option<String>. Cannot call .to_upper() directly.
  |
  hint: Use `user.phone?.to_upper()` or `user.phone!.to_upper()`
```

---

*Rel Language Specification v0.2.0-draft*
*Copyright 2026 — Rel Language Design Group*
